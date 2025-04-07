
#### 一、现实场景：餐厅点餐类比

- **场景描述**：餐厅同时接待 100 位客人，需优化点餐流程以平衡服务效率与成本。
- **角色对应**：
    - **客人**：客户端请求。
    - **点餐内容**：客户端发送的数据。
    - **老板**：操作系统。
    - **人力成本**：系统资源。
    - **菜单**：文件描述符（受限，Linux 下用 ulimit -n 查看）。
    - **服务员**：内核 IO 线程。
    - **厨师**：应用线程。
    - **餐单传递方式**：IO 模型。

##### 三种方法对比

1. **方法 A：阻塞式/非阻塞式同步 IO**
    - **流程**：1 名服务员持 1 本菜单，逐个服务客人，等待点餐完成再服务下一位。
    - **特点**：简单但效率极低，后续客人等待超时流失。
    - **对应**：单线程阻塞 IO，accept() 和 read() 顺序处理请求。
2. **方法 B：多线程同步 IO**
    - **流程**：雇佣 99 名服务员，每人持 1 本菜单，分别服务 1 位客人，厨师也增至 100 名。
    - **特点**：VIP 服务，效率高，但人力成本极高。
    - **对应**：为每个客户端分配线程，资源消耗大，线程数受限。
3. **方法 C：多路复用 IO**
    - **流程**：客人自取菜单，想好后再呼叫服务员，服务员记录多份菜单后批量交给厨师。
    - **特点**：成本低，客人稍有等待但可接受，服务员效率最大化。
    - **对应**：NIO 的 Reactor 模型，单线程监听多通道，批量处理事件。

- **选择依据**：视并发量（到店情况）灵活配置，低并发用方法 A，高并发用方法 C。

#### 二、多路复用 IO 实现

- **主流技术**：select、poll、epoll、kqueue。
- **特性比较**：
    
    |IO 模型|相对性能|关键思路|操作系统|Java 支持情况|
    |---|---|---|---|---|
    |select|较高|Reactor|Windows/Linux|支持，Linux 2.4 前默认，Windows 同步 IO 主流|
    |poll|较高|Reactor|Linux|Java NIO 支持，Linux 2.6 前默认|
    |epoll|高|Reactor/Proactor|Linux|Linux 2.6+ 支持，模拟异步 IO|
    |kqueue|高|Proactor|Linux|Java 当前不支持|
    
- **适用场景**：高并发（1ms 内 ≥ 1000 请求），低并发下优势不明显，且实现较复杂。

#### 三、Reactor 与 Proactor 模型

##### 1. 传统 IO 模型

- **特点**：
    - 每个客户端连接分配 1 个线程，线程全程处理读、解码、计算、编码、写。
    - 吞吐量与线程数线性相关。
- **问题**：
    - 线程数受限，无法无限增长。
    - 线程既处理 IO 又计算，效率低。
    - 阻塞式 IO 在网络状况差时降低线程利用率。

##### 2. Reactor 事件驱动模型

- **改进**：基于 Java NIO 非阻塞 API，以事件驱动处理网络事件。
- **角色**：
    - **Reactor**：事件分发器。
    - **Acceptor**：接收客户端连接。
    - **Handler**：处理具体事件。
- **优点**：
    - 将连接、读写、计算拆分，提升效率。
    - 非阻塞，线程可处理其他任务。

##### 3. Reactor 改进：业务与 IO 分离

- **问题**：单线程 Reactor 网络读写和业务处理仍混杂，高并发下瓶颈明显。
- **改进**：
    - 单线程处理连接和读写。
    - 线程池处理业务（编解码 + 计算）。
- **效果**：吞吐量提升，但单线程读写仍有限制。

##### 4. Reactor 并发读写

- **改进**：
    - **mainReactor**：单线程接收连接。
    - **subReactor**：线程池处理读写。
    - **业务线程池**：处理计算。
- **效果**：读写能力随线程数增强，可支持百万连接。

##### 5. Reactor 示例代码

- **Reactor**：监听并分发连接。
    
    `public class Reactor implements Runnable { private final Selector selector; private final ServerSocketChannel serverSocket; public Reactor(int port) throws IOException { serverSocket = ServerSocketChannel.open(); serverSocket.configureBlocking(false); selector = Selector.open(); serverSocket.register(selector, SelectionKey.OP_ACCEPT).attach(new Acceptor(serverSocket)); serverSocket.bind(new InetSocketAddress(port)); } public void run() { while (!Thread.interrupted()) { selector.select(); Set<SelectionKey> keys = selector.selectedKeys(); Iterator<SelectionKey> it = keys.iterator(); while (it.hasNext()) { dispatch(it.next()); it.remove(); } selector.selectNow(); } } private void dispatch(SelectionKey key) throws IOException { ((Runnable) key.attachment()).run(); } }`
    
- **Acceptor**：接收连接，交由线程池。
    
    `public class Acceptor implements Runnable { private final ExecutorService executor = Executors.newFixedThreadPool(20); private final ServerSocketChannel serverSocket; public Acceptor(ServerSocketChannel serverSocket) { this.serverSocket = serverSocket; } public void run() { SocketChannel channel = serverSocket.accept(); if (channel != null) executor.execute(new Handler(channel)); } }`
    
- **Handler**：处理读写和业务。
    
    `public class Handler implements Runnable { private final SocketChannel channel; private SelectionKey key; private ByteBuffer input = ByteBuffer.allocate(1024); private ByteBuffer output = ByteBuffer.allocate(1024); public Handler(SocketChannel channel) throws IOException { this.channel = channel; channel.configureBlocking(false); Selector selector = Selector.open(); key = channel.register(selector, SelectionKey.OP_READ); } public void run() { while (selector.isOpen() && channel.isOpen()) { Set<SelectionKey> keys = select(); Iterator<SelectionKey> it = keys.iterator(); while (it.hasNext()) { SelectionKey key = it.next(); it.remove(); if (key.isReadable()) read(key); else if (key.isWritable()) write(key); } } } private void read(SelectionKey key) throws IOException { channel.read(input); if (input.position() > 0) { input.flip(); process(); input.clear(); key.interestOps(SelectionKey.OP_WRITE); } } private void write(SelectionKey key) throws IOException { output.flip(); if (channel.isOpen()) { channel.write(output); channel.close(); output.clear(); } } private void process() { byte[] bytes = new byte[input.remaining()]; input.get(bytes); String message = new String(bytes); System.out.println("Received: " + message); output.put("Hello client".getBytes()); } }`
    

#### 四、Java NIO 支持

##### 1. Channel（通道）

- **定义**：应用程序与操作系统交互的桥梁，拥有独立文件描述符，支持双向读写。
- **类型**：
    - ServerSocketChannel：服务器监听，支持 TCP/UDP。
    - SocketChannel：TCP 连接通道。
    - DatagramChannel：UDP 数据通道。

##### 2. Buffer（缓冲区）

- **作用**：为通道提供数据缓存，提升读写速度。
- **模式**：
    - 写模式：可读写（小心脏读）。
    - 读模式：只读。
- **属性**：
    - position：当前操作位置。
    - limit：操作上限。
    - capacity：最大容量。

##### 3. Selector（选择器）

- **作用**：
    - 事件订阅与通道管理。
    - 轮询代理，代替应用直接询问操作系统。
    - 适配不同操作系统实现（如 WindowsSelectorImpl）。
- **管理容器**：存储注册的通道（如 SelectionKeyImpl[]）。

##### 4. Java NIO 设计

- **核心**：通过 SelectorProvider 抽象类适配不同操作系统。
- **方法**：
    - openSelector()：创建选择器。
    - openServerSocketChannel()：创建服务器通道。
- **Netty 示例**：
    
    `private static ServerSocketChannel newSocket(SelectorProvider provider) { return provider.openServerSocketChannel(); }`

##### 5. Java NIO 示例

- **简单版（SocketServer1）**：
    - 单缓冲区处理完整消息。
- **优化版（SocketServer2）**：
    - 小容量缓冲区 + ConcurrentHashMap 存储分段消息。
    - 处理流程：
        1. 接收连接，注册 OP_READ。
        2. 分段读取，拼接完整消息。
        3. 遇 “over” 结束，回发结果。

#### 五、多路复用 IO 优缺点

- **优点**：
    - 无需多线程处理 IO，节省内核和应用资源。
    - 单端口支持多协议（TCP/UDP）。
    - 操作系统优化，高效处理多客户端事件。
- **缺点**：
    - 仍属同步 IO，上层需主动询问事件。

#### 六、总结

- **现实启发**：方法 C（多路复用）平衡效率与成本，适合高并发。
- **技术实现**：epoll 等提升性能，Reactor 模型优化分工。
- **Java 支持**：NIO 提供灵活框架，需根据业务选择使用。