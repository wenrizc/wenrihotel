
#### WebSocket 技术概述
- **定义**：
  - WebSocket 是一种基于 TCP 的全双工通信协议，允许客户端和服务器之间建立持久连接，实现实时双向数据传输。
- **目的**：
  - 解决 HTTP 轮询（如 AJAX）效率低、实时性差的问题。
- **核心特点**：
  1. **全双工**：客户端和服务器可同时发送和接收数据。
  2. **持久连接**：一次握手后保持连接。
  3. **轻量协议**：头部开销小，效率高。

#### 核心点
- WebSocket 提供高效、实时的通信机制。

---

### 1. WebSocket 原理
#### (1) 协议基础
- **基于 TCP**：
  - 在 TCP 上运行，与 HTTP 同属应用层。
- **URL 格式**：
  - `ws://example.com`（非加密）。
  - `wss://example.com`（加密，基于 TLS）。
- **协议标识**：
  - RFC 6455（2011），HTTP/1.1 的扩展。

#### (2) 连接建立
- **握手过程**：
  1. **客户端发起请求**：
     - 通过 HTTP 发送 Upgrade 请求。
```
GET /chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
```
  1. **服务器响应**：
     - 返回 101 状态码，确认切换协议。
```
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```
  1. **连接成功**：
     - 建立 WebSocket 通道，之后不再用 HTTP。

- **关键头**：
  - `Sec-WebSocket-Key`：客户端生成随机密钥。
  - `Sec-WebSocket-Accept`：服务器计算响应密钥。

#### (3) 数据传输
- **帧格式**：
  - 数据分帧传输，包含：
    - **Opcode**：数据类型（文本 0x1、二进制 0x2 等）。
    - **Payload**：实际数据。
    - **Mask**：客户端发数据需掩码，服务器无需。
- **特点**：
  - 小头部（2-14 字节），比 HTTP 轻量。
- **示例**：
  - 客户端发送：`Hello`。
  - 服务器回复：`Hi`。

#### (4) 连接关闭
- **方式**：
  - 发送关闭帧（Opcode 0x8）。
  - 状态码（如 1000 表示正常关闭）。
- **示例**：
```javascript
ws.close(1000, "Normal closure");
```

---

### 2. WebSocket 特点
#### (1) 全双工通信
- 客户端和服务器可同时收发消息。
- 对比 HTTP：请求-响应单向。

#### (2) 持久连接
- 一次握手后保持连接，避免反复建立。
- 对比 HTTP：每次请求新建连接（HTTP/1.1 有 Keep-Alive 但仍有限）。

#### (3) 低开销
- 头部小（2 字节起），适合频繁通信。
- HTTP 头部几十字节甚至更多。

#### (4) 跨域支持
- 支持 CORS，需服务器允许。

#### (5) 实时性
- 无轮询延迟，数据即时推送。

---

### 3. 与 HTTP 对比
| **特性**     | **HTTP**            | **WebSocket**       |
|--------------|---------------------|---------------------|
| **连接**     | 短连接/Keep-Alive  | 持久连接           |
| **通信**     | 单向（请求-响应）  | 全双工             |
| **头部**     | 较大               | 较小               |
| **实时性**   | 需轮询实现         | 原生支持           |
| **协议**     | HTTP/1.1, 2        | WS/WSS             |

---

### 4. WebSocket API（客户端）
- **JavaScript 示例**：
```javascript
// 创建连接
const ws = new WebSocket("ws://example.com/chat");

// 连接成功
ws.onopen = () => console.log("Connected");

// 接收消息
ws.onmessage = (event) => console.log("Received:", event.data);

// 发送消息
ws.send("Hello Server");

// 连接关闭
ws.onclose = () => console.log("Disconnected");

// 错误处理
ws.onerror = (error) => console.error("Error:", error);
```

---

### 5. 服务端实现
- **框架支持**：
  - **Java**：Spring WebSocket、Netty。
  - **Node.js**：`ws` 库。
- **示例（Spring）**：
```java
@ServerEndpoint("/chat")
public class ChatEndpoint {
    @OnMessage
    public void onMessage(String message, Session session) {
        session.getBasicRemote().sendText("Echo: " + message);
    }
}
```

---

### 6. 应用场景
- **实时通信**：
  - 聊天应用（如 WhatsApp）。
- **数据推送**：
  - 股票价格更新。
- **协作工具**：
  - 在线文档编辑（Google Docs）。
- **游戏**：
  - 多人在线游戏。

---

### 7. 优缺点
#### 优点
- 实时性高，延迟低。
- 双向通信，适合动态交互。
- 节省带宽，减少轮询。

#### 缺点
- 服务器需维持长连接，内存压力大。
- 不支持 HTTP 缓存。
- 兼容性问题（老浏览器需降级）。

---

### 8. 延伸与面试角度
- **与 SSE（Server-Sent Events）**：
  - WebSocket：双向。
  - SSE：单向推送，基于 HTTP。
- **心跳机制**：
  - 定时发送 `PING/PONG` 检测连接。
- **面试点**：
  - 问“原理”时，提握手和帧。
  - 问“场景”时，提聊天和推送。

---

### 总结
WebSocket 通过 HTTP 握手建立 TCP 持久连接，提供全双工、低开销的实时通信，广泛用于聊天、推送等场景。面试时，可提握手过程或写简单代码，展示理解深度。