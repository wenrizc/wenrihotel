## 1. 浏览器做的第一件事：解析 URL 与做缓存决策

输入 URL 后，浏览器会先做一些“本地决策”，尽可能避免网络请求：

- URL 解析：协议、主机、端口、路径、查询参数。
- HSTS：如果命中 HSTS，`HTTP` 会被强制升级为 `HTTPS`。
- Service Worker：若存在拦截逻辑，可能直接返回缓存响应或改写请求。
- 浏览器缓存：根据 Cache-Control、ETag、Last-Modified 判断是否可用或需协商缓存。

## 2. DNS 解析：把域名变成 IP

DNS 的目标是把域名解析为 IP（A/AAAA），典型链路：

1. 浏览器/系统 DNS 缓存命中则直接返回。
2. 未命中则向递归解析器（运营商/公司 DNS）发起查询。
3. 递归解析器向根域、顶级域、权威 DNS 逐级查询，返回结果并缓存（受 TTL 控制）。

工程上常见的“DNS 慢”原因包括：缓存未命中、解析器链路异常、跨网访问等。

## 3. 建立连接：TCP（必要时）+ TLS

### 3.1 TCP 三次握手

TCP 建连需要 1 个 RTT 的握手成本。若开启 Keep-Alive 或连接复用，后续请求可以复用已有连接，显著降低延迟。

### 3.2 TLS 握手（HTTPS）

HTTPS 需要 TLS 握手，包含：

- 证书链校验与域名校验
- 协商密码套件与会话密钥

会话复用（Session Resumption）可以减少握手 RTT 与计算开销。

## 4. 发送 HTTP 请求：协议版本会影响性能形态

请求发出后会携带：

- 方法、路径、查询参数
- Header（Host、Cookie、Accept-Encoding、User-Agent 等）

协议差异要点：

- HTTP/1.1：并发依赖多连接，存在队头阻塞。
- HTTP/2：单连接多路复用，头部压缩，提升并发效率。
- HTTP/3：基于 QUIC（UDP），改善弱网与握手成本（实现与部署相关）。

## 5. 服务端处理：从边缘到应用再到存储

典型链路（不一定全部存在）：

1. CDN：就近命中缓存，未命中回源。
2. 负载均衡：选择后端实例（四层/七层）。
3. 反向代理：Nginx 等进行路由、限流、压缩、静态资源处理。
4. 应用服务：鉴权、业务逻辑、调用缓存/数据库/搜索等。
5. 存储层：MySQL、Redis、消息队列等。

服务端的关键指标通常是：TTFB（首字节时间）与后端依赖耗时。

## 6. 接收响应：缓存与压缩相关的关键头

响应包含状态码、头部与 body，常见关键头：

- `Cache-Control`：缓存策略（max-age、no-cache、no-store）。
- `ETag` / `Last-Modified`：协商缓存。
- `Content-Encoding`：gzip/brotli 压缩。

## 7. 浏览器渲染：从字节到像素

渲染主流程可以按“构建 → 计算 → 绘制”理解：

1. 解析 HTML → DOM。
2. 解析 CSS → CSSOM。
3. DOM + CSSOM → Render Tree。
4. Layout（布局）计算几何信息。
5. Paint（绘制）生成绘制指令。
6. Composite（合成）在 GPU 上合成图层并显示。

关键规则：

- CSS 会阻塞渲染（没有 CSSOM 就无法正确绘制）。
- 同步 JS 可能阻塞 DOM 解析（除非 `defer/async`）。

## 8. 对应阶段的性能优化抓手（面试加分）

- DNS：`dns-prefetch`、就近递归解析器、合理 TTL。
- 建连：连接复用、TLS 会话复用、HTTP/2/3。
- 资源：CDN、压缩（gzip/brotli）、图片优化（WebP/AVIF）、`preload`。
- 渲染：减少阻塞脚本、拆分关键 CSS、避免频繁重排/重绘。

