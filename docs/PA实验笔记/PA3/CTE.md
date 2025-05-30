
CTE (Context Extension) 是 AM (Abstract Machine) 中新增的一类 API，用于**抽象不同架构的上下文管理功能**，为操作系统提供统一的接口，类似于 IOE 对 I/O 操作的抽象。
- **架构相关的上下文管理：**  
  不同架构（如 x86, MIPS32, RISC-V32）使用不同的指令（例如 x86 的 `int`、MIPS32 的 `syscall`、RISC-V32 的 `ecall`）以及不同的机制来处理自陷（trap）和异常响应。同一架构下，上下文（例如寄存器状态）也存在差异。

- **操作系统需求：**  
  操作系统需要一种统一、架构无关的方式来处理上下文切换，获取必要的状态信息，并执行后续操作。CTE 的设计正是为了解决这一需求。

---

## CTE 的抽象目标

CTE 的目标是以架构无关的方式向操作系统提供两大关键信息：

1. **执行流切换的原因 (事件 - Event)：**  
   说明是什么原因导致从用户程序切换到操作系统的执行流，如：
   - 除零异常
   - 非法指令异常
   - 断点异常
   - 用户程序主动请求 (自陷 yield)

2. **程序上下文 (Context)：**  
   当发生执行流切换时，记录用户程序的状态信息，如：
   - 程序计数器 (PC)
   - 通用寄存器
   - 其他架构相关的状态信息

---

## 抽象实现

### 1. 事件抽象 (Event)

CTE 定义了统一的 `Event` 数据结构，用于描述执行流切换的原因。示例如下：

```c
typedef struct Event {
  enum {
    EVENT_NONE,
    EVENT_YIELD,
    EVENT_DIVIDE_ZERO,
    EVENT_ILLEGAL_INSTR,
    EVENT_BREAKPOINT,
    // ...其他事件类型
  } event;
  uintptr_t cause; // 事件补充信息，可用于记录错误码或异常原因
  uintptr_t ref;   // 补充参考信息，具体含义由事件类型决定
  const char *msg; // 文本描述信息
} Event;
```

- **event:** 采用枚举类型定义一系列架构无关的事件类型，如 `EVENT_YIELD` 表示用户程序主动调用 yield。
- **cause & ref:** 提供额外的事件相关信息。
- **msg:** 用于存储事件的描述信息。在部分实现中，可能仅需关注事件编号。

### 2. 上下文抽象 (Context)

CTE 定义了统一的 `Context` 结构体，表示程序在发生上下文切换时的状态。具体成员由各架构自行定义，但操作系统应通过 CTE 提供的接口访问，而不直接依赖具体实现。

```c
typedef struct Context {
  // 架构相关的上下文信息，由各架构自行定义
  // 例如：程序计数器、通用寄存器保存区等
} Context;
```

- **注意：**  
  为确保操作系统代码的可移植性，操作系统应避免直接访问 `Context` 结构体的具体成员，而是通过 CTE 提供的 API 进行间接访问。

---

## CTE 提供的 API

### 1. 初始化：`cte_init`

初始化 CTE 模块，并注册操作系统提供的事件处理回调函数。

```c
bool cte_init(Context* (*handler)(Event ev, Context *ctx));
```

- **handler:**  
  指向操作系统事件处理函数的函数指针。当事件发生时，CTE 调用该函数，传入 `Event` 和 `Context` 信息，由操作系统处理后续操作。

### 2. 触发上下文切换：`yield`

触发自陷操作，产生一个事件编号为 `EVENT_YIELD` 的事件，通常由用户程序主动调用。

```c
void yield(void);
```

- **内部实现：**  
  根据不同的 ISA，`yield()` 内部会使用特定的自陷指令（例如 x86 的 `int`、MIPS32 的 `syscall`、RISC-V32 的 `ecall`）来触发异常，从而实现上下文切换。

---

## 异常 vs. 事件

- **异常 (Exception)：**  
  是硬件层面的概念，由处理器在检测到错误（例如除零、非法指令）或自陷指令时自动触发。

- **事件 (Event)：**  
  是 AM 层面对异常的封装与抽象，提供了一个更高层次、架构无关的描述。  
  同一硬件异常可能对应多个事件（如 `EVENT_YIELD`、系统调用等），CTE 通过额外的信息（指令参数、寄存器值等）来区分不同事件。

---

## AM 中的 Undefined Behavior (UB)

在 AM 的运行时环境中，为了简化教学和实现，存在以下 UB 行为：
1. **浮点指令：**  
   AM 运行时环境不支持浮点数，执行浮点指令会导致未定义行为 (UB).

2. **栈溢出：**  
   AM 为程序提供的栈空间有限，栈溢出会导致 UB，系统不会检测出此类错误。

---

CTE 通过上述抽象实现，为操作系统提供了一个统一的上下文管理接口，有效屏蔽了具体架构的差异，从而使得操作系统能够跨平台运行，实现安全和稳定的执行流切换。