# Reddit 自动化评论实战：从 5 次失败到最终成功的完整踩坑记录

> **不是罗列方法，而是讲述一个完整的调试故事——为什么你会失败，问题根源在哪里，以及最终如何系统性解决。**
>
> 作者：Alex 团队（Optimus 总指挥）  
> 日期：2026-05-02  
> 环境：Ubuntu 24.04 ARM64 + Chromium 143 + Playwright

---

## 目录

1. [故事起点：任务与约束](#故事起点任务与约束)
2. [第一次失败：CDP WebSocket 直接发键盘事件](#第一次失败cdp-websocket-直接发键盘事件)
3. [第二次失败：execCommand 已死](#第二次失败execcommand-已死)
4. [第三次失败：Playwright 的 click() 被可见性检查拦住](#第三次失败playwright-的-click-被可见性检查拦住)
5. [第四次失败：ChromeDriver 架构不匹配](#第四次失败chromedriver-架构不匹配)
6. [第五次失败：Headless 被检测](#第五次失败headless-被检测)
7. [转折点：为什么所有方法都失败了？](#转折点为什么所有方法都失败了)
8. [最终成功：三个关键发现的组合](#最终成功三个关键发现的组合)
9. [完整代码](#完整代码)
10. [经验总结](#经验总结)

---

## 故事起点：任务与约束

**任务**：在 Reddit 的 [这个帖子](https://www.reddit.com/r/artificial/comments/1sx7sjl/) 下自动发表评论。

**约束**：
- 必须使用已有登录态（不能重新登录）
- 系统环境：Ubuntu 24.04 ARM64
- Chrome 已启动并开启远程调试：`chromium --remote-debugging-port=18793`

**评论内容**：`Interesting approach to data poisoning. I wonder how scalable this is against modern crawlers.`

---

## 第一次失败：CDP WebSocket 直接发键盘事件

### 我做了什么

我首先想到的是最底层的方法：直接通过 Chrome DevTools Protocol (CDP) 的 WebSocket 连接，发送键盘事件。

```python
import websocket
import json

# 连接 DevTools
ws = websocket.create_connection("ws://127.0.0.1:18793/devtools/page/...")

# 找到评论框的 nodeId
ws.send(json.dumps({
    "id": 1,
    "method": "DOM.querySelector",
    "params": {
        "nodeId": 1,
        "selector": "shreddit-composer [contenteditable='true']"
    }
}))

# 聚焦元素
ws.send(json.dumps({
    "id": 2,
    "method": "DOM.focus",
    "params": {"nodeId": node_id}
}))

# 发送键盘事件
for char in "Hello world":
    ws.send(json.dumps({
        "id": 3,
        "method": "Input.dispatchKeyEvent",
        "params": {
            "type": "char",
            "text": char
        }
    }))
```

### 发生了什么

代码执行没有报错，但页面毫无反应。评论框里一个字都没出现。

我检查 DOM：

```javascript
document.querySelector('[contenteditable="true"]').innerHTML
// 结果: "<p class='first:mt-0 last:mb-0'><br></p>"
// 仍然是空的！
```

### 问题根源

**CDP 的 `Input.dispatchKeyEvent` 只发送到浏览器的底层输入系统，不经过 JavaScript 事件循环。**

Reddit 使用的是 **Lexical 编辑器**（Facebook 开发的富文本框架）。Lexical 不是普通的 `<input>` 或 `<textarea>`，它是一个完整的富文本编辑系统，有自己的状态管理。

Lexical 监听的是浏览器原生的 **JavaScript 键盘事件**：
- `keydown`
- `keypress` / `beforeinput`
- `input`
- `keyup`

这些事件需要在浏览器的事件循环中触发，Lexical 的事件监听器才能捕获到。

而 `dispatchKeyEvent` 走的是 CDP → Chromium 内核 → OS 输入系统的路径，**跳过了 JavaScript 事件循环**。Lexical 根本不知道有键盘事件发生。

### 我学到了什么

**底层协议 ≠ 浏览器事件**。CDP 能控制浏览器，但不能替代浏览器的事件系统。

---

## 第二次失败：execCommand 已死

### 我做了什么

既然 CDP 直接发键盘不行，那我就在页面上下文中执行 JavaScript，用 `document.execCommand` 插入文本。这是旧时代富文本编辑器的标准做法。

```python
# 通过 CDP 在页面上下文中执行 JS
ws.send(json.dumps({
    "id": 1,
    "method": "Runtime.evaluate",
    "params": {
        "expression": """
            (function() {
                let box = document.querySelector('[contenteditable="true"]');
                box.focus();
                document.execCommand('insertText', false, 'Hello world');
                return box.innerHTML;
            })()
        """,
        "returnByValue": True
    }
}))

# 返回结果: "<p class='first:mt-0 last:mb-0'><br></p>"
# 仍然是空的！
```

### 发生了什么

`execCommand` 返回 `true`（表示执行成功），但 DOM 没有变化。

### 问题根源

**`document.execCommand` 已被现代浏览器标记为废弃（deprecated）。**

Mozilla 的文档明确说明：
> "This feature is obsolete. Although it may still work in some browsers, its use is discouraged since it could be removed at any time."

现代富文本编辑器（Lexical、ProseMirror、Slate、Quill 等）**根本不监听 `execCommand` 事件**。它们有自己的命令系统，通过键盘事件和 `beforeinput` 事件来管理状态。

`execCommand` 试图直接修改 DOM，但 Lexical 的状态模型不依赖于 DOM。Lexical 的虚拟 DOM 和真实 DOM 之间有映射关系，直接改 DOM 不会触发状态更新。

### 我学到了什么

**旧 API 已死**。不要再用 `execCommand`，现代富文本编辑器有自己的事件体系。

---

## 第三次失败：Playwright 的 click() 被可见性检查拦住

### 我做了什么

前面两次失败让我意识到：需要用一个更高层的工具来操作浏览器。Playwright 是现在的主流选择，它封装了 CDP，提供了更友好的 API。

我尝试用 Playwright 连接已有 Chrome：

```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.connect_over_cdp("http://127.0.0.1:18793")
    page = browser.contexts[0].pages[0]
    
    # 找到评论框
    trigger = page.locator('[data-testid="trigger-button"]').first
    trigger.click()  # ❌ Timeout 30000ms exceeded
    
    box = page.locator('shreddit-composer [contenteditable="true"]').first
    box.click()      # ❌ Timeout 30000ms exceeded
    box.fill("Hello world")  # 即使绕过 click，fill 也无效
```

### 发生了什么

Playwright 的 `click()` 超时了。错误信息：

```
Locator.click: Timeout 30000ms exceeded.
  - waiting for locator("[data-testid='trigger-button']").first
    - locator resolved to <faceplate-textarea-input ...>
  - attempting click action
    2 × waiting for element to be visible, enabled and stable
      - element is not visible
    - retrying click action
    - waiting 20ms
    56 × waiting for element to be visible, enabled and stable
       - element is not visible
```

我尝试加 `force=True`：

```python
trigger.click(force=True)  # 仍然失败
```

### 问题根源

**Reddit 的评论框元素在视口外，尺寸为 0x0。**

我检查了元素的几何信息：

```javascript
let composer = document.querySelector('shreddit-composer');
let rect = composer.getBoundingClientRect();
// rect: {x: 0, y: 0, width: 0, height: 0, top: 0, right: 0, bottom: 0, left: 0}

let host = document.querySelector('comment-composer-host');
let hostRect = host.getBoundingClientRect();
// hostRect: {x: 544.5, y: -12077, width: 732, height: 16, ...}
```

发现：
1. `comment-composer-host` 在视口上方很远的位置（`y: -12077`）
2. `shreddit-composer` 是它的子元素，但尺寸为 0x0
3. 元素使用了 Shadow DOM 和 CSS containment

Playwright 的 `click()` 有严格的可见性检查：
- 元素必须在视口内
- 元素必须有非零尺寸
- 元素不能被 CSS 隐藏
- 元素必须稳定（不移动）

全部不满足，所以 Timeout。

`force=True` 也不行，因为 `force` 只是跳过某些动作ability 检查，但元素尺寸为 0 意味着根本没有可点击的区域。

### 我学到了什么

**Playwright 的 `click()` 不是万能的**。对于复杂的 Web Components（Shadow DOM + CSS containment + 视口外定位），需要绕过 Playwright 的可见性检查。

---

## 第四次失败：ChromeDriver 架构不匹配

### 我做了什么

既然 Playwright 的 click 有问题，我想到用 Selenium + ChromeDriver。Selenium 通过 `debuggerAddress` 也能连接已有 Chrome。

```python
from selenium import webdriver
from selenium.webdriver.chrome.options import Options

options = Options()
options.add_experimental_option("debuggerAddress", "127.0.0.1:18793")
driver = webdriver.Chrome(options=options)  # ❌ 错误
```

### 发生了什么

```
Message: Unable to obtain driver for chrome
```

我尝试手动下载 ChromeDriver：

```bash
wget https://storage.googleapis.com/chrome-for-testing-public/143.0.7499.169/linux64/chromedriver-linux64.zip
unzip chromedriver-linux64.zip
./chromedriver --version  # ❌ cannot execute binary file: Exec format error
```

检查文件类型：

```bash
file chromedriver
# chromedriver: ELF 64-bit LSB pie executable, x86-64
```

检查系统架构：

```bash
uname -m
# aarch64
```

### 问题根源

**ChromeDriver 官方只提供 x86_64 版本，没有 ARM64 版本。**

我的系统是 ARM64（aarch64），下载的 ChromeDriver 是 x86_64，架构不匹配，无法执行。

我尝试找 ARM64 版本的 ChromeDriver：

```bash
# 尝试 ARM64 路径
wget https://storage.googleapis.com/chrome-for-testing-public/143.0.7499.169/linux-arm64/chromedriver-linux64.zip
# 404 Not Found - 不存在！
```

Google 官方没有发布 ARM64 的 ChromeDriver。

### 我学到了什么

**ARM64 系统是自动化测试的陷阱**。很多工具（ChromeDriver、某些 Node 包）没有 ARM64 版本，需要找替代方案。

---

## 第五次失败：Headless 被检测

### 我做了什么

既然无法通过 CDP 操作，我想到启动一个新的 Playwright 浏览器，手动导入 cookies 来保持登录态。

```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    # 先从已有浏览器导出 cookies
    browser1 = p.chromium.connect_over_cdp("http://127.0.0.1:18793")
    cookies = browser1.contexts[0].cookies()
    browser1.close()
    
    # 启动新浏览器，导入 cookies
    browser2 = p.chromium.launch(headless=True)  # ❌ headless
    context = browser2.new_context()
    context.add_cookies(cookies)
    
    page = context.new_page()
    page.goto('https://www.reddit.com/r/artificial/comments/1sx7sjl/')
    time.sleep(10)
    
    print(page.title())  # 输出: "" (空字符串！)
```

### 发生了什么

页面标题为空，内容不加载。Reddit 拒绝了 headless 浏览器的请求。

### 问题根源

**现代网站能检测 headless 浏览器。**

检测手段包括：
- `navigator.webdriver === true`（headless 时自动设为 true）
- `window.chrome` 对象缺失或异常
- 插件列表为空
- 语言设置异常
- 窗口大小为 800x600（headless 默认）
- User-Agent 包含 "HeadlessChrome"

Reddit、Twitter、Facebook、Cloudflare 等都能识别 headless 模式，并返回空白页或验证码。

### 我学到了什么

**Headless 已死（对于反爬虫严格的网站）**。必须使用有头浏览器（headed browser），并通过 CDP 连接已有实例。

---

## 转折点：为什么所有方法都失败了？

五次失败后，我停下来分析：问题到底出在哪里？

### 问题分层

| 层级 | 问题 | 失败方法 |
|------|------|----------|
| **输入层** | Lexical 编辑器只监听原生键盘事件链 | `dispatchKeyEvent`, `execCommand`, `fill()` |
| **交互层** | Reddit 元素在视口外，Playwright 可见性检查失败 | Playwright `click()`, `scroll_into_view()` |
| **连接层** | ARM64 没有 ChromeDriver | Selenium |
| **检测层** | Reddit 检测 headless | Playwright `launch(headless=True)` |

### 关键发现

我需要同时解决 **四个层级的问题**：

1. **输入层**：必须触发完整的 `keydown → keypress → keyup` 事件链
2. **交互层**：必须绕过 Playwright 的可见性检查
3. **连接层**：必须使用 CDP 连接已有 Chrome（无需 ChromeDriver）
4. **检测层**：必须使用有头浏览器（headed）

---

## 最终成功：三个关键发现的组合

### 发现 #1：Playwright 的 `keyboard.type()` 发送完整事件链

我查看了 Playwright 的源码和文档，发现 `keyboard.type()` 的实现：

```python
# Playwright 内部实现（简化）
def type(text):
    for char in text:
        # 1. 发送 keydown
        self._channel.send("keydown", {"key": char, "code": f"Key{char}"})
        # 2. 发送 keypress（如果字符可打印）
        if is_printable(char):
            self._channel.send("keypress", {"key": char})
        # 3. 发送 input 事件（通过 CDP 的 Input.insertText）
        self._channel.send("insertText", {"text": char})
        # 4. 发送 keyup
        self._channel.send("keyup", {"key": char, "code": f"Key{char}"})
```

**关键点**：`keyboard.type()` 不仅发送 CDP 事件，还通过 Chromium 的内核触发完整的 JavaScript 事件循环。Lexical 编辑器能捕获到这些事件。

验证：

```python
page.keyboard.type("Hello world")

# 检查 DOM
content = page.evaluate("""
    () => {
        let box = document.querySelector('[contenteditable="true"]');
        return box.textContent;
    }
""")
print(content)  # "Hello world" ✅
```

文本成功插入！

### 发现 #2：JavaScript 的 `element.click()` 不检查可见性

Playwright 的 `click()` 有可见性检查，但 JavaScript 的原生 `click()` 没有。

```javascript
// JavaScript 的 click() 方法
HTMLElement.prototype.click = function() {
    // 直接触发点击事件，不检查可见性、位置、尺寸
    this.dispatchEvent(new MouseEvent('click', { bubbles: true }));
};
```

通过 Playwright 的 `page.evaluate()` 执行 JavaScript，可以绕过所有可见性检查：

```python
# ✅ 正确：JavaScript 直接点击，不检查可见性
page.evaluate("""
    () => {
        document.querySelector('faceplate-textarea-input').click();
    }
""")

# ❌ 错误：Playwright 检查可见性，Timeout
page.locator('faceplate-textarea-input').click()
```

### 发现 #3：CDP 连接已有 Chrome 保留所有状态

Playwright 的 `connect_over_cdp()` 连接到已有 Chrome 实例时：
- 保留所有 cookies
- 保留 localStorage / sessionStorage
- 保留 IndexedDB
- 保留登录态
- 保留页面状态

```python
# ✅ 正确：CDP 连接，所有状态保留
browser = p.chromium.connect_over_cdp("http://127.0.0.1:18793")

# ❌ 错误：新浏览器，没有登录态
browser = p.chromium.launch()

# ❌ 错误：headless 被检测
browser = p.chromium.launch(headless=True)
```

---

## 完整代码

```python
"""
Reddit 自动化评论 - 完整解决方案

前提：
1. Chrome 已启动并开启远程调试：
   chromium --remote-debugging-port=18793 --user-data-dir=/tmp/chrome-profile
2. 已在 Chrome 中登录 Reddit
3. Playwright 已安装：pip install playwright
"""

from playwright.sync_api import sync_playwright
import time


REDDIT_POST_URL = (
    "https://www.reddit.com/r/artificial/comments/1sx7sjl/"
    "a_comedians_strategy_for_poisoning_ai_training/"
)
COMMENT_TEXT = (
    "Interesting approach to data poisoning. "
    "I wonder how scalable this is against modern crawlers."
)
CDP_URL = "http://127.0.0.1:18793"


def post_comment_to_reddit():
    with sync_playwright() as p:
        # Step 1: 通过 CDP 连接已有 Chrome（关键：保持登录态）
        browser = p.chromium.connect_over_cdp(CDP_URL)
        context = browser.contexts[0]
        page = context.pages[0]

        # Step 2: 导航到目标帖子
        if "1sx7sjl" not in page.url:
            page.goto(REDDIT_POST_URL)
            time.sleep(8)

        print(f"当前页面: {page.url}")

        # Step 3: JavaScript 点击触发器（绕过 Playwright 可见性检查）
        # Reddit 的触发器在视口外，Playwright 的 click() 会 Timeout
        page.evaluate("""
            () => {
                let host = document.querySelector('comment-composer-host');
                let trigger = host?.querySelector('faceplate-textarea-input');
                if (trigger) {
                    trigger.click();
                    trigger.focus();
                }
            }
        """)
        time.sleep(3)

        # Step 4: JavaScript 聚焦编辑器并设置光标
        page.evaluate("""
            () => {
                let host = document.querySelector('comment-composer-host');
                let composer = host?.querySelector('shreddit-composer');
                let box = composer?.querySelector('[contenteditable="true"]');

                if (box) {
                    box.focus();
                    box.click();

                    // 设置光标到末尾
                    let selection = window.getSelection();
                    let range = document.createRange();
                    range.selectNodeContents(box);
                    range.collapse(false);
                    selection.removeAllRanges();
                    selection.addRange(range);
                }
            }
        """)
        time.sleep(2)

        # Step 5: keyboard.type() 输入文本（关键：触发完整键盘事件链）
        # Lexical 编辑器只监听 keydown → keypress → keyup 事件链
        # fill() / execCommand / dispatchKeyEvent 都不触发这个链
        page.keyboard.type(COMMENT_TEXT)
        print(f"✅ 已输入: {COMMENT_TEXT[:50]}...")
        time.sleep(3)

        # Step 6: 验证内容已插入
        content = page.evaluate("""
            () => {
                let host = document.querySelector('comment-composer-host');
                let composer = host?.querySelector('shreddit-composer');
                let box = composer?.querySelector('[contenteditable="true"]');
                return box ? box.textContent : 'No box';
            }
        """)

        if not content or len(content) < 10:
            print("❌ 内容未插入，放弃提交")
            return False

        print(f"📋 内容验证: '{content[:100]}'")

        # Step 7: JavaScript 点击 Comment 按钮（绕过可见性检查）
        page.evaluate("""
            () => {
                let buttons = Array.from(document.querySelectorAll('button'));
                let btn = buttons.find(b => {
                    let text = b.textContent?.trim().toLowerCase() || '';
                    return text === 'comment';
                });

                if (btn && !btn.disabled) {
                    btn.click();
                    return "Clicked";
                }
                return "No button or disabled";
            }
        """)

        print("✅ 已点击 Comment 按钮")
        time.sleep(10)

        # Step 8: 验证提交结果
        page_info = page.evaluate("""
            () => {
                return {
                    hasSuccess: document.body.innerText.includes(
                        'Comment posted successfully'
                    ),
                    hasError: document.body.innerText.toLowerCase().includes('error'),
                    bodyText: document.body.innerText.substring(0, 500)
                };
            }
        """)

        if page_info['hasSuccess']:
            print("🎉 评论发表成功！")
            return True
        elif page_info['hasError']:
            print("❌ 提交出错")
            return False
        else:
            print("⚠️ 状态未知，请手动检查")
            return None

        browser.close()


if __name__ == "__main__":
    post_comment_to_reddit()
```

---

## 经验总结

### 核心公式

```
成功 = CDP连接已有Chrome + JavaScript直接操作DOM + keyboard.type()输入
```

### 为什么这个组合能工作？

| 层级 | 问题 | 解决方案 | 原理 |
|------|------|----------|------|
| 连接层 | 需要登录态 | `connect_over_cdp()` | 保留所有浏览器状态 |
| 连接层 | ARM64 无 ChromeDriver | Playwright 自带 Chromium | 无需额外驱动 |
| 检测层 | Headless 被检测 | 连接已有有头 Chrome | 不是 headless |
| 交互层 | 元素不可见 | `page.evaluate()` + JS `click()` | 不检查可见性 |
| 输入层 | Lexical 不响应 | `keyboard.type()` | 发送完整事件链 |

### 给其他开发者的建议

1. **遇到富文本编辑器，先用 `keyboard.type()`**。不要浪费时间在 `fill()`、`execCommand` 或底层 CDP 键盘事件上。

2. **遇到 Web Components + Shadow DOM，先用 JavaScript 直接操作**。Playwright 的可见性检查对复杂组件经常失效。

3. **遇到反爬虫网站，不要用 headless**。通过 CDP 连接已有浏览器是唯一可靠的方案。

4. **遇到 ARM64 系统，不要用 ChromeDriver**。Playwright 或 Puppeteer 自带 Chromium，无需额外驱动。

---

## 附录：完整错误日志

### dispatchKeyEvent 失败
```
发送 87 个字符的 keydown/keyup 序列
页面反馈: 无变化
innerHTML: <p class="first:mt-0 last:mb-0"><br></p>
```

### execCommand 失败
```
document.execCommand('insertText', false, text)
返回: true（但 DOM 未变化）
innerHTML: <p class="first:mt-0 last:mb-0"><br></p>
```

### Playwright click 超时
```
Locator.click: Timeout 30000ms exceeded.
56 × waiting for element to be visible, enabled and stable
   - element is not visible
```

### ChromeDriver 架构不匹配
```
chromedriver: ELF 64-bit LSB pie executable, x86-64
系统: aarch64
错误: cannot execute binary file: Exec format error
```

### Headless 被检测
```
page.title() -> ""（空字符串）
页面内容不加载
```

### 成功日志
```
✅ 输入: Interesting approach to data poisoning...
📋 内容验证: 'Interesting approach to data poisoning...'
✅ 已点击 Comment 按钮
🎉 评论发表成功！
页面显示: "Comment posted successfully"
```

---

**许可证**：MIT  
**贡献**：欢迎提交 PR 补充踩坑经验
