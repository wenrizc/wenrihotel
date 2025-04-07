# 多协议下载器系统设计文档

## 1. 系统概述

多协议下载器是一个面向多种网络协议的下载管理系统，支持HTTP、FTP、BitTorrent、磁力链接和HLS等多种协议，同时提供单文件、多文件和种子文件等多种下载形式。系统具有高度可扩展性和可配置性。

## 2. 整体架构

### 2.1 分层架构

系统采用分层架构，自上而下分为：

1. **表现层**：用户界面层，提供GUI和CLI接口
2. **应用层**：任务管理和调度层
3. **服务层**：下载器实现和协议处理层
4. **基础设施层**：网络通信、文件操作和配置管理

### 2.2 核心模块关系图

```
                       +----------------+
                       |     GUI/CLI    |
                       +-------+--------+
                               |
                       +-------v--------+
                       |  TaskManager   |
                       +-------+--------+
                               |
                       +-------v--------+
                       |DownloadScheduler|
                       +-------+--------+
                               |
           +------------------+v+------------------+
           |                  |                    |
    +------v------+    +------v------+     +------v------+
    |MonoDownloader|    |MultiDownloader|   |TorrentSession|
    +------+------+    +------+------+     +------+------+
           |                  |                    |
    +------v------+    +------v------+     +------v------+
    | HttpProtocol|    | FtpProtocol |     |TorrentProtocol|
    |  HlsProtocol|    |             |     |MagnetProtocol |
    +-------------+    +-------------+     +-------------+
```

## 3. 模块详细设计

### 3.1 Protocol模块

负责URL解析和协议处理，是整个系统的基础。

#### 3.1.1 核心接口

```java
public interface IProtocol {
    boolean parse(String url);
    String getType();
    boolean validateUrl(String url);
    ResourceMetadata getMetadata();
}
```

#### 3.1.2 协议实现

- **HttpProtocol**: HTTP/HTTPS协议实现
- **FtpProtocol**: FTP协议实现
- **TorrentProtocol**: BitTorrent种子文件解析
- **MagnetProtocol**: 磁力链接解析与元数据获取
- **HlsProtocol**: HTTP Live Streaming流媒体协议

### 3.2 Downloader模块

提供实际的下载功能，与不同协议对接。

#### 3.2.1 核心接口

```java
public interface IDownloader {
    boolean start();
    boolean pause();
    boolean resume();
    boolean cancel();
    DownloadStatus getStatus();
    double getProgress();
    long getSpeed();
    void setSpeedLimit(long limit);
    long getDownloadedBytes();
    long getTotalBytes();
}
```

#### 3.2.2 下载器实现

- **MonofileDownloader**: 单文件下载实现
- **MultifileDownloader**: 多文件下载实现
- **TorrentSessionDownloader**: BitTorrent下载实现
  - 负责管理种子会话、Peer连接
  - 处理文件分片和拼接

### 3.3 TaskManagement模块

管理下载任务的创建、调度和状态跟踪。

#### 3.3.1 主要组件

- **TaskManager**: 
  - 任务创建与管理
  - 任务状态监控
  - 事件分发

- **DownloadScheduler**: 
  - 并发任务调度
  - 资源分配管理
  - 队列管理

#### 3.3.2 任务状态模型

```
[创建] → [等待] → [连接中] → [下载中] → [已完成]
                    ↓           ↓
                   [失败]      [暂停]
                    ↓           ↓
                   [重试]      [恢复]
```

### 3.4 Context模块

处理任务上下文和会话信息，连接任务管理和下载器。

#### 3.4.1 关键组件

- **TaskContext**: 
  - 存储任务元数据和状态
  - 任务标识和生命周期管理

- **TaskSession**: 
  - 关联下载器和协议处理器
  - 提供会话持久化能力

### 3.5 Config模块

系统和任务级别的配置管理。

#### 3.5.1 配置层次

- **IConfig**: 基础配置接口
- **DownloaderConfig**: 全局下载器配置
- **NetworkConfig**: 网络相关配置
- **TaskConfig**: 任务级配置

### 3.6 Net模块

网络通信和协议实现的底层支持。

#### 3.6.1 BitTorrent实现

- **TrackerSession**: 处理与Tracker服务器的通信
- **PeerSession**: 管理与其他Peer的连接
- **TorrentWindow**: 种子文件预览和选择

### 3.7 Utils模块

提供通用工具函数。

- **NetworkUtils**: 网络操作助手
- **FileUtils**: 文件系统操作

### 3.8 Common模块

公共数据结构和模型定义。

- **FileInfo/TorrentFileInfo**: 文件信息模型
- **ResourceMetadata**: 资源元数据
- **DownloadStatus**: 下载状态枚举

## 4. 数据流设计

### 4.1 下载任务创建流程

```
1. 用户提供URL → 
2. TaskManager调用协议解析器验证URL → 
3. 协议解析器获取资源元数据 → 
4. TaskManager创建TaskContext → 
5. 根据协议类型选择合适的下载器 → 
6. 生成TaskSession并提交给DownloadScheduler
```

### 4.2 任务执行流程

```
1. DownloadScheduler根据优先级和资源限制启动任务 →
2. 下载器通过协议处理器获取数据 →
3. 数据写入文件系统 → 
4. 任务状态更新 → 
5. 完成后触发事件通知
```

### 4.3 BitTorrent下载流程

```
1. 解析种子文件/磁力链接 →
2. 连接Tracker获取Peer列表 →
3. 建立与Peer的连接 →
4. 交换位图信息 →
5. 请求和下载数据片段 →
6. 组装文件 →
7. 可选：做种分享
```

## 5. 接口设计

### 5.1 任务管理接口

```java
public interface TaskManager {
    String addTask(String url, String savePath);
    String addTask(String url, String savePath, TaskConfig config);
    boolean startTask(String taskId);
    boolean pauseTask(String taskId);
    boolean resumeTask(String taskId);
    boolean removeTask(String taskId, boolean deleteFiles);
    TaskSession getTask(String taskId);
    List<TaskSession> getAllTasks();
    List<TaskSession> getActiveTasks();
    List<TaskSession> getCompletedTasks();
    void addTaskManagerListener(TaskManagerListener listener);
    void removeTaskManagerListener(TaskManagerListener listener);
}
```

### 5.2 下载调度接口

```java
public interface DownloadScheduler {
    void start();
    void stop();
    void pauseAll();
    void resumeAll();
    int getRunningTaskCount();
    int getWaitingTaskCount();
    void setMaxConcurrentTasks(int count);
    void scheduleTask(TaskSession task);
    void unscheduleTask(String taskId);
}
```

### 5.3 事件监听接口

```java
public interface TaskManagerListener {
    void onTaskAdded(String taskId);
    void onTaskStarted(String taskId);
    void onTaskPaused(String taskId);
    void onTaskResumed(String taskId);
    void onTaskCompleted(String taskId);
    void onTaskFailed(String taskId, String error);
    void onTaskRemoved(String taskId);
}

public interface TaskStatusChangeListener {
    void onStatusChanged(String taskId, DownloadStatus oldStatus, DownloadStatus newStatus);
    void onProgressChanged(String taskId, double progress);
    void onSpeedChanged(String taskId, long speed);
}
```

## 6. 扩展机制设计

### 6.1 协议扩展点

通过实现IProtocol接口添加新协议支持：

```java
public interface IProtocolFactory {
    boolean canHandle(String url);
    IProtocol createProtocol();
    String getProtocolName();
}
```

### 6.2 下载器扩展点

通过实现IDownloader接口添加新的下载器类型：

```java
public interface IDownloaderFactory {
    boolean canHandle(String protocolType);
    IDownloader createDownloader(TaskContext context);
    String getDownloaderName();
}
```

### 6.3 插件系统设计

```java
public interface Plugin {
    String getName();
    String getVersion();
    void initialize(PluginContext context);
    void shutdown();
    List<IProtocolFactory> getProtocolFactories();
    List<IDownloaderFactory> getDownloaderFactories();
}

public class PluginManager {
    boolean registerPlugin(Plugin plugin);
    boolean unregisterPlugin(String pluginName);
    Plugin getPlugin(String name);
    List<Plugin> getAllPlugins();
}
```

## 7. 异常处理机制

### 7.1 异常层次结构

```
DownloaderException
  ├── ProtocolException
  │     ├── HttpProtocolException
  │     ├── FtpProtocolException
  │     └── TorrentProtocolException
  ├── DownloaderException
  │     ├── ConnectionException
  │     ├── FileSystemException
  │     └── ResourceNotFoundException
  └── ConfigurationException
```

### 7.2 重试机制

```java
public interface RetryPolicy {
    boolean shouldRetry(int attemptCount, Exception e);
    long getRetryDelay(int attemptCount);
    int getMaxAttempts();
}
```

## 8. 持久化设计

### 8.1 任务持久化

```java
public interface TaskPersistence {
    boolean saveTask(TaskContext context);
    TaskContext loadTask(String taskId);
    boolean deleteTask(String taskId);
    List<String> getAllTaskIds();
}
```

### 8.2 配置持久化

```java
public interface ConfigPersistence {
    boolean saveConfig(String key, Map<String, String> configData);
    Map<String, String> loadConfig(String key);
    boolean deleteConfig(String key);
}
```

## 9. 并发控制

### 9.1 线程池设计

```java
public interface ThreadPoolManager {
    ExecutorService getDownloadThreadPool();
    ExecutorService getIoThreadPool();
    ScheduledExecutorService getSchedulerThreadPool();
    void shutdown();
}
```

### 9.2 资源控制

```java
public interface ResourceManager {
    void setGlobalDownloadLimit(long bytesPerSecond);
    void setGlobalUploadLimit(long bytesPerSecond);
    void setTaskDownloadLimit(String taskId, long bytesPerSecond);
    void suspendLowPriorityTasks();
    void resumeLowPriorityTasks();
}
```

## 10. 安全机制设计

### 10.1 下载资源验证

```java
public interface SecurityManager {
    boolean validateDownloadSource(String url);
    boolean scanFileForThreats(String filePath);
    List<SecurityPolicy> getSecurityPolicies();
    void addSecurityPolicy(SecurityPolicy policy);
}
```

## 11. 实现建议

1. **协议实现**:
   - 使用Java NIO或Netty实现高效的网络通信
   - HTTP协议考虑使用HttpClient或OkHttp库
   - BitTorrent协议实现可参考开源项目如libtorrent

2. **下载器实现**:
   - 支持断点续传
   - 数据完整性校验
   - 并行分片下载

3. **持久化实现**:
   - 使用SQLite或H2数据库存储任务信息
   - 使用JSON格式存储配置信息

4. **多线程处理**:
   - 使用Java的ExecutorService管理线程池
   - 使用CompletableFuture处理异步操作

5. **UI实现**:
   - 桌面端考虑JavaFX或Swing
   - Web界面可考虑Spring Boot + RESTful API

## 12. 待优化事项

1. **性能优化**:
   - HTTP请求池化
   - 文件系统操作优化
   - BitTorrent算法优化

2. **可用性提升**:
   - 更友好的错误处理和恢复机制
   - 实时下载速度监控和调整
   - 下载队列优先级动态调整

3. **扩展性增强**:
   - 更完善的插件系统
   - 支持自定义协议处理器
   - 支持云存储集成

4. **测试覆盖**:
   - 单元测试框架
   - 集成测试方案
   - 性能测试用例

## 13. 项目规划

| 阶段 | 重点任务 | 时间估计 |
|------|----------|----------|
| 阶段一 | 核心框架实现、基础协议支持 | 2个月 |
| 阶段二 | BitTorrent完整支持、多文件下载 | 1.5个月 |
| 阶段三 | 高级功能、UI完善、性能优化 | 1.5个月 |
| 阶段四 | 测试、文档、发布准备 | 1个月 |

## 14. 总结

该多协议下载器系统采用模块化设计，通过清晰的接口定义和扩展点设计，实现了高内聚、低耦合的架构。系统支持多种协议、多种下载形式，并提供了灵活的配置和扩展机制，为用户提供高效、可靠的文件下载体验。