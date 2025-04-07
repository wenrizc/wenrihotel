
#### 一、标准 IO 与 NIO 的基本区别

- **标准 IO（BIO）**：
    - 操作单位：字节流（Stream），每次读写一个字节。
    - 方式：按顺序逐字节处理。
- **NIO**：
    - 操作单位：块（Block），每次读写一个数据块（byte[]）。
    - 方式：将数据读入内存后以字节数组形式操作，可一次性处理多个字节。
- **本质差异**：
    - BIO 是流式读写，类似水流逐滴传输。
    - NIO 是块式读写，类似磁盘操作，效率更高。

#### 二、流与块的对比

- **流式 IO（BIO）**：
    - **特点**：每次处理一个字节，输入流产生数据，输出流消费数据。
    - **优点**：
        - 易于创建过滤器（如链式处理）。
        - 实现简单，逻辑优雅。
    - **缺点**：速度慢，逐字节操作效率低。
- **块式 IO（NIO）**：
    - **特点**：以块为单位处理数据。
    - **优点**：比流式处理快，适合大数据量操作。
    - **缺点**：缺少流式 IO 的优雅性和简单性。
- **集成性**：
    - java.io.* 包已基于 NIO 优化，部分类支持块读写（如 BufferedReader），提升了流式系统的性能。

#### 三、NIO 核心组件

##### 1. 通道（Channel）

- **定义**：模拟传统 IO 的流，但支持双向数据传输。
- **与流的区别**：
    - 流：单向（输入流 InputStream 或输出流 OutputStream）。
    - 通道：双向，可同时读写。
- **主要类型**：
    - FileChannel：文件读写。
    - DatagramChannel：UDP 网络数据读写。
    - SocketChannel：TCP 网络数据读写。
    - ServerSocketChannel：监听 TCP 连接，生成 SocketChannel。

##### 2. 缓冲区（Buffer）

- **定义**：通道读写数据的中间容器，本质是一个数组，提供结构化访问和读写状态跟踪。
- **工作原理**：
    - 写数据：先放入缓冲区，再写入通道。
    - 读数据：从通道读入缓冲区，再处理。
- **主要类型**：
    - ByteBuffer、CharBuffer、ShortBuffer、IntBuffer、LongBuffer、FloatBuffer、DoubleBuffer。

##### 3. 缓冲区状态变量

- **capacity**：缓冲区最大容量，固定不变。
- **position**：当前读写位置（已处理字节数）。
- **limit**：可读写字节的上限。
- **状态变化示例**：
    1. 新建缓冲区（容量 8）：position = 0, limit = capacity = 8。
    2. 读入 5 字节：position = 5, limit = 8。
    3. 调用 flip()（切换读模式）：limit = 5, position = 0。
    4. 写出 4 字节：position = 4, limit = 5。
    5. 调用 clear()（清空）：position = 0, limit = 8。

#### 四、文件 NIO 示例

- **功能**：快速复制文件。
- **代码**：
    `public static void fastCopy(String src, String dist) throws IOException { FileInputStream fin = new FileInputStream(src); FileChannel fcin = fin.getChannel(); FileOutputStream fout = new FileOutputStream(dist); FileChannel fcout = fout.getChannel(); ByteBuffer buffer = ByteBuffer.allocateDirect(1024); while (true) { int r = fcin.read(buffer); if (r == -1) break; // EOF buffer.flip(); fcout.write(buffer); buffer.clear(); } }`
    
- **流程**：
    1. 创建输入输出流和通道。
    2. 分配缓冲区（1024 字节）。
    3. 循环读入缓冲区，切换模式后写入目标文件。

#### 五、选择器（Selector）

- **定义**：实现 IO 多路复用（Reactor 模型），一个线程通过选择器监听多个通道事件。
- **非阻塞特性**：
    - 通道配置为非阻塞，若事件未就绪，继续轮询其他通道。
    - 减少线程切换开销，提升性能。
- **限制**：仅套接字通道（SocketChannel 等）支持非阻塞，FileChannel 不支持。

##### 操作步骤

1. **创建选择器**：
    `Selector selector = Selector.open();`
    
2. **注册通道**：
    `ServerSocketChannel ssChannel = ServerSocketChannel.open(); ssChannel.configureBlocking(false); ssChannel.register(selector, SelectionKey.OP_ACCEPT);`
    
    - 事件类型：OP_ACCEPT、OP_READ、OP_WRITE、OP_CONNECT。
3. **监听事件**：
    `int num = selector.select(); // 阻塞直到事件到达`
    
4. **处理事件**：
    `Set<SelectionKey> keys = selector.selectedKeys(); Iterator<SelectionKey> it = keys.iterator(); while (it.hasNext()) { SelectionKey key = it.next(); if (key.isAcceptable()) { /* 处理连接 */ } else if (key.isReadable()) { /* 处理读取 */ } it.remove(); }`
    
5. **事件循环**：
    - 放入 while (true) 持续监听。

#### 六、套接字 NIO 示例

- **服务器端**：
    `public class NIOServer { public static void main(String[] args) throws IOException { Selector selector = Selector.open(); ServerSocketChannel ssChannel = ServerSocketChannel.open(); ssChannel.configureBlocking(false); ssChannel.register(selector, SelectionKey.OP_ACCEPT); ServerSocket serverSocket = ssChannel.socket(); serverSocket.bind(new InetSocketAddress("127.0.0.1", 8888)); while (true) { selector.select(); Set<SelectionKey> keys = selector.selectedKeys(); Iterator<SelectionKey> it = keys.iterator(); while (it.hasNext()) { SelectionKey key = it.next(); if (key.isAcceptable()) { SocketChannel sChannel = ssChannel.accept(); sChannel.configureBlocking(false); sChannel.register(selector, SelectionKey.OP_READ); } else if (key.isReadable()) { SocketChannel sChannel = (SocketChannel) key.channel(); System.out.println(readDataFromSocketChannel(sChannel)); sChannel.close(); } it.remove(); } } } }`
    
- **客户端**：
    `public class NIOClient { public static void main(String[] args) throws IOException { Socket socket = new Socket("127.0.0.1", 8888); OutputStream out = socket.getOutputStream(); out.write("hello world".getBytes()); out.close(); } }`
    

#### 七、内存映射文件

- **定义**：将文件直接映射到内存，读写速度比流式或通道 IO 快。
- **特点**：
    - 修改缓冲区即修改文件，操作直接影响磁盘。
- **示例**：
    `MappedByteBuffer mbb = fc.map(FileChannel.MapMode.READ_WRITE, 0, 1024);`
    
    - 映射文件前 1024 字节到内存。

#### 八、NIO 与 BIO 对比

1. **阻塞性**：
    - BIO：阻塞式。
    - NIO：非阻塞式。
2. **数据处理**：
    - BIO：面向流，逐字节。
    - NIO：面向块，批量处理。

#### 九、总结

- **NIO 优势**：
    - 非阻塞特性支持高并发。
    - 块式处理提升效率。
- **核心组件**：
    - 通道（双向传输）、缓冲区（数据容器）、选择器（事件监听）。
- **应用场景**：
    - 文件操作、网络通信，尤其适合高并发服务器。