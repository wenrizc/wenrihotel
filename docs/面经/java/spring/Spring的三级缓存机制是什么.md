## 1. 三级缓存机制在解决什么问题

Spring 的三级缓存是为了解决一个典型问题：**单例 Bean 的 setter 循环依赖**。

例如 A 依赖 B，B 又依赖 A。创建 A 时还没注入完，就需要把 A 的“早期引用”暴露出去，让 B 能先拿到 A，从而打破依赖闭环。

此外，三级缓存还要兼顾 AOP：如果 A 最终会被代理，B 注入进去的最好也是“同一个代理引用”，否则可能出现“注入的是原始对象，容器里拿到的是代理对象”的不一致。

## 2. 三级缓存分别是什么

三级缓存（以 `DefaultSingletonBeanRegistry` 的概念为主）：

- `singletonObjects`：一级缓存，存放**完全初始化完成**的单例对象。
- `earlySingletonObjects`：二级缓存，存放**早期暴露**的单例对象（可能是原始对象或代理）。
- `singletonFactories`：三级缓存，存放 `ObjectFactory`，用于按需创建早期引用（常用于创建早期代理）。

## 3. 关键流程（能讲出主干就够加分）

### 3.1 创建 A：先注册工厂，允许提前暴露

当容器开始创建单例 A 时，会先把一个 `ObjectFactory` 放入 `singletonFactories`，表示“如果别人需要 A 的早期引用，可以通过这个工厂拿到”。

### 3.2 创建 B：依赖注入时发现需要 A，触发早期引用

创建 B 的过程中要注入 A，于是会走 `getSingleton(beanName, allowEarlyReference=true)`：

1. 先查一级缓存 `singletonObjects`。
2. 若没有且允许早期引用，则查二级缓存 `earlySingletonObjects`。
3. 若还没有，则从三级缓存 `singletonFactories` 取出工厂，创建早期引用，并放入二级缓存。

### 3.3 A 初始化完成：放入一级缓存

A 完成属性填充与初始化后，最终成品对象放入 `singletonObjects`，并清理二级/三级缓存中的临时数据。

## 4. 为什么需要第三级：为早期代理留一个钩子

如果只有二级缓存，只能提前暴露“原始对象”。但当 A 需要被 AOP 代理时：

- B 注入原始对象会绕过代理，导致事务、切面等增强失效或行为不一致。
- 三级缓存的 `ObjectFactory` 可以在“提前暴露”阶段就调用 `getEarlyBeanReference`，把早期代理暴露出去，保证引用一致。

## 5. 适用范围与限制

- 仅解决 **singleton + setter/属性注入** 的循环依赖。
- 构造器循环依赖无法解决（因为实例化阶段就需要依赖）。
- prototype Bean 循环依赖一般也无法通过该机制兜住。
