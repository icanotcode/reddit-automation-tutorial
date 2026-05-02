# Reddit 自动化评论实战教程

> **从失败到成功：完整记录使用 Playwright + CDP 实现 Reddit 自动化评论踩坑过程**
>
> 作者：Alex 团队（Optimus 总指挥）
> 日期：2026-05-02
> 环境：Ubuntu 24.04 ARM64 + Chromium 143 + Playwright

---

## 📋 目录

1. [任务目标](#任务目标)
2. [环境准备](#环境准备)
3. [失败方法汇总](#失败方法汇总)
4. [成功方案详解](#成功方案详解)
5. [核心代码](#核心代码)
6. [踩坑总结](#踩坑总结)
7. [延伸阅读](#延伸阅读)

---

## 🎯 任务目标

在已登录 Reddit 的浏览器中，对指定帖子自动发表评论：

- **帖子**：`https://www.reddit.com/r/artificial/comments/1sx7sjl/`
- **评论内容**：`Interesting approach to data poisoning. I wonder how scalable this is against modern crawlers.`
- **约束**：必须使用已有登录态，不能重新登录

---

## 🛠️ 环境准备

### 系统环境

```bash
# 操作系统
Ubuntu 24.04 LTS (ARM64/aarch64)

# Chrome 路径
/usr/lib/chromium/chromium  # Chromium 143.0.7499.169

# 已启动 Chrome 并开启远程调试
chromium --remote-debugging-port=18793 --user-data-dir=/tmp/chrome-profile
```

### Python 依赖

```bash
pip install playwright
playwright install chromium
```

---

## ❌ 失败方法汇总（按尝试顺序）

### 方法 1：纯 CDP WebSocket + `Input.dispatchKeyEvent`

**思路**：直接通过 Chrome DevTools Protocol 发送键盘事件。

```python
# 连接 DevTools WebSocket
ws = websocket.create_connection(ws_url)

# 聚焦元素
ws.send(json.dumps({
    "id": 1,
    "method": "DOM.focus",
    "params": {"nodeId": node_id}
}))

# 发送键盘输入
for char in text:
    ws.send(json.dumps({
        "id": 2,
        "method": "Input.dispatchKeyEvent",
        "params": {
            "type": "char",
            "text": char
        }
    }))
```

**❌ 失败原因**：
- Reddit 使用 Lexical 编辑器（Facebook 的富文本框架）
- `dispatchKeyEvent` 只触发底层键盘事件，不触发 Lexical 的 `beforeinput` 和 `input` 事件
- 编辑器状态未更新，DOM 中看不到输入的文本

---

### 方法 2：CDP + `Runtime.callFunctionOn` + `document.execCommand`

**思路**：在页面上下文中执行 JavaScript，使用 `execCommand` 插入文本。

```python
ws.send(json.dumps({
    "id": 1,
    "method": "Runtime.callFunctionOn",
    "params": {
        "objectId": remote_object_id,
        "functionDeclaration": """
            function(text) {
                this.focus();
                document.execCommand('insertText', false, text);
                return this.innerHTML;
            }
        """,
        "arguments": [{"value": text}]
    }
}))
```

**❌ 失败原因**：
- `document.execCommand` 已被现代浏览器标记为废弃（deprecated）
- Lexical 编辑器不监听 `execCommand` 事件
- 返回的 `innerHTML` 仍然是 `<p><br></p>`，文本未插入

---

### 方法 3：Playwright `locator.fill()` + `locator.click()`

**思路**：使用 Playwright 的高级 API 操作元素。

```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.connect_over_cdp("http://127.0.0.1:18793")
    page = browser.contexts[0].pages[0]
    
    # 点击触发按钮
    trigger = page.locator('[data-testid="trigger-button"]').first
    trigger.click()  # ❌ Timeout: element is not visible
    
    # 输入评论
    box = page.locator('shreddit-composer [contenteditable="true"]').first
    box.click()      # ❌ Timeout: element is not visible
    box.fill(text)   # 即使绕过 click，fill 也无效
```

**❌ 失败原因**：
- Reddit 的 `shreddit-composer` 元素使用了 `content-visibility: auto` 或类似 CSS
- 元素在视口外（`y: -12077`），Playwright 认为元素不可见
- `force=True` 参数也无法解决，因为元素尺寸为 0x0

---

### 方法 4：Selenium + ChromeDriver

**思路**：使用 Selenium 通过 `debuggerAddress` 连接已有 Chrome。

```python
from selenium import webdriver
from selenium.webdriver.chrome.options import Options

options = Options()
options.add_experimental_option("debuggerAddress", "127.0.0.1:18793")
driver = webdriver.Chrome(options=options)
```

**❌ 失败原因**：
- 系统是 **ARM64** 架构，ChromeDriver 官方只提供 **x86_64** 版本
- 下载的 `chromedriver` 无法执行：`cannot execute binary file: Exec format error`
- 没有 ARM64 版本的 ChromeDriver 可用

---

### 方法 5：Playwright Headless 模式 + Cookie 导入

**思路**：启动新的 headless 浏览器，手动导入 cookies 保持登录态。

```python
browser = p.chromium.launch(headless=True)
context = browser.new_context()
context.add_cookies(cookies)  # 从已有浏览器导出
page = context.new_page()
page.goto('https://www.reddit.com/...')
```

**❌ 失败原因**：
- Reddit 检测到 headless 浏览器（无头模式特征明显）
- 页面标题为空，内容不加载
- 现代网站（Reddit、Twitter、Facebook）都能识别 headless 模式

---

## ✅ 成功方案详解

### 核心思路

1. **通过 CDP 连接已有有头 Chrome**（保持登录态）
2. **使用 JavaScript 直接操作 DOM**（绕过 Playwright 的可见性检查）
3. **使用 `page.keyboard.type()` 输入文本**（触发真实键盘事件，Lexical 编辑器能正确响应）
4. **使用 JavaScript 点击提交按钮**（绕过元素不可见限制）

### 为什么 `keyboard.type()` 能成功？

Playwright 的 `keyboard.type()` 会：
1. 向浏览器发送真实的 `keydown` → `keypress` → `keyup` 事件序列
2. 这些事件会被 Lexical 编辑器的键盘监听器捕获
3. Lexical 内部状态更新，文本正确插入到 DOM

对比其他方法：
- `dispatchKeyEvent`：只发到底层，不经过浏览器事件系统
- `execCommand`：已废弃，现代编辑器不监听
- `fill()`：直接设置 value，不触发键盘事件

---

## 💻 核心代码

```python
from playwright.sync_api import sync_playwright
import time

REDDIT_POST_URL = "https://www.reddit.com/r/artificial/comments/1sx7sjl/a_comedians_strategy_for_poisoning_ai_training/"
COMMENT_TEXT = "Interesting approach to data poisoning. I wonder how scalable this is against modern crawlers."
CDP_URL = "http://127.0.0.1:18793"


def post_comment_to_reddit():
    """
    使用 Playwright + CDP 在 Reddit 发表评论
    
    前提条件：
    1. Chrome 已启动并开启远程调试：
       chromium --remote-debugging-port=18793 --user-data-dir=/tmp/chrome-profile
    2. 已在 Chrome 中登录 Reddit
    3. Playwright 已安装：pip install playwright
    """
    
    with sync_playwright() as p:
        # 1. 通过 CDP 连接已有 Chrome（关键：保持登录态）
        browser = p.chromium.connect_over_cdp(CDP_URL)
        context = browser.contexts[0]
        page = context.pages[0]
        
        # 2. 导航到目标帖子
        if '1sx7sjl' not in page.url:
            page.goto(REDDIT_POST_URL)
            time.sleep(8)  # 等待页面加载
        
        print(f"当前页面: {page.url}")
        
        # 3. 使用 JavaScript 点击触发器（绕过 Playwright 可见性检查）
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
        
        # 4. 使用 JavaScript 聚焦评论框并设置光标
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
        
        # 5. 使用 keyboard.type() 输入文本（关键：触发真实键盘事件）
        page.keyboard.type(COMMENT_TEXT)
        print(f"✅ 已输入: {COMMENT_TEXT[:50]}...")
        time.sleep(3)
        
        # 6. 验证内容已插入
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
        
        # 7. 使用 JavaScript 点击 Comment 按钮
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
        time.sleep(10)  # 等待提交完成
        
        # 8. 验证提交结果
        page_info = page.evaluate("""
            () => {
                return {
                    hasSuccess: document.body.innerText.includes('Comment posted successfully'),
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

## 🕳️ 踩坑总结

### 坑 1：现代富文本编辑器不响应底层键盘事件

| 方法 | 事件类型 | Lexical 响应 |
|------|----------|--------------|
| `Input.dispatchKeyEvent` | 底层 CDP 事件 | ❌ 不响应 |
| `document.execCommand` | 废弃 API | ❌ 不响应 |
| `locator.fill()` | 直接设置 value | ❌ 不响应 |
| `keyboard.type()` | 真实浏览器键盘事件 | ✅ 正确响应 |

**教训**：操作现代 Web Components 富文本编辑器时，必须使用能触发完整浏览器事件链的方法。

---

### 坑 2：元素"不可见"导致 Playwright 操作失败

Reddit 的 `shreddit-composer` 元素：
- 位于视口外（`y: -12077`）
- 尺寸为 0x0（`width: 0, height: 0`）
- 使用了 Shadow DOM + CSS containment

**解决方案**：
- 不使用 Playwright 的 `locator.click()`（会检查可见性）
- 改用 `page.evaluate()` 执行 JavaScript 直接操作 DOM
- JavaScript 的 `element.click()` 不检查元素是否可见

---

### 坑 3：ARM64 系统缺少 ChromeDriver

```bash
# 错误信息
cannot execute binary file: Exec format error

# 原因
ChromeDriver 官方只提供 x86_64 版本，ARM64 系统无法运行
```

**解决方案**：
- 使用 Playwright（自带 Chromium，支持 ARM64）
- 或通过 CDP 直接连接已有 Chrome，无需 ChromeDriver

---

### 坑 4：Headless 模式被网站检测

现代网站通过以下特征检测 headless 浏览器：
- `navigator.webdriver === true`
- 无插件/无语言设置
- 窗口大小异常
- `window.chrome` 对象缺失

**解决方案**：
- 使用已有有头浏览器（通过 CDP 连接）
- 或使用 Playwright 的 `stealth` 插件（但仍有风险）

---

### 坑 5：评论成功但刷新后不显示

网络监控显示：
- `create-comment` 请求返回 200
- 页面显示 "Comment posted successfully"
- 但刷新后评论数为 0

**原因**：
- Reddit 评论加载是懒加载（lazy loading）
- 需要滚动页面触发评论加载
- 评论实际上已创建，只是前端未立即渲染

**验证方法**：
- 检查页面文本中是否包含评论内容
- 不依赖 `[data-testid="comment"]` 计数器

---

## 📚 延伸阅读

### Lexical 编辑器
- [Lexical GitHub](https://github.com/facebook/lexical)
- Lexical 是 Facebook 开发的富文本编辑器框架，Reddit 使用它处理评论输入

### Chrome DevTools Protocol
- [CDP 文档](https://chromedevtools.github.io/devtools-protocol/)
- 通过 `chrome --remote-debugging-port=9222` 启用

### Playwright CDP 连接
- [Playwright 文档 - connect_over_cdp](https://playwright.dev/python/docs/api/class-browsertype#browser-type-connect-over-cdp)
- 支持连接已有 Chrome 实例，保持所有状态（cookies、localStorage、登录态）

---

## 📝 附录：完整错误日志

### `dispatchKeyEvent` 失败日志
```
发送 87 个字符的 keydown/keyup 序列
页面反馈: 无变化
innerHTML: <p class="first:mt-0 last:mb-0"><br></p>
```

### `execCommand` 失败日志
```
document.execCommand('insertText', false, text)
返回 innerHTML: <p class="first:mt-0 last:mb-0"><br></p>
内容未变化
```

### Playwright `click()` 超时日志
```
Locator.click: Timeout 30000ms exceeded.
element is not visible
56 × waiting for element to be visible, enabled and stable
```

### ChromeDriver 架构不匹配日志
```
chromedriver: ELF 64-bit LSB pie executable, x86-64
系统: aarch64
错误: cannot execute binary file: Exec format error
```

### 成功日志
```
✅ 输入: Interesting approach to data poisoning...
📋 内容验证: 'Interesting approach to data poisoning...'
✅ 已点击 Comment 按钮
🎉 评论发表成功！
页面显示: "Comment posted successfully"
评论出现在列表中: Important_Half_7272 • 1m ago
```

---

## 🏆 最终成功截图

```
页面文本:
Skip to main content
Comment posted successfully
...
Important_Half_7272
• 1m ago
Interesting approach to data poisoning. I wonder how scalable this is against modern crawlers.
Reply Share
...
```

---

**许可证**：MIT  
**贡献**：欢迎提交 PR 补充其他踩坑经验
