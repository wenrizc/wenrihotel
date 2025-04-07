## 1. 运行时环境的需求

- **加载与执行**：程序需要被加载到正确的内存地址，并由程序计数器 (PC) 指向第一条指令开始执行。
- **输入/输出支持**：复杂程序（如超级玛丽）需要与用户交互，这要求运行时环境提供输入输出的能力。
- **动态链接与上下文管理**：更复杂的应用程序需要模块化支持，如动态链接库的加载、上下文切换等。
- **程序退出机制**：程序在结束时必须有合适的退出方法，例如通过特殊指令触发系统中断来结束程序.

## 2. AM 项目的核心思想

为了满足各种程序对计算机功能的不同需求，我们把所有这些需求抽象成一组统一的 API，这组 API 被称为抽象计算机 (AM)。各个底层平台（或架构）只需根据自己的特性实现这组 API，程序则只调用统一的接口而无须关心底层细节. 这样就实现了程序与计算机硬件的解耦.

AM 的实现模块如下：

- **TRM (Turing Machine)**  
  为程序提供基本计算能力，是最原始的运行时基础.

- **IOE (I/O Extension)**  
  为程序提供输入输出功能，满足用户交互的需求.

- **CTE (Context Extension)**  
  管理程序运行时的上下文，支持多任务和上下文切换.

- **VME (Virtual Memory Extension)**  
  提供虚拟内存管理，支持更复杂的内存分配与地址空间隔离.

- **MPE (Multi-Processor Extension)**  
  为多处理器系统提供并行通信与协作.

## 3. 裸机与抽象计算机的关系

- 在 **NEMU** 中实现了 **硬件功能**：包括 CPU 指令、内存管理等基础设施.
- 在 **AM** 中提供了 **运行时环境**：封装了上述抽象 API，屏蔽不同架构的差异，使得所有程序调用统一的接口.
- 在 **APP 层** 运行应用程序：程序不再直接依赖硬件，而是依赖于由 AM 提供的运行时环境.

这种设计模型使得：
- **程序** 只需依赖统一的接口（例如调用 `halt()` 来结束程序），而不必关心运行在哪种架构上.
- **架构** 只需要实现各自的抽象函数，就能支撑多种程序的运行.

## 4. 总结

AM (Abstract Machine) 通过将运行时环境抽象为统一的 API 模块，极大地解耦了程序与硬件之间的联系. 其核心思想体现在：

- **模块化设计**：不同功能分层，TRM 提供基本计算，IOE、CTE、VME 和 MPE 分别扩展至输入输出、上下文管理、虚拟内存与多处理器支持.
- **平台无关性**：程序仅调用抽象 API，无需关心底层实现，便于跨平台移植和维护.
- **清晰的界限划分**：NEMU 负责硬件级实现，AM 负责运行时环境，为程序提供必要的支持，二者共同构成了一个完整的裸机运行平台.

(在NEMU中)实现硬件功能 -> (在AM中)提供运行时环境 -> (在APP层)运行程序

## AM包含的主要组件

### 1. TRM（图灵机）

基本计算功能抽象：

```c
extern Area heap;         // 堆内存区域
void putch(char ch);      // 字符输出
void halt(int code);      // 停止执行
```

### 2. IOE（输入/输出设备）

设备访问的统一接口：

```c
bool ioe_init(void);              // 初始化IO设备
void ioe_read(int reg, void *buf);    // 读取设备寄存器
void ioe_write(int reg, void *buf);   // 写入设备寄存器
```

支持的设备包括：

- **TIMER**：时钟设备（RTC实时时钟和uptime计时）
- **GPU**：图形显示（帧缓冲绘制）
- **INPUT**：输入设备（键盘）
- **AUDIO**：音频设备
- **DISK**：磁盘设备
- **UART**：串口通信
- **NET**：网络设备

### 3. CTE（上下文交换）

处理中断和上下文切换：

```c
bool cte_init(Context *(*handler)(Event ev, Context *ctx));
void yield(void);         // 主动让出CPU
bool ienabled(void);      // 查询中断是否启用
void iset(bool enable);   // 设置中断状态
```

### 4. VME（虚拟内存）

虚拟内存管理：

```c
bool vme_init(void *(*pgalloc)(int), void (*pgfree)(void *));
void protect(AddrSpace *as);                // 保护地址空间
void map(AddrSpace *as, void *va, void *pa, int prot);  // 建立映射
```

### 5. MPE（多处理器）

多处理器支持：

```c
bool mpe_init(void (*entry)());      // 初始化多处理器环境
int cpu_count(void);                 // 获取CPU总数
int cpu_current(void);               // 获取当前CPU ID
int atomic_xchg(int *addr, int newval);  // 原子交换操作
``` 