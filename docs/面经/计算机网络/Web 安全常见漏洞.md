## 1. Web 安全问题

前端访问后端时，浏览器为了安全引入了一套边界机制，但服务端错误放宽了边界：

- **同源策略（Same-Origin Policy，SOP）**：浏览器的默认隔离机制；
- **CORS（Cross-Origin Resource Sharing）**：服务端显式放宽跨源访问的协议；
- **CSRF（Cross-Site Request Forgery）**：利用浏览器自动带上凭证的特性，伪造用户请求；
- **XSS（Cross-Site Scripting）**：把恶意脚本注入页面，在受害站点上下文执行；
- **点击劫持（Clickjacking）**：把目标页面嵌进透明 `iframe`，诱导用户点击。

## 2. 同源策略

### 2.1 什么叫同源

两个 URL 是否同源，看三元组是否一致：

- 协议（scheme）
- 主机（host）
- 端口（port）

例如：

| URL 对比                                                     | 是否同源 | 原因     |
| ---------------------------------------------------------- | ---- | ------ |
| `https://a.example.com/app` 与 `https://a.example.com/user` | 是    | 只有路径不同 |
| `https://a.example.com` 与 `http://a.example.com`           | 否    | 协议不同   |
| `https://a.example.com` 与 `https://b.example.com`          | 否    | 主机不同   |
| `https://a.example.com` 与 `https://a.example.com:8443`     | 否    | 端口不同   |

### 2.2 同源策略限制什么，不限制什么

同源策略的重点不是“完全禁止跨域”，而是限制跨源读取敏感数据：

| 行为 | 默认是否允许 | 说明 |
| --- | --- | --- |
| 跨站读取接口响应（`fetch`、`XMLHttpRequest`） | 否 | 典型受 SOP 约束 |
| 跨站提交表单 | 通常允许 | 这是 CSRF 能成立的前提 |
| 跨站加载图片、脚本、样式 | 通常允许 | 属于跨站嵌入，不等于可读响应内容 |
| 跨站读取 `localStorage`、DOM | 否 | 浏览器隔离页面上下文 |

### 2.3 为什么必须有同源策略

如果没有同源策略，恶意网站可以在用户已登录银行、邮箱、企业后台的情况下，直接用脚本读取这些站点返回的数据，再把数据上传给攻击者。

也就是说，同源策略保护的是：

- 用户在其他站点的登录态；
- 站内页面的 DOM 与脚本上下文；
- 同站点的 Cookie、存储、接口响应等敏感内容。

## 3. CORS：跨源资源共享，不是漏洞本身

### 3.1 CORS 解决什么问题

现代前后端分离很常见，例如：

- 前端站点：`https://app.example.com`
- API 服务：`https://api.example.com`

这两个域名不同，默认跨源读取会被浏览器拦截。所以服务端需要用 `CORS` 响应头告诉浏览器：**我允许哪个源访问我**。

### 3.2 CORS 的核心响应头

| 响应头 | 作用 | 常见风险 |
| --- | --- | --- |
| `Access-Control-Allow-Origin` | 允许哪个源读取响应 | 配成 `*` 或错误回显 Origin |
| `Access-Control-Allow-Credentials` | 是否允许携带凭证 | 和宽松 Origin 组合后风险很高 |
| `Access-Control-Allow-Methods` | 允许的方法 | 过宽会放大攻击面 |
| `Access-Control-Allow-Headers` | 允许的请求头 | 过宽会让前端可发送更多高权限请求 |
| `Access-Control-Max-Age` | 预检缓存时间 | 配置过长会放大错误配置影响窗口 |

一个最基本的例子：

```shell
HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://app.example.com
Vary: Origin
```

这表示只有 `https://app.example.com` 可以读取该响应。

### 3.3 简单请求与预检请求

浏览器会把跨源请求分成两类：

- **简单请求**：例如普通表单风格的 `GET`、`POST`；
- **非简单请求**：例如带自定义请求头、`application/json`、`PUT`、`DELETE` 等，浏览器会先发一次 `OPTIONS` 预检。

典型预检过程如下：

```shell
OPTIONS /transfer HTTP/1.1
Origin: https://app.example.com
Access-Control-Request-Method: POST
Access-Control-Request-Headers: Content-Type, X-CSRF-Token
```

```shell
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: POST
Access-Control-Allow-Headers: Content-Type, X-CSRF-Token
Access-Control-Allow-Credentials: true
```

预检通过后，浏览器才会发真实请求。

### 3.4 CORS 的常见错误配置

真正的风险点通常不是“有 CORS”，而是**CORS 配错了**。

高频错误包括：

- `Access-Control-Allow-Origin: *` 暴露本不该公开的接口；
- 把请求头里的 `Origin` 原样回显，但没有白名单校验；
- 给敏感接口同时返回 `Access-Control-Allow-Origin: https://evil.example` 与 `Access-Control-Allow-Credentials: true`；
- 测试环境为了图省事全开放，结果配置被带到生产。

需要特别记住一条规则：

- **带凭证请求时，`Access-Control-Allow-Origin` 不能是 `*`，必须是明确源。**

## 4. CSRF：利用浏览器自动带凭证伪造请求

### 4.1 CSRF 为什么会成立

`CSRF` 的核心前提是：

1. 用户已经登录目标站点；
2. 浏览器会自动携带该站点的认证凭证，常见是 Cookie；
3. 服务端只根据 Cookie 判断“是不是本人”，没有额外校验“是不是该页面主动发起”。

于是攻击者就可以诱导用户访问恶意页面，偷偷向目标站点发请求：

```html
<form action="https://bank.example.com/transfer" method="POST">
  <input type="hidden" name="to" value="attacker" />
  <input type="hidden" name="amount" value="10000" />
</form>
<script>
  document.forms[0].submit();
</script>
```

浏览器虽然不会把银行响应内容暴露给恶意站点，但**请求本身已经发出去了，而且可能已经成功修改状态**。这就是为什么“读不到响应”并不等于“没有风险”。

### 4.2 CSRF 和 CORS 的关系

很多人会把这两个概念混掉。正确关系是：

- `CORS` 主要解决“跨站读响应”；
- `CSRF` 主要利用“跨站能发请求且能自动带 Cookie”；
- **即使没有放开 CORS，CSRF 仍然可能成功。**

只有当你的服务端把跨站有凭证请求也错误放开时，`CORS` 才可能进一步放大 `CSRF` 风险。

### 4.3 CSRF 的典型防护手段

#### 4.3.1 CSRF Token

最经典的方案是：服务端在页面里下发一个**不可预测的随机 Token**，前端提交表单或请求时一并带上，服务端校验通过才允许修改状态。

示例：

```html
<form action="/transfer" method="POST">
  <input type="hidden" name="_csrf" value="a8f91d2e4c..." />
  <input type="text" name="amount" />
</form>
```

服务端校验逻辑要点：

- Token 必须不可预测；
- Token 要与用户会话绑定；
- 只对修改状态的请求校验，不要只保护登录页；
- 校验失败要明确拒绝，不能“放行并记日志”。

#### 4.3.2 SameSite Cookie

`SameSite` 是 Cookie 的附加属性，用来控制浏览器是否在跨站请求里携带 Cookie。

| 取值 | 含义 | 适用场景 |
| --- | --- | --- |
| `Strict` | 任何跨站请求都不带 Cookie | 安全性最高，但用户体验最严格 |
| `Lax` | 顶层导航且安全方法可带 Cookie | 常见默认折中方案 |
| `None` | 允许跨站发送 Cookie | 必须同时配合 `Secure` |

要点是：

- **`SameSite` 只能作为纵深防御，不能替代 CSRF Token。**
- 一些业务需要跨站单点登录、第三方嵌入时，`SameSite` 会受到兼容性与产品形态约束。

#### 4.3.3 避免使用简单请求

如果你的前端通过 `fetch`、`XMLHttpRequest` 发送修改状态请求，可以故意把它设计成**非简单请求**，例如：

```javascript
fetch("/api/transfer", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "X-CSRF-Token": csrfToken
  },
  body: JSON.stringify({ to: "u1001", amount: 100 })
});
```

这样浏览器跨站发起时通常会先触发预检，恶意页面不能像普通表单那样轻易构造出完全等价的请求。

但要注意一个反直觉点：

- **如果你把 CORS 配得过于宽松，非简单请求这层保护也会被你自己放开。**

#### 4.3.4 校验请求来源

工程上还会结合这些请求头做辅助校验：

- `Origin`
- `Referer`
- `Sec-Fetch-Site`

其中 `Sec-Fetch-Site` 是浏览器附带的 Fetch Metadata 请求头之一，可以帮助服务端判断请求是：

- `same-origin`
- `same-site`
- `cross-site`

它适合做一层额外的拦截，但仍然更适合作为**补充防线**，而不是唯一防线。

### 4.4 什么时候 CSRF 风险会明显降低

以下场景里，传统 `CSRF` 风险会下降，但不是绝对没有：

- 认证不放在 Cookie，而是放在 `Authorization: Bearer <token>`；
- Token 由前端脚本主动读取并显式加到请求头里；
- 服务端拒绝所有跨站有状态修改请求。

不过如果站点存在 `XSS`，攻击脚本可能直接在站内上下文里发合法请求，所以**XSS 经常能绕过基于页面上下文的 CSRF 防线**。

## 5. XSS：把恶意脚本注入受害站点页面

### 5.1 XSS 的本质

`XSS` 不是“弹个框”这么简单，它的本质是：**攻击者让浏览器把不可信输入当成脚本或可执行 HTML 来解释**。

一旦成功，攻击者可能做到：

- 读取页面上的敏感数据；
- 冒用当前用户调用接口；
- 窃取非 `HttpOnly` Cookie、Token、表单数据；
- 篡改 DOM，诱导用户输入密码、验证码；
- 作为跳板继续发起 CSRF、业务欺诈或横向移动。

### 5.2 XSS 的主要类型

| 类型 | 特征 | 例子 |
| --- | --- | --- |
| 反射型 XSS | 恶意输入经服务端拼接后立即回显 | 搜索页把查询词原样拼进 HTML |
| 存储型 XSS | 恶意脚本被存进数据库，后续页面渲染时执行 | 评论区、用户昵称、富文本简介 |
| DOM 型 XSS | 前端脚本在浏览器里把不可信数据直接写入危险 DOM sink | `innerHTML`、`document.write` |

### 5.3 为什么模板引擎通常能防一部分 XSS

现代模板引擎和前端框架一般会做**输出编码（Output Encoding）**，把 `<`、`>`、`"` 等字符转义成普通文本，而不是让浏览器当作标签或脚本执行。

例如：

```html
<div>{{ userName }}</div>
```

如果 `userName` 是：

```html
<img src=x onerror=alert('xss')>
```

经过正确编码后，浏览器只会把它显示成文本，而不会执行。

### 5.4 为什么仍然会出现 XSS

因为很多漏洞不是出在“普通文本插值”，而是出在**把不可信输入放进危险上下文**。典型错误包括：

- 使用 `innerHTML`、`outerHTML`、`insertAdjacentHTML`；
- 在 HTML 属性、URL、内联事件里直接拼接用户输入；
- 对富文本只做“黑名单替换”，没有做可信白名单清洗；
- React 中滥用 `dangerouslySetInnerHTML`；
- Vue 中滥用 `v-html`。

危险示例：

```javascript
const keyword = new URLSearchParams(location.search).get("q");
document.querySelector("#box").innerHTML = keyword; // 危险：直接进入 HTML sink
```

### 5.5 XSS 的典型防护手段

#### 5.5.1 输出编码优先

如果业务语义是“把输入当文本显示”，就应该做上下文相关的编码：

- HTML 上下文做 HTML 编码；
- HTML 属性上下文做属性编码；
- URL 上下文做 URL 编码；
- JavaScript 字符串上下文做 JS 编码。

关键原则是：

- **不要试图用一个通用替换函数解决所有上下文。**

#### 5.5.2 不可信 HTML 必须做白名单清洗

如果业务确实允许富文本，例如评论、公告、富文本编辑器内容，就不能只靠编码，而应使用成熟的 HTML Sanitizer，只允许少量安全标签和属性。

工程上常见思路：

- 允许 `p`、`br`、`strong`、`ul`、`li` 等安全标签；
- 禁止 `script`、事件属性、危险协议如 `javascript:`；
- 服务端与前端都要理解这套规则，避免出现“双端口径不一致”。

#### 5.5.3 配置 CSP

`CSP`（Content Security Policy）可以减少 XSS 成功后的危害面，例如：

```shell
Content-Security-Policy: default-src 'self'; script-src 'self' 'nonce-rAnd0m'; object-src 'none'; base-uri 'self'
```

它不能替代编码和清洗，但能显著收紧脚本来源、内联脚本执行和资源加载边界。

#### 5.5.4 尽量避免危险 DOM API

能用 `textContent`、`innerText` 的地方，不要用 `innerHTML`。

安全示例：

```javascript
const keyword = new URLSearchParams(location.search).get("q");
document.querySelector("#box").textContent = keyword; // 安全：按纯文本写入
```

#### 5.5.5 Trusted Types

在现代浏览器环境中，可以结合 `CSP` 的 `require-trusted-types-for` 指令，限制把普通字符串直接传给危险 DOM sink，从机制上约束团队不要随意把未净化字符串塞进 `innerHTML`。

它更像是**工程治理工具**，不是单点修复手段。

## 6. 点击劫持：诱导用户点击真实页面

### 6.1 攻击方式

点击劫持通常会把目标页面嵌进透明或半透明 `iframe`，覆盖在诱导按钮下面。用户以为自己点的是“抽奖”“播放”“领取优惠券”，实际上点到的是目标站点里的“转账”“授权”“关注”等敏感按钮。

### 6.2 为什么它能成立

它利用的是：

- 目标页面允许被其他站点嵌入；
- 用户已登录目标站点；
- 关键操作依赖用户点击，且页面没有额外确认。

### 6.3 防护手段

两个核心响应头：

```shell
Content-Security-Policy: frame-ancestors 'none'
X-Frame-Options: DENY
```

说明：

- `CSP: frame-ancestors` 是更现代、更灵活的方案；
- `X-Frame-Options` 是老方案，常见值有 `DENY`、`SAMEORIGIN`；
- 这两个都必须通过 **HTTP 响应头** 下发，不能依赖 `<meta>` 标签。

## 7. 关系与对比

| 问题 | 核心机制 | 主要危害 | 关键防护 |
| --- | --- | --- | --- |
| 同源策略 | 浏览器默认隔离 | 防止跨站读取敏感数据 | 浏览器内建机制 |
| CORS 错配 | 服务端错误放宽跨源读取 | 敏感接口被任意站点读取 | 精确 Origin 白名单，谨慎开启凭证 |
| CSRF | 利用自动带 Cookie 伪造请求 | 代替用户执行转账、改密、下单 | CSRF Token、SameSite、来源校验 |
| XSS | 注入脚本到站内执行 | 读页面、盗 Token、伪造操作 | 输出编码、清洗、CSP、避免危险 API |
| 点击劫持 | `iframe` 透明覆盖诱导点击 | 诱导授权、转账、关注 | `frame-ancestors`、`X-Frame-Options` |

## 8. 工程落地：一个 Web 系统最少要做哪些防护

### 8.1 服务端基线

- 所有敏感接口都必须做认证和授权，不能指望 CORS 挡攻击者；
- 会修改状态的接口默认开启 CSRF 防护；
- Cookie 至少配置 `HttpOnly`、`Secure`，并评估 `SameSite=Lax` 或 `Strict`；
- CORS 只对白名单源开放，不允许无脑回显 `Origin`；
- 返回 HTML 的页面统一加 `CSP`、`X-Frame-Options` 或 `frame-ancestors`；
- 登录、转账、改密、绑定手机等高风险操作增加二次确认或验证码。

### 8.2 前端基线

- 不把用户输入直接写进 `innerHTML`；
- 富文本展示统一走经过审计的 Sanitizer；
- 状态修改请求统一走封装层，自动带 `CSRF Token`；
- 不在 `localStorage` 长期存放高价值敏感凭证，尤其要考虑 `XSS` 风险；
- 对外链、跳转、文件预览、第三方组件引入做额外审查。

### 8.3 安全测试基线

- 用浏览器 DevTools 检查 `CORS`、Cookie 属性、响应头；
- 对搜索框、评论区、昵称、富文本做 XSS 探测；
- 对表单、订单、转账、资料修改接口做 CSRF 验证；
- 验证页面是否能被第三方 `iframe` 嵌入；
- 验证预发、灰度、管理后台是否残留“测试全开放”配置。
