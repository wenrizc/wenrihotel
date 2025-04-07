
#### 一、异步 IO 概述

- **与同步 IO 的区别**：
    - **同步 IO**（阻塞式、非阻塞式、多路复用）：应用程序主动询问操作系统事件状态，操作系统不主动通知（“不问不答”）。
    - **异步 IO**：采用“订阅-通知”模式，应用程序注册监听后继续执行，操作系统在事件发生并准备好数据后主动通知。
- **操作系统支持**：
    - **Windows**：提供真正的异步 IO 技术 **IOCP**（I/O 完成端口）。
    - **Linux**：无原生异步 IO，使用 epoll（多路复用 IO）模拟异步行为。

#### 二、Java 对 AIO 的支持

- **框架特点**：
    - **Windows**：基于 IOCP 实现（如 WindowsAsynchronousSocketChannelImpl）。
    - **Linux**：基于 epoll 模拟异步 IO。
- **核心类**：
    - 从 java.nio.channels.spi.AsynchronousChannelProvider 开始，适配不同操作系统。
    - **通道类型**：
        - AsynchronousServerSocketChannel：服务器监听通道。
        - AsynchronousSocketChannel：客户端套接字通道。
- **与 NIO 的关系**：
    - 共享 NetworkChannel 接口，但 AIO 无需 Selector，直接通过通道订阅事件。

#### 三、为何选择 Netty

- **Java NIO/AIO 局限**：
    - 无上层格式封装（如 JSON、Protocol Buffer）。
    - 缺少权限管理、数据读取等高级功能。
    - Linux 下 poll/epoll bug（Selector.select 不阻塞，CPU 100%）。
- **Netty 优势**：
    - 封装数据格式（责任链模式编解码）。
    - 提供权限、易维护性、高性能支持。
    - 处理 NIO bug（JDK 1.7 未完全修复）。
    - Netty 底层仍基于 NIO/AIO，但更易用。

#### 四、总结

- **异步 IO**：订阅-通知模式，效率高于同步 IO。
- **Java AIO**：
    - Windows 用 IOCP，Linux 用 epoll 模拟。
    - 无 Selector，通道直接订阅事件。
- **应用场景**：高并发网络通信，需搭配线程池优化。
- **Netty 推荐**：弥补 AIO 不足，提供完整解决方案。