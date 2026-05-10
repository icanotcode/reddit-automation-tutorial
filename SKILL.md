---
name: reddit-automation-websocket
description: Reddit自动化操作 - 使用WebSocket/CDP方式点赞和评论
version: 1.0
tags: [reddit, automation, cdp, websocket, social-media]
---

# Reddit 自动化操作（WebSocket/CDP 方式）

## 前置条件
- Chrome/Chromium 已启动并开启远程调试：`chromium --remote-debugging-port=18793`
- 已在浏览器中登录 Reddit
- Python 依赖：`pip install websocket-client requests`

## 核心原则

Reddit 使用 **Shadow DOM** 和 **Lexical 编辑器**，必须通过 JavaScript 直接操作 DOM，不能依赖 Playwright 的可见性检查或 `fill()` 方法。

## 1. 点赞操作

### 方法
通过 `shreddit-post` 元素的 Shadow DOM 点击第一个按钮（upvote）。

```python
import websocket
import json
import requests

def upvote_reddit_post(post_url, cdp_port=18793):
    # 获取 DevTools 页面列表
    pages = requests.get(f"http://127.0.0.1:{cdp_port}/json/list").json()
    ws_url = pages[0]['webSocketDebuggerUrl']
    
    # 连接 WebSocket
    ws = websocket.create_connection(ws_url)
    
    # 导航到帖子（如需要）
    if post_url:
        ws.send(json.dumps({
            "id": 1,
            "method": "Runtime.evaluate",
            "params": {
                "expression": f"window.location.href = '{post_url}'",
                "returnByValue": True
            }
        }))
        ws.recv()
        time.sleep(5)
    
    # 执行点赞
    ws.send(json.dumps({
        "id": 2,
        "method": "Runtime.evaluate",
        "params": {
            "expression": """
                (() => {
                    let post = document.querySelector('shreddit-post');
                    if (post && post.shadowRoot) {
                        let buttons = post.shadowRoot.querySelectorAll('button');
                        if (buttons.length >= 1) {
                            buttons[0].click();
                            return 'Upvote clicked';
                        }
                    }
                    return 'Failed';
                })()
            """,
            "returnByValue": True
        }
    }))
    
    response = ws.recv()
    result = json.loads(response)
    print(result['result']['result']['value'])
    
    ws.close()
```

### 验证点赞成功
检查按钮颜色是否变为橙色：
```javascript
let post = document.querySelector('shreddit-post');
let color = window.getComputedStyle(post.shadowRoot.querySelectorAll('button')[0]).color;
// 已点赞 = rgb(255, 69, 0) 或类似橙色
// 未点赞 = rgb(128, 128, 128) 或类似灰色
```

## 2. 评论操作

### 关键：Lexical 编辑器需要通过 JS 直接设置文本

**重要发现**：使用 `Input.dispatchKeyEvent` 模拟键盘输入会导致文本乱序（Lexical 编辑器状态更新跟不上）。**正确方法是通过 JavaScript 直接设置 `innerText`**。

```python
import websocket
import json
import requests
import time

def comment_reddit_post(post_url, comment_text, cdp_port=18793):
    pages = requests.get(f"http://127.0.0.1:{cdp_port}/json/list").json()
    ws_url = pages[0]['webSocketDebuggerUrl']
    ws = websocket.create_connection(ws_url)
    
    # 导航到帖子
    ws.send(json.dumps({
        "id": 1,
        "method": "Runtime.evaluate",
        "params": {
            "expression": f"window.location.href = '{post_url}'",
            "returnByValue": True
        }
    }))
    ws.recv()
    time.sleep(5)
    
    # 聚焦评论框
    focus_script = """
        (() => {
            let host = document.querySelector('comment-composer-host');
            let trigger = host?.querySelector('faceplate-textarea-input');
            if (trigger) { trigger.click(); trigger.focus(); }
            
            let box = host?.querySelector('[contenteditable="true"]');
            if (box) {
                box.focus();
                box.click();
                return 'Focused';
            }
            return 'No box';
        })()
    """
    
    ws.send(json.dumps({
        "id": 2,
        "method": "Runtime.evaluate",
        "params": {"expression": focus_script, "returnByValue": True}
    }))
    ws.recv()
    time.sleep(2)
    
    # 通过 JavaScript 直接设置文本内容（避免键盘事件乱序）
    # 注意：需要对单引号进行转义
    safe_text = comment_text.replace("'", "\\'")
    js_script = f"""
        (() => {{
            let host = document.querySelector('comment-composer-host');
            let box = host?.querySelector('[contenteditable="true"]');
            if (!box) return 'No editor found';
            
            // 直接设置 innerText
            box.innerText = '{safe_text}';
            
            // 触发 input 事件通知 Lexical 编辑器
            let event = new InputEvent('input', {{
                bubbles: true,
                cancelable: true,
                data: '{safe_text}',
                inputType: 'insertText'
            }});
            box.dispatchEvent(event);
            
            return 'Text set via JS';
        }})()
    """
    
    ws.send(json.dumps({
        "id": 3,
        "method": "Runtime.evaluate",
        "params": {"expression": js_script, "returnByValue": True}
    }))
    ws.recv()
    time.sleep(2)
    
    # 点击 Comment 按钮
    ws.send(json.dumps({
        "id": 4,
        "method": "Runtime.evaluate",
        "params": {
            "expression": """
                (() => {
                    let buttons = Array.from(document.querySelectorAll('button'));
                    let btn = buttons.find(b => b.textContent?.trim().toLowerCase() === 'comment');
                    if (btn && !btn.disabled) {
                        btn.click();
                        return 'Comment clicked';
                    }
                    return 'No button';
                })()
            """,
            "returnByValue": True
        }
    }))
    
    response = ws.recv()
    result = json.loads(response)
    print(result['result']['result']['value'])
    
    ws.close()
```

## 常见坑

| 坑 | 原因 | 解决 |
|---|---|---|
| Playwright click() 超时 | Reddit 元素在视口外，Playwright 可见性检查失败 | 使用 JavaScript 直接操作 DOM |
| fill() 无效 | Lexical 编辑器不监听直接 DOM 修改 | 使用 JS 直接设置 `innerText` + 触发 `InputEvent` |
| 键盘事件导致文本乱序 | Lexical 编辑器状态更新跟不上逐字符输入 | **使用 JS 直接设置 `innerText` + 触发 `InputEvent`** |
| 只发 char 事件无效 | Lexical 需要 keyDown 触发输入准备状态 | 已废弃：旧版方法，现推荐 JS 直接设置 innerText |
| Shadow DOM 找不到元素 | 元素在 Shadow DOM 内部 | 使用 `element.shadowRoot.querySelector()` |
| 帖子已删除 | Page not found | 跳过，无法解决 |

## 完整教程

GitHub 仓库：https://github.com/icanotcode/reddit-automation-tutorial
