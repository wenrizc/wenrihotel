
### 注解的作用和区别

**@Component**:
- 是Spring组件模型的基础注解，用于标识一个类作为组件被Spring容器自动扫描和管理
- 是其他三个注解的"父注解"，它们都被@Component元注解标记
- 通常用于通用组件，当不确定组件归属于哪一层时使用

**@Controller**:
- 特化的@Component，用于标识表现层组件
- 主要用于处理用户请求，返回响应
- 在minispring中，被此注解标记的类会被`DispatcherServlet`特殊处理，扫描其中的@GetMapping和@PostMapping方法

**@Service**:
- 特化的@Component，用于标识业务逻辑层组件
- 主要包含业务处理逻辑
- 从功能上与@Component相同，但提供了更明确的语义，表示这是一个服务类

**@Repository**:
- 特化的@Component，用于标识数据访问层组件
- 通常用于DAO(数据访问对象)，处理数据库操作
- 在某些框架实现中，能将数据库异常转换为Spring的统一异常体系
