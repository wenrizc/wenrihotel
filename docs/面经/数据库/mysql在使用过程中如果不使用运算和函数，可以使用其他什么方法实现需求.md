
### 在应用程序中处理逻辑（最常见、最灵活的方法）

这是将计算和函数逻辑移出数据库的首选方案。你的后端代码（如Java, Python, PHP, Go, Node.js等）将承担所有的数据处理任务。

流程如下：
1.  你的应用程序向MySQL发送一个非常简单的查询，只负责筛选和获取最原始、未经处理的数据。
2.  MySQL返回一个原始数据集（例如，一个用户列表，每个用户有 `first_name` 和 `last_name` 字段）。
3.  应用程序在内存中遍历这个数据集，并执行所有必要的计算和逻辑。

#### 示例场景：

*   需求：计算每个订单的总价 (`price * quantity`)。
    *   不好的做法 (在MySQL中计算): `SELECT order_id, quantity * price AS total FROM order_items;`
    *   替代做法 (在应用中处理):
        1.  SQL查询: `SELECT order_id, quantity, price FROM order_items;`
        2.  应用层代码 (以Python为例):
            ```python
            results = db.query("SELECT order_id, quantity, price FROM order_items;")
            for row in results:
                total_price = row['quantity'] * row['price']
                print(f"Order {row['order_id']} total is {total_price}")
            ```

*   需求：拼接用户的全名。
    *   不好的做法 (使用MySQL函数): `SELECT CONCAT(first_name, ' ', last_name) AS full_name FROM users;`
    *   替代做法 (在应用中处理):
        1.  SQL查询: `SELECT first_name, last_name FROM users;`
        2.  应用层代码 (以Java为例):
            ```java
            ResultSet rs = statement.executeQuery("SELECT first_name, last_name FROM users;");
            while (rs.next()) {
                String fullName = rs.getString("first_name") + " " + rs.getString("last_name");
                System.out.println(fullName);
            }
            ```

优点：
*   灵活性极高：你可以使用任何复杂的算法和编程语言的全部功能。
*   易于测试：业务逻辑在你的应用代码中，可以进行单元测试和集成测试。
*   降低数据库负载：数据库CPU只专注于IO和数据检索，不消耗在计算上。

缺点：
*   网络开销大：需要将更多原始数据从数据库传输到应用程序。
*   可能导致应用内存压力：如果结果集非常大，在应用内存中处理可能成为瓶頸。

### 通过数据库设计和结构来规避计算

这种方法是在数据库表结构设计阶段就提前考虑，通过增加冗余字段或使用关联表来避免在查询时进行实时计算或逻辑判断。

#### 1. 预计算与反范式化 (Denormalization)

在写入或更新数据时就计算好结果，并将其存储在一个专门的列中。查询时直接读取这个结果列即可。

*   需求：获取订单总价。
*   替代做法：在 `order_items` 表中增加一个 `total_price` 列。
    *   当创建或更新订单项时，应用程序会计算 `quantity * price` 的结果，并同时写入 `total_price` 列。
    *   查询时，SQL语句变为：`SELECT order_id, total_price FROM order_items;` 这样就完全避免了查询时的运算。

#### 2. 使用查找表 (Lookup Tables)

对于需要根据某个值转换成另一个值的场景（类似函数或`CASE`语句的功能），可以使用一个独立的“查找表”。

*   需求：根据状态码（1, 2, 3）显示状态名称（"待处理", "处理中", "已完成"）。
*   不好的做法 (使用`CASE`): `SELECT CASE status WHEN 1 THEN '待处理' WHEN 2 THEN '处理中' ELSE '已完成' END FROM orders;`
*   替代做法：
    1.  创建一个新表 `order_statuses` (id INT, status_name VARCHAR)。
    2.  插入数据：(1, '待处理'), (2, '处理中'), (3, '已完成')。
    3.  查询时使用 `JOIN`：
        ```sql
        SELECT o.order_id, s.status_name
        FROM orders AS o
        JOIN order_statuses AS s ON o.status = s.id;
        ```
    `JOIN` 是一种结构化查询，不是函数或运算，因此符合你的要求。

### 使用数据库的编程功能（间接方式）

虽然你的即时查询（Ad-hoc Query）不使用函数和运算，但你可以将这些逻辑封装在数据库的其他功能中。

#### 1. 视图 (Views)

创建一个视图，在视图的定义中包含计算和函数。然后你的应用程序查询这个视图，就好像它是一张普通的、已经计算好的表。

*   需求：获取用户全名和年龄。
*   替代做法：
    1.  创建视图：
        ```sql
        CREATE VIEW v_user_details AS
        SELECT
            id,
            CONCAT(first_name, ' ', last_name) AS full_name,
            TIMESTAMPDIFF(YEAR, date_of_birth, CURDATE()) AS age
        FROM users;
        ```
    2.  应用程序查询：`SELECT full_name, age FROM v_user_details WHERE id = 123;`
        这个查询本身没有任何函数或运算。

#### 2. 存储过程 (Stored Procedures)

将复杂的逻辑封装在一个存储过程中。应用程序只需调用这个存储过程即可。

*   需求：创建一个复杂的月度报告。
*   替代做法：
    1.  创建一个存储过程，内部包含各种计算、聚合函数和逻辑。
    2.  应用程序调用：`CALL generate_monthly_report('2025-08');`
