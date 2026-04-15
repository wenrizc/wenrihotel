## 1. WebSocket 的核心定位

WebSocket 是一种**在单条 TCP 连接上实现全双工、低开销、长连接通信**的协议。它通常先借助一次 HTTP/1.1 请求完成握手升级，随后双方传输的不再是 HTTP 报文，而是 WebSocket 帧。

它解决的核心问题是：**HTTP 请求-响应模型不适合高频实时双向通信**。如果用轮询、长轮询实现聊天室、行情推送、协同编辑，会产生大量重复请求、连接管理开销和较高延迟。

## 2. WebSocket 与 HTTP 的主要区别

| 维度 | WebSocket | HTTP |
| --- | --- | --- |
| 通信模型 | 全双工，双方都可主动发消息 | 半双工，请求-响应 |
| 连接形态 | 一次握手，长期复用同一条 TCP 连接 | 通常按请求处理 |
| 协议开销 | 建连后帧头很小 | 每次请求都带较完整的 Header |
| 适用场景 | 聊天、推送、实时协同、游戏 | 页面加载、表单提交、常规 API |
| 服务端推送 | 原生支持 | 需轮询、长轮询、SSE 等变通方案 |

## 3. 建立连接的过程

### 3.1 握手为什么基于 HTTP

WebSocket 没有重新发明一套端口发现和代理穿透机制，而是复用 HTTP 常见的 80/443 端口和代理链路。这样更容易穿过防火墙、网关和浏览器安全模型。

### 3.2 典型握手报文

客户端先发起升级请求：

```http
GET /ws/chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
Origin: https://example.com
```

服务端校验通过后返回：

```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

### 3.3 `Sec-WebSocket-Accept` 是怎么来的

服务端会把客户端传来的 `Sec-WebSocket-Key` 与固定 GUID：

`258EAFA5-E914-47DA-95CA-C5AB0DC85B11`

拼接后做 `SHA-1`，再进行 `Base64` 编码，结果放入 `Sec-WebSocket-Accept`。这样可以证明服务端理解 WebSocket 协议，而不是把这次请求当成普通 HTTP 请求处理。

## 4. WebSocket 帧的数据格式是怎么设计的

### 4.1 帧结构总览

连接建立后，应用层数据会被切成一个个 WebSocket 帧。基础格式如下：

| 字段 | 位数 | 作用 |
| --- | --- | --- |
| `FIN` | 1 bit | 是否为当前消息的最后一帧 |
| `RSV1/RSV2/RSV3` | 各 1 bit | 为扩展预留，常见于压缩扩展 |
| `Opcode` | 4 bit | 帧类型，如文本、二进制、关闭、心跳 |
| `MASK` | 1 bit | 是否带掩码 |
| `Payload len` | 7 bit | 载荷长度或长度标记 |
| `Extended payload length` | 16/64 bit | 当载荷较大时补充真实长度 |
| `Masking key` | 32 bit | 客户端发给服务端时必须携带 |
| `Payload data` | N byte | 业务数据 |

### 4.2 常见 `Opcode`

| `Opcode` | 含义 |
| --- | --- |
| `0x0` | 延续帧（Continuation） |
| `0x1` | 文本帧（Text） |
| `0x2` | 二进制帧（Binary） |
| `0x8` | 关闭帧（Close） |
| `0x9` | `Ping` |
| `0xA` | `Pong` |

### 4.3 长度字段为什么这样设计

WebSocket 的长度设计兼顾了**小消息高频传输**和**大消息兼容性**：

- 长度小于等于 `125` 字节时，直接放在 `Payload len` 中，帧头最短只有 2 字节。
- 长度等于 `126` 时，后续再用 16 bit 表示真实长度。
- 长度等于 `127` 时，后续再用 64 bit 表示真实长度。

这样做的好处是：**聊天、通知这类小消息头部极小，大文件或音视频分片又不会被协议限制死。**

### 4.4 为什么客户端必须 Mask

浏览器发往服务端的帧必须带 `Masking key`，服务端收到后需要做一次异或还原。其核心目的不是保密，而是**避免中间代理把 WebSocket 数据误识别为普通 HTTP 流量，从而导致缓存投毒、协议混淆等问题**。

注意：

- **客户端到服务端必须 mask。**
- **服务端到客户端通常不 mask。**

### 4.5 为什么支持分片

WebSocket 允许一个完整消息拆成多个帧发送，靠 `FIN` 和 `Continuation` 帧拼起来。这样设计主要有三个原因：

- 避免超大消息一次性占满发送缓冲区。
- 允许控制帧插队，比如长消息传输中间仍能插入 `Ping/Pong`。
- 便于和压缩、流式处理结合。

### 4.6 控制帧的约束

控制帧包括 `Close`、`Ping`、`Pong`，它们有两个重要限制：

- **控制帧负载长度不能超过 `125` 字节。**
- **控制帧不能被分片。**

这是为了保证控制语义简单、可快速处理，不会被超长消息阻塞。

## 5. 前端是如何创建和发送 WebSocket 的

### 5.1 浏览器中如何创建连接

前端通常直接使用浏览器原生 `WebSocket` API：

```javascript
const ws = new WebSocket("wss://example.com/ws/chat", ["json"]);

ws.onopen = () => {
  ws.send(JSON.stringify({ type: "join", roomId: 1 }));
};

ws.onmessage = (event) => {
  const message = JSON.parse(event.data);
  console.log("receive:", message);
};

ws.onerror = (event) => {
  console.error("websocket error", event);
};

ws.onclose = (event) => {
  console.log("closed", event.code, event.reason);
};
```

### 5.2 前端调用 `send()` 后发生了什么

前端代码只是在逻辑层调用 `ws.send(data)`，真正的底层工作由浏览器完成：

1. 浏览器检查 `readyState` 是否为 `OPEN`。
2. 把字符串编码为 UTF-8，或直接处理 `ArrayBuffer` / `Blob`。
3. 根据数据类型组装 `Text` 或 `Binary` 帧。
4. 生成 masking key，对 payload 做异或。
5. 把帧写入浏览器网络栈，再交给 TCP 发送。

也就是说，**前端开发者通常是“按消息编程”，浏览器是“按帧编程”**。开发者看见的是消息，浏览器负责拆成帧、掩码、重传、粘包拆包处理。

### 5.3 前端需要关心的几个 API

| API / 属性 | 作用 |
| --- | --- |
| `new WebSocket(url)` | 创建连接 |
| `onopen` | 握手成功后的回调 |
| `onmessage` | 收到服务端消息 |
| `send(data)` | 发送字符串或二进制数据 |
| `close(code, reason)` | 主动关闭连接 |
| `readyState` | 当前状态 |
| `bufferedAmount` | 尚未真正发出的字节数 |

### 5.4 前端常见实践

- 一般会先发送认证信息，或把 `token` 放在 URL / Cookie / 首包中。
- 应用层最好自定义消息体，如 `{type, requestId, data}`，便于扩展。
- 大消息要考虑压缩和拆分，不能无限制 `send()`。
- 断线重连要带退避策略，不能无脑瞬时重连。

## 6. 后端是如何处理 WebSocket 的

### 6.1 后端处理的本质

后端通常不是“收一个 HTTP 请求，回一个 HTTP 响应”，而是维护一个**长生命周期连接对象**。连接建立后，服务端会持续执行以下动作：

1. 处理握手升级。
2. 保存连接上下文。
3. 循环读取帧并解析。
4. 把帧聚合成完整消息。
5. 执行业务逻辑。
6. 再把响应编码成 WebSocket 帧返回。

### 6.2 服务端的典型处理链路

| 阶段 | 服务端要做什么 |
| --- | --- |
| 握手阶段 | 校验 `Upgrade`、`Connection`、版本、来源、鉴权 |
| 建连阶段 | 创建会话对象，记录用户、房间、订阅关系 |
| 收包阶段 | 解析帧头、长度、mask，并对 payload 解码 |
| 聚合阶段 | 如果是分片帧，组装成完整消息 |
| 分发阶段 | 根据消息类型路由到聊天、通知、订阅等处理器 |
| 回包阶段 | 编码为 Text/Binary/Ping/Pong/Close 帧发回 |
| 断连阶段 | 清理连接、订阅、资源占用 |

### 6.3 一个简化的后端示例

下面以 `Spring` 的 `TextWebSocketHandler` 为例：

```java
public class ChatWebSocketHandler extends TextWebSocketHandler {

    @Override
    public void afterConnectionEstablished(WebSocketSession session) throws Exception {
        // 建立连接后，可以绑定用户和会话信息
        session.sendMessage(new TextMessage("{\"type\":\"connected\"}"));
    }

    @Override
    protected void handleTextMessage(WebSocketSession session, TextMessage message) throws Exception {
        String payload = message.getPayload();
        // 这里通常会做鉴权、反序列化、消息路由
        session.sendMessage(new TextMessage("{\"type\":\"ack\",\"data\":" + payload + "}"));
    }

    @Override
    public void afterConnectionClosed(WebSocketSession session, CloseStatus status) throws Exception {
        // 连接关闭后，清理会话和订阅关系
    }
}
```

这个例子只体现“消息级”处理。更底层的框架，比如 `Netty`，还会显式处理 HTTP 升级、帧解码、聚合和心跳。

### 6.4 后端真正要补上的能力

光能“收发消息”还不够，工程上通常还要补这些能力：

- **鉴权**：连接建立时确认用户身份，并处理过期、顶号、踢人。
- **会话管理**：用户和连接可能是一对多，例如一个用户多个设备在线。
- **消息顺序**：单连接内通常天然有序，多连接、多节点场景需要业务序号。
- **消息确认**：重要消息可以设计 `ack` 机制，避免只靠 TCP 成功送到内核缓冲区。
- **广播与路由**：房间广播、用户单播、集群跨节点转发常依赖 Redis、MQ 或网关层。
- **限流与背压**：防止慢客户端拖垮发送队列。

## 7. WebSocket 发送一条消息的完整过程

### 7.1 前端发消息到服务端

1. 业务代码调用 `ws.send()`。
2. 浏览器把业务数据编码成 WebSocket 消息。
3. 浏览器把消息封装为一个或多个帧，并添加 mask。
4. 帧交给 TCP 协议栈，切成 TCP segment 发出。
5. 网络设备转发到服务端机器。
6. 服务端内核收到 TCP 数据，放入 socket 接收缓冲区。
7. WebSocket 框架从 socket 读取字节流，解析帧结构。
8. 服务端执行 unmask、拼帧、反序列化，得到业务消息。
9. 业务处理器执行逻辑，比如鉴权、入库、广播。

### 7.2 服务端把消息回给客户端

1. 业务层生成响应消息。
2. WebSocket 框架把消息编码成 Text 或 Binary 帧。
3. 服务端把帧写入 socket 发送缓冲区。
4. 数据经 TCP 发往客户端。
5. 浏览器网络栈接收字节流并解析帧。
6. 浏览器把完整消息投递给 `onmessage` 回调。
7. 前端代码更新页面、状态管理或本地缓存。

### 7.3 这条链路上分别由谁保证什么

| 层次 | 负责内容 |
| --- | --- |
| WebSocket 协议 | 消息边界、帧类型、心跳、关闭语义 |
| TCP | 字节流传输、重传、顺序、流控、拥塞控制 |
| TLS (`wss`) | 加密、防窃听、防篡改 |
| 业务协议 | 消息类型、鉴权、幂等、顺序号、确认机制 |

要注意：**TCP 只保证字节流可靠送达，不保证你的业务“消息已被处理成功”。** 如果消息不能丢，就要在应用层补 `ack`、重试、去重、幂等。

## 8. 开发时需要注意什么

### 8.1 鉴权不要只做一次“表面登录”

很多系统只是连接建立时校验一次 `token`，后续完全信任连接，这容易出问题。更稳妥的方式是：

- 握手阶段完成基础鉴权。
- 会话中保留用户身份、租户、权限信息。
- 对敏感消息继续做业务权限校验。
- 处理 `token` 过期、用户被踢下线、权限变更。

### 8.2 心跳机制要分清层次

- TCP 有 `keepalive`，但默认周期很长，且不够灵活。
- WebSocket 有 `Ping/Pong`，适合协议层保活。
- 业务层还可以定义心跳消息，用于统计 RTT、用户在线状态、链路健康。

通常线上更常用的是：**应用层心跳 + 服务端空闲连接回收**。

### 8.3 长连接场景必须关注负载均衡

WebSocket 连接建立后会长期停留在某台机器上，因此和普通 HTTP 请求不一样：

- 如果连接状态保存在本机内存，需要粘性会话。
- 如果要无状态扩容，需要把会话路由、订阅关系、用户状态放入共享层。
- 广播消息常要做跨节点分发，否则用户只会收到本机连接的推送。

### 8.4 注意消息大小和背压

不能假设客户端永远消费得过来，也不能假设网络永远稳定：

- 限制单条消息最大长度，防止内存被打爆。
- 关注发送队列长度，避免慢连接积压。
- 大消息尽量压缩、分片或改走对象存储。
- 前端可观察 `bufferedAmount`，后端可监控发送缓冲区和队列深度。

### 8.5 断线重连不是简单重连

重连会带来三个典型问题：

- **重复连接**：用户旧连接未完全清理，新连接又上来了。
- **消息丢失**：断线期间的消息是否补发。
- **消息重复**：重连补偿时同一消息可能收到两次。

常见做法是增加：

- 会话唯一标识。
- 消息序号或游标。
- `ack` / 补拉机制。
- 幂等消费。

### 8.6 WebSocket 不一定适合所有实时场景

如果只是服务端单向推送，且不要求双向交互，`SSE` 可能更简单。若消息量极低、链路无需常驻，也可能普通 HTTP 更合适。**不要为了“实时”就默认上 WebSocket。**

