
#### GET 和 POST 区别
- **GET**：
  - 用于获取资源，参数在 URL 中。
- **POST**：
  - 用于提交数据，参数在请求体中。
- **主要区别**：
  1. **用途**：GET 获取，POST 提交。
  2. **参数位置**：GET 在 URL，POST 在 Body。
  3. **安全性**：GET 明文，POST 更安全。
  4. **数据量**：GET 受 URL 长度限制，POST 无限制。

#### 核心点
- 功能和传输方式不同。

---

### 1. 区别详解
#### (1) 用途
- **GET**：
  - 请求服务器返回资源（如页面、文件）。
  - 幂等：多次请求结果一致。
- **POST**：
  - 向服务器提交数据（如表单、文件）。
  - 非幂等：多次请求可能改变状态。
- **示例**：
  - GET：`GET /user?id=1` 获取用户。
  - POST：`POST /user` 创建用户。

#### (2) 参数位置
- **GET**：
  - 参数附加在 URL 查询字符串（`?key=value`）。
  - 示例：`http://example.com?name=Alice&age=25`。
- **POST**：
  - 参数放在 HTTP 请求体中。
  - 示例：
```
POST /submit HTTP/1.1
Host: example.com
Content-Type: application/json

{"name": "Alice", "age": 25}
```

#### (3) 安全性
- **GET**：
  - 参数明文可见，易被记录（浏览器历史、日志）。
  - 不适合敏感数据。
- **POST**：
  - 参数在请求体，相对隐蔽。
  - 可配合 HTTPS 加密。
- **注意**：
  - GET 并非不安全，只是暴露参数。

#### (4) 数据量
- **GET**：
  - URL 长度受限（浏览器和服务器限制，约 2KB-8KB）。
- **POST**：
  - 请求体无明确限制，可传大文件。
- **示例**：
  - GET：查询条件。
  - POST：文件上传。

---

### 2. 其他区别
- **缓存**：
  - GET：可被浏览器缓存（如 304）。
  - POST：通常不缓存。
- **编码**：
  - GET：URL 编码（`application/x-www-form-urlencoded`）。
  - POST：支持多种（如 `multipart/form-data`）。
- **后退/刷新**：
  - GET：无副作用。
  - POST：可能重复提交。

---

### 3. 对比表
| **特性**     | **GET**             | **POST**            |
|--------------|---------------------|---------------------|
| **用途**     | 获取资源           | 提交数据           |
| **参数位置** | URL 查询字符串     | 请求体             |
| **安全性**   | 明文，较低         | 隐蔽，较高         |
| **数据量**   | 有限（URL 长度）   | 无限制             |
| **幂等性**   | 是                 | 否                 |
| **缓存**     | 支持               | 不支持             |

---

### 4. 延伸与面试角度
- **与 REST**：
  - GET：查询（Read）。
  - POST：创建（Create）。
- **实际应用**：
  - GET：搜索框参数。
  - POST：登录表单。
- **误区**：
  - GET 不一定比 POST 快，取决于服务器。
- **面试点**：
  - 问“区别”时，提用途和安全性。
  - 问“场景”时，提 REST 或表单。

---

### 总结
GET 用于获取资源，参数在 URL，适合简单查询；POST 用于提交数据，参数在请求体，适合复杂操作。区别在用途、位置、安全性和数据量。面试时，可提幂等性或举例，展示理解深度。