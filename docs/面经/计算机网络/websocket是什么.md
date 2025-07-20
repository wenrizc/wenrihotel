
WebSocket是一种在单个TCP连接上进行全双工通信的协议。它使得客户端和服务器之间可以进行持久性的、双向的实时数据交换。WebSocket是作为HTML5规范的一部分被引入的，旨在解决传统HTTP协议在实时应用中效率不高的问题。

WebSocket的主要特点和工作原理：

1.  持久连接（Persistent Connection）：
    *   与HTTP不同（尤其是在HTTP/1.0和HTTP/1.1的短连接或有keep-alive限制的连接模式下），WebSocket连接一旦建立，就会保持打开状态，直到显式关闭。这避免了为每次通信都重新建立TCP连接和TLS握手（如果使用WSS）的开销。

2.  全双工通信（Full-Duplex Communication）：
    *   连接建立后，客户端和服务器双方都可以随时主动向对方发送数据，而不需要像HTTP那样总是由客户端发起请求，服务器再响应。这使得实时交互成为可能。

3.  较低的开销：
    *   一旦WebSocket连接建立，后续的数据帧头部开销非常小（通常只有几个字节），远小于HTTP请求/响应头部的开销。这对于需要频繁发送小块数据的应用（如在线游戏、实时聊天、股票行情推送等）非常高效。

4.  协议升级：
    *   WebSocket连接的建立过程始于一个HTTP(S)请求，这个请求中包含特殊的头部字段（如 `Upgrade: websocket`, `Connection: Upgrade`, `Sec-WebSocket-Key`, `Sec-WebSocket-Version`等）。
    *   如果服务器支持WebSocket，它会响应该请求，并同样包含特殊的头部字段（如 `HTTP/1.1 101 Switching Protocols`, `Upgrade: websocket`, `Connection: Upgrade`, `Sec-WebSocket-Accept`等）。
    *   这个初始的HTTP(S)握手完成后，底层的TCP连接就从HTTP协议“升级”到了WebSocket协议。之后，双方就按照WebSocket协议定义的帧格式进行通信，不再使用HTTP的请求/响应模式。

5.  数据帧（Frames）：
    *   WebSocket传输的数据被分割成一个或多个帧。每个帧都有一个小的头部，指示了帧的类型（如文本帧、二进制帧、关闭帧、Ping/Pong帧等）、载荷长度等信息。
    *   数据可以是文本（UTF-8编码）或二进制格式。

6.  协议标识符：
    *   未加密的WebSocket使用 `ws://` 协议标识符。
    *   加密的WebSocket（通过TLS/SSL层保护，类似于HTTPS）使用 `wss://` 协议标识符。WSS提供了与HTTPS同等级别的安全性。

7.  适用场景：
    WebSocket非常适合需要低延迟、高频率双向通信的应用场景，例如：
    *   实时聊天室/即时通讯
    *   在线多人游戏
    *   股票、期货等金融行情的实时推送
    *   实时数据监控（如服务器状态、物联网设备数据）
    *   在线协作工具（如共享文档、白板）
    *   实时地理位置共享

与传统HTTP轮询/长轮询的比较：

*   HTTP短轮询：客户端定期向服务器发送请求查询是否有新数据。缺点是延迟高，服务器和网络资源浪费大。
*   HTTP长轮询（Comet）：客户端发送一个请求，服务器保持连接打开，直到有新数据或超时才响应。客户端收到响应后立即发起新的长轮询请求。缺点是仍然有连接建立和头部开销，服务器需要维护大量挂起连接。
*   WebSocket：一次握手后，连接保持，双方可随时推送数据，头部开销小，延迟低，效率高。
