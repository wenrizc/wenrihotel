
## 一、概述

将C程序从源代码转换为可执行程序需经过四个主要步骤：**编译**、**汇编**、**链接**和**加载**。这些步骤将高级语言代码逐步转换为机器语言，并在内存中准备好执行。某些系统可能合并部分步骤以提高效率，但逻辑上这四个阶段不可或缺。

**翻译层次**：

- **C源代码** → **汇编语言** → **目标文件（机器语言）** → **可执行文件** → **内存加载执行**。
- **文件后缀**：
    - UNIX：`.c`（C源文件）、`.s`（汇编文件）、`.o`（目标文件）、`.a`（静态库）、`.so`（动态库）、`a.out`（默认可执行文件）。
    - MS-DOS：`.c`、`.asm`、`.obj`、`.lib`、`.dll`、`.exe`。

## 二、编译器

**功能**：将C程序转换为汇编语言程序（符号形式），提高程序员效率。

- **优势**：
    - 高级语言代码量少，编程效率高。
    - 现代优化编译器生成接近汇编专家水平的代码，有时优于手写汇编。
- **背景**：
    - 1975年，受限于存储器容量和编译器效率，操作系统多用汇编语言编写。
    - 如今，DRAM容量增长约50万倍，编译器优化能力大幅提升，减轻程序大小限制。

**定义**：

- **汇编语言**：符号语言，可翻译为二进制机器语言，充当高级语言与硬件接口。

## 三、汇编器

**功能**：将汇编语言程序转换为目标文件，包含机器语言指令、数据及内存分配信息。

- **伪指令**：
    - 汇编器支持硬件未实现的指令变种（如`move`、`blt`），简化编程。
    - 示例：`move $t0, $t1`转换为`add $t0, $zero, $t1`。
    - MIPS伪指令利用寄存器`$at`，丰富指令集，降低编程复杂性。
- **符号表**：
    - 记录标号与内存地址的对应关系，用于处理分支和数据寻址。
- **目标文件结构**（UNIX）：
    - **目标文件头**：描述各部分大小和位置。
    - **代码段**：机器语言代码。
    - **静态数据段**：程序生命周期内分配的数据。
    - **重定位信息**：标记需在加载时更新的绝对地址。
    - **符号表**：未定义的外部引用。
    - **调试信息**：关联机器指令与C源代码，便于调试。
- **数字基数**：汇编器支持二进制、十进制、十六进制，十六进制更紧凑，便于位模式转换。

## 四、链接器

**功能**：将多个独立汇编的目标文件组合为可执行文件，解析未定义引用。

- **优点**：支持单独编译，修改某模块只需重新编译该模块，节省资源（相比重新编译整个程序）。
- **步骤**：
    1. 将代码和数据模块象征性放入内存。
    2. 确定指令和数据标签的地址。
    3. 修补内部和外部引用（分支、跳转、数据寻址）。
- **重定位**：
    - 汇编器无法预知模块间的相对地址，链接器根据MIPS内存分配（代码段：`0x00400000`，数据段：`0x10000000`）更新绝对地址。
- **可执行文件**：
    - 格式同目标文件，但无未解决引用。
    - 可包含符号表、调试信息或重定位信息（“剥离”版本不含）。
- **示例**：链接两个目标文件（过程A和B）：
    - **输入**：
        - 过程A：代码段`100`字节，数据段`20`字节，引用变量`x`和过程`B`。
        - 过程B：代码段`200`字节，数据段`30`字节，引用变量`y`和过程`A`。
    - **输出**（可执行文件）：
        - 代码段：`0x00400000`（A代码）+`0x00400100`（B代码）。
        - 数据段：`0x10000000`（x）+`0x10000020`（y）。
        - 更新指令：
            - `lw $a0, 0x8000($gp)`（`x`地址，`$gp=0x10008000`）。
            - `jal 0x00400100`（过程B）。
            - `sw $a1, 0x8020($gp)`（`y`地址）。
            - `jal 0x00400000`（过程A）。
    - **注意**：`jal`使用伪直接寻址，丢弃低2位，地址为26位左移2位+PC高4位。

## 五、加载器

**功能**：将可执行文件加载到内存，准备执行。

- **步骤**（UNIX）：
    1. 读取文件头，确定代码和数据段大小。
    2. 创建足够地址空间。
    3. 复制指令和数据到内存。
    4. 将主程序参数复制到栈顶。
    5. 初始化寄存器，设置栈指针`$sp`。
    6. 跳转到启动例程，调用`main`函数，结束后通过`exit`系统调用终止。
- **定义**：
    - **加载器**：将目标程序加载到内存的系统程序。

## 六、动态链接库（DLL）

**静态链接缺点**：

- 库代码嵌入可执行文件，更新库需重新链接。
- 加载整个库，即使只用部分例程，增加内存占用（如C标准库2.5MB）。

**动态链接库**：

- **定义**：程序运行时才链接和加载的库例程。
- **优点**：
    - 支持库更新而不需重新编译程序。
    - 仅加载实际调用的例程，节省内存。
- **实现**：
    - **初始版本**：加载器调用动态链接器，解析所有可能调用的例程引用。
    - **晚过程链接（lazy procedure linkage）**：
        - 仅在例程首次调用时链接。
        - 过程：
            1. 调用虚入口，执行间接跳转。
            2. 动态链接器识别例程，重映射地址，更新跳转目标。
            3. 后续调用直接跳转，无额外开销。
        - 依赖虚拟内存管理，避免复制例程。
- **应用**：Windows和UNIX默认使用DLL。

## 七、Java程序启动

**特点**：

- 目标：跨平台安全运行，牺牲部分性能。
- 翻译层次：**Java源代码** → **Java字节码** → **JVM解释执行**（或**JIT编译**）。
- **Java字节码**：为解释设计的指令集，接近Java语言，编译简单，无优化。
- **Java虚拟机（JVM）**：解释字节码的软件，支持跨平台运行。
- **即时编译器（JIT）**：
    - 运行时识别“热点”方法，编译为宿主机机器语言。
    - 缓存编译结果，提升后续执行速度。
    - 平衡解释和编译开销，缩小与C/C++性能差距。
- **优点**：
    - **可移植性**：JVM支持多种设备（如手机、浏览器）。
    - **简单性**：地址由编译器或JVM解析，无需单独汇编。
- **缺点**：解释执行性能较差（早期比C慢10倍），JIT编译缓解此问题。

## 八、C程序示例：排序程序

### 1. Swap过程（图2-24）

**C代码**：

```c
void swap(int v[], int k) {
    int temp;
    temp = v[k];
    v[k] = v[k+1];
    v[k+1] = temp;
}
```

**翻译步骤**：

- **寄存器分配**：
    - 参数：`v`（`$a0`）、`k`（`$a1`）。
    - 临时变量：`temp`（`$t0`）。
- **生成代码**：
    - 计算`v[k]`地址：`k*4`（字寻址）。
    - 加载和存储数据，实现交换。
- **MIPS汇编**：

```assembly
swap:
    sll $t1, $a1, 2        # $t1 = k * 4
    add $t1, $a0, $t1      # $t1 = v + (k * 4)
    lw $t0, 0($t1)         # temp = v[k]
    lw $t2, 4($t1)         # $t2 = v[k+1]
    sw $t2, 0($t1)         # v[k] = $t2
    sw $t0, 4($t1)         # v[k+1] = temp
    jr $ra                 # return
```

- **特点**：叶过程，无需保存寄存器，使用伪指令`move`（如`move $t0, $zero`）简化代码。

### 2. Sort过程

**C代码**（冒泡排序）：

```c
void sort(int v[], int n) {
    int i, j;
    for (i = 0; i < n; i += 1) {
        for (j = i - 1; j >= 0 && v[j] > v[j+1]; j -= 1) {
            swap(v, j);
        }
    }
}
```

**翻译步骤**：

- **寄存器分配**：
    - 参数：`v`（`$a0`）、`n`（`$a1`）。
    - 变量：`i`（`$s0`）、`j`（`$s1`）。
    - 保存：`$a0`（`$s2`）、`$a1`（`$s3`）以供`swap`调用。
- **生成代码**：
    - 外层循环：`i`从0到`n-1`。
    - 内层循环：`j`从`i-1`到0，比较并交换`v[j]`和`v[j+1]`。
    - 调用`swap`：传递参数`v`和`j`。
- **保存寄存器**：保存`$ra`、`$s0-$s3`（栈分配20字节）。
- **MIPS汇编**（图2-27）：

```assembly
sort:
    addi $sp, $sp, -20     # 分配栈空间
    sw $ra, 16($sp)        # 保存$ra
    sw $s3, 12($sp)        # 保存$s3
    sw $s2, 8($sp)         # 保存$s2
    sw $s1, 4($sp)         # 保存$s1
    sw $s0, 0($sp)         # 保存$s0
    move $s2, $a0          # 保存$a0到$s2
    move $s3, $a1          # 保存$a1到$s3
    move $s0, $zero        # i = 0
for1tst:
    slt $t0, $s0, $s3      # $t0 = 0 if i >= n
    beq $t0, $zero, exit1  # 若i >= n，退出
    addi $s1, $s0, -1      # j = i - 1
for2tst:
    slti $t0, $s1, 0       # $t0 = 1 if j < 0
    bne $t0, $zero, exit2  # 若j < 0，退出
    sll $t1, $s1, 2        # $t1 = j * 4
    add $t2, $s2, $t1      # $t2 = v + (j * 4)
    lw $t3, 0($t2)         # $t3 = v[j]
    lw $t4, 4($t2)         # $t4 = v[j+1]
    slt $t0, $t4, $t3      # $t0 = 0 if v[j+1] >= v[j]
    beq $t0, $zero, exit2  # 若v[j+1] >= v[j]，退出
    move $a0, $s2          # 传递v
    move $a1, $s1          # 传递j
    jal swap               # 调用swap
    addi $s1, $s1, -1      # j -= 1
    j for2tst              # 继续内层循环
exit2:
    addi $s0, $s0, 1       # i += 1
    j for1tst              # 继续外层循环
exit1:
    lw $s0, 0($sp)         # 恢复$s0
    lw $s1, 4($sp)         # 恢复$s1
    lw $s2, 8($sp)         # 恢复$s2
    lw $s3, 12($sp)        # 恢复$s3
    lw $ra, 16($sp)        # 恢复$ra
    addi $sp, $sp, 20      # 释放栈空间
    jr $ra                 # 返回
```

- **优化**：
    - **内联**：将`swap`代码直接嵌入，节省4条指令，但可能增加代码量，影响缓存命中率。
    - **性能比较**（图2-28、2-29）：
        - 无优化：CPI最佳，但指令数多。
        - O3优化：执行时间最短（100,000元素排序）。
        - Java解释执行：比无优化C慢8.3倍，JIT编译后接近C性能。
        - 快速排序：比冒泡排序快3个数量级。

## 九、数组与指针

**目的**：通过比较数组和指针实现的清零程序，理解C中指针的本质。

- **C代码**：

```c
void clear1(int array[], int size) {
    int i;
    for (i = 0; i < size; i += 1)
        array[i] = 0;
}
void clear2(int *array, int size) {
    int *p;
    for (p = &array[0]; p < &array[size]; p = p + 1)
        *p = 0;
}
```

- **寄存器分配**：
    - `clear1`：`array`（`$a0`）、`size`（`$a1`）、`i`（`$t0`）。
    - `clear2`：`array`（`$a0`）、`size`（`$a1`）、`p`（`$t0`）。

### 1. Clear1（数组版本）

**MIPS汇编**：

```assembly
clear1:
    move $t0, $zero        # i = 0
loop1:
    sll $t1, $t0, 2        # $t1 = i * 4
    add $t2, $a0, $t1      # $t2 = &array[i]
    sw $zero, 0($t2)       # array[i] = 0
    addi $t0, $t0, 1       # i += 1
    slt $t3, $t0, $a1      # $t3 = (i < size)
    bne $t3, $zero, loop1  # 若i < size，继续
```

- **特点**：每次迭代需计算数组地址（`i*4`），6条指令/循环。

### 2. Clear2（指针版本）

**MIPS汇编**（优化版）：

```assembly
clear2:
    move $t0, $a0          # p = &array[0]
    sll $t1, $a1, 2        # $t1 = size * 4
    add $t2, $a0, $t1      # $t2 = &array[size]
loop2:
    sw $zero, 0($t0)       # *p = 0
    addi $t0, $t0, 4       # p += 4
    slt $t3, $t0, $t2      # $t3 = (p < &array[size])
    bne $t3, $zero, loop2  # 若p < &array[size]，继续
```

- **特点**：
    - 指针直接递增，循环体4条指令。
    - 数组末地址计算移到循环外，减少开销。
- **比较**：
    - 数组版本：需每次计算地址，指令多。
    - 指针版本：更高效，现代编译器可为数组版本生成类似优化代码。
- **优化**：现代编译器通过**强度减少**（移位替代乘法）和**变量消除**（避免重复地址计算）使数组版本性能接近指针版本。

## 十、扩展与总结

### 1. 性能优化

- **编译器优化**：
    - 强度减少：用移位代替乘法。
    - 循环不变量外提：如`clear2`中数组末地址计算。
    - 内联：减少调用开销，但需权衡代码膨胀。
- **算法选择**：快速排序比冒泡排序快50-1000倍，算法对性能影响远超语言或优化。
- **动态链接**：晚过程链接降低内存占用，提升程序更新灵活性。

### 2. 硬件支持

- **MIPS特性**：
    - 伪指令（如`move`、`blt`）简化编程。
    - 字对齐：指令地址低2位为0，`jal`寻址范围达2^28字节。
    - 寄存器约定：临时寄存器无需保存，保存寄存器需栈管理。
- **内存管理**：代码段、数据段、堆、栈的明确划分支持高效加载和执行。

### 3. Java与C对比

- **执行模式**：
    - C：编译为机器语言，性能高。
    - Java：字节码解释执行，JIT编译接近C性能。
- **适用场景**：
    - C：高性能需求（如嵌入式系统）。
    - Java：跨平台应用（如Web、移动设备）。
- **性能差距**：JIT编译使Java性能接近C，快速排序放大算法优势。

### 4. 总结

- **翻译流程**：编译、汇编、链接、加载四个步骤将C程序转化为可执行代码。
- **关键技术**：
    - 伪指令和符号表简化汇编。
    - 链接器支持模块化开发，动态链接提升灵活性。
    - 加载器确保程序正确运行。
- **程序示例**：
    - 排序程序展示寄存器分配、栈管理和过程调用。
    - 数组与指针清零程序揭示编译器优化潜力。
- **未来趋势**：
    - 编译器优化持续缩小高级语言与汇编性能差距。
    - 动态链接和JIT编译推动跨平台和高性能并存。