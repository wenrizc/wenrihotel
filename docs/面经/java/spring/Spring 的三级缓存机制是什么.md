
#### Spring 三级缓存机制概述
- **定义**：
  - Spring 三级缓存机制是 Spring IoC 容器为解决 **Bean 循环依赖** 问题而设计的缓存结构，通过三个层次的缓存存储 Bean 的不同创建阶段，确保循环依赖场景下 Bean 的正确初始化。
- **目的**：
  - 支持单例 Bean 的循环依赖（如 A 依赖 B，B 依赖 A）。
  - 保证 Bean 初始化安全和一致性。
- **三级缓存**：
  1. **一级缓存**（singletonObjects）：存储已完全初始化的单例 Bean。
  2. **二级缓存**（earlySingletonObjects）：存储早期暴露的 Bean（已创建但未初始化）。
  3. **三级缓存**（singletonFactories）：存储 Bean 的工厂对象，用于生成早期 Bean。

#### 核心点
- 三级缓存通过提前暴露 Bean 引用，解决循环依赖，Spring AOP 的代理对象也依赖此机制。

---

### 1. 三级缓存详解
#### (1) 三级缓存定义
- **一级缓存**（`singletonObjects`）：
  - 类型：`ConcurrentHashMap<String, Object>`。
  - 存储：完整的单例 Bean（已完成属性注入和初始化）。
  - 作用：提供最终 Bean 实例，供客户端使用。
- **二级缓存**（`earlySingletonObjects`）：
  - 类型：`ConcurrentHashMap<String, Object>`。
  - 存储：早期暴露的 Bean（已实例化但未完成属性注入或初始化）。
  - 作用：解决循环依赖，临时持有半成品 Bean。
- **三级缓存**（`singletonFactories`）：
  - 类型：`ConcurrentHashMap<String, ObjectFactory<?>>`。
  - 存储：Bean 的工厂对象（`ObjectFactory`），可动态生成早期 Bean。
  - 作用：支持 AOP 代理等扩展，延迟创建代理对象。

#### (2) 源码位置
- 定义在 `DefaultSingletonBeanRegistry`：
```java
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);
private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);
```

---

### 2. 三级缓存工作原理
#### (1) Bean 创建流程
1. **实例化**：
   - Spring 调用构造方法创建 Bean（未注入属性）。
2. **放入三级缓存**：
   - 创建 `ObjectFactory`，放入 `singletonFactories`，暴露早期引用。
3. **属性注入**：
   - 填充依赖，若发现循环依赖，从缓存获取。
4. **初始化**：
   - 执行 `afterPropertiesSet`、初始化方法。
5. **缓存转移**：
   - 完成初始化后，移入一级缓存，清除二、三级缓存。

#### (2) 循环依赖解决
- **场景**：
  - A 依赖 B，B 依赖 A。
  - 类定义：
```java
@Component
public class A {
    @Autowired
    private B b;
}

@Component
public class B {
    @Autowired
    private A a;
}
```
- **流程**：
  1. **创建 A**：
     - 实例化 A（构造方法）。
     - 创建 `ObjectFactory`（生成 A 的引用），放入三级缓存（`singletonFactories`）。
     - 注入 B，发现依赖 B。
  2. **创建 B**：
     - 实例化 B。
     - 创建 `ObjectFactory`，放入三级缓存。
     - 注入 A，从三级缓存获取 A 的 `ObjectFactory`，生成早期 A，放入二级缓存（`earlySingletonObjects`），移除三级缓存。
     - B 完成注入和初始化，放入一级缓存（`singletonObjects`）。
  3. **继续 A**：
     - A 从一级缓存获取 B（已完成）。
     - A 完成注入和初始化，移入一级缓存，移除二、三级缓存。
- **结果**：
  - A 和 B 正确初始化，循环依赖解决。

#### (3) 图示
```
[A 创建] --> [singletonFactories: A=ObjectFactory]
  |           [B 创建] --> [singletonFactories: B=ObjectFactory]
  |                        [B 需 A] --> [earlySingletonObjects: A]
  |                        [B 完成] --> [singletonObjects: B]
[A 需 B] --> [singletonObjects: B]
[A 完成] --> [singletonObjects: A]
```

---

### 3. 三级缓存的作用
#### (1) 为什么需要三级缓存？
- **一级缓存**：
  - 存储最终 Bean，确保单例唯一。
  - 不足：无法提前暴露未初始化 Bean，循环依赖会失败。
- **二级缓存**：
  - 存储早期 Bean，解决循环依赖。
  - 不足：无法动态生成代理（如 AOP 增强）。
- **三级缓存**：
  - 存储 `ObjectFactory`，支持 AOP 代理创建。
  - 例：Spring AOP（您提问）需为 Bean 生成代理对象。

#### (2) 三级缓存与 AOP
- **场景**：
  - A 需要 AOP 代理（如 `@Transactional`）。
- **流程**：
  - 三级缓存的 `ObjectFactory` 可生成代理：
```java
singletonFactories.put(beanName, () -> getEarlyBeanReference(beanName, bean));
```
  - `getEarlyBeanReference` 调用 AOP 逻辑，生成代理。
- **效果**：
  - 循环依赖中注入的是代理对象，而非原始 Bean。

#### (3) 源码分析
- **添加三级缓存**：
```java
protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
    synchronized (this.singletonObjects) {
        if (!this.singletonObjects.containsKey(beanName)) {
            this.singletonFactories.put(beanName, singletonFactory);
            this.earlySingletonObjects.remove(beanName);
        }
    }
}
```
- **获取早期 Bean**：
```java
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    Object singletonObject = this.singletonObjects.get(beanName);
    if (singletonObject == null && allowEarlyReference) {
        singletonObject = this.earlySingletonObjects.get(beanName);
        if (singletonObject == null) {
            ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
            if (singletonFactory != null) {
                singletonObject = singletonFactory.getObject();
                this.earlySingletonObjects.put(beanName, singletonObject);
                this.singletonFactories.remove(beanName);
            }
        }
    }
    return singletonObject;
}
```

---

### 4. 循环依赖限制
#### (1) 支持场景
- **单例 Bean**：
  - 三级缓存仅支持单例（`scope="singleton"`）。
- **构造器无依赖**：
  - 构造器注入的循环依赖无法解决。
  - 例：
```java
@Component
public class A {
    public A(B b) {}
}
@Component
public class B {
    public B(A a) {}
}
```
  - 报错：`BeanCurrentlyInCreationException`。

#### (2) 不支持场景
- **原型 Bean**（`scope="prototype"`）：
  - 每次创建新实例，无缓存。
- **构造器注入**：
  - 构造时依赖未创建。
- **解决办法**：
  - 用 `@Lazy` 延迟注入。
  - 改用字段或 Setter 注入：
```java
@Component
public class A {
    @Autowired
    private B b;
}
```

---

### 5. 与您的问题关联
- **Spring AOP**：
  - 三级缓存的 `ObjectFactory` 支持 AOP 代理生成。
  - 例：`@Transactional` 的代理在循环依赖中正确注入。
- **事务传播**：
  - 事务依赖代理，三级缓存确保代理 Bean 可用。
- **代理模式**：
  - 三级缓存生成动态代理（JDK 或 CGLIB）。
- **单点登录**：
  - 认证服务单例化，循环依赖（如服务间引用）靠三级缓存。

---

### 6. 优缺点
#### 优点
- **解决循环依赖**：
  - 支持复杂单例依赖。
- **支持扩展**：
  - AOP、代理无缝集成。
- **高效**：
  - 缓存减少重复创建。

#### 缺点
- **复杂性**：
  - 三级逻辑调试困难。
- **限制**：
  - 不支持原型和构造器依赖。
- **内存**：
  - 缓存占用空间。

---

### 7. 常见问题与解决
- **问题**：循环依赖报错。
  - **原因**：构造器注入或非单例。
  - **解决**：
    - 改 Setter 注入。
    - 加 `@Lazy`：
```java
@Component
public class A {
    @Lazy
    @Autowired
    private B b;
}
```
- **问题**：AOP 代理失效。
  - **原因**：提前引用原始 Bean。
  - **解决**：确保三级缓存生效（默认单例）。

---

### 8. 延伸与面试角度
- **与 AOP**：
  - 三级缓存支持动态代理生成。
- **与事务**：
  - 事务代理依赖缓存。
- **实际应用**：
  - 电商：订单和用户服务循环依赖。
  - 微服务：单例组件管理。
- **面试点**：
  - 问“原理”时，提三级缓存流程。
  - 问“限制”时，提构造器问题。

---

### 9. 总结
Spring 三级缓存（`singletonObjects`、`earlySingletonObjects`、`singletonFactories`）通过提前暴露 Bean 引用解决单例循环依赖，支持 AOP 代理生成。`REQUIRED` 事务和代理模式依赖此机制。结合您的问题，缓存确保 SSO 服务单例一致。面试时，可提流程图、代码或 AOP 关联，展示理解深度。

---

如果您想深入某部分（如源码分析或循环依赖案例），请告诉我，我可以进一步优化！