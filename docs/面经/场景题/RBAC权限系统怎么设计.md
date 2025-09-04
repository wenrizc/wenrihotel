
RBAC（Role-Based Access Control）权限系统的设计是企业级后端开发中非常核心和常见的一环。它能够极大地简化权限管理，将复杂的用户-权限关系解耦。您提出的问题还增加了一个非常实际的挑战：如何在此基础上扩展，实现数据级别的权限控制。

我的设计思路会分为两个部分：首先，构建一个经典的、通用的RBAC模型；然后，在其基础上无缝地增加一个数据权限层。

### 经典RBAC模型设计

经典的RBAC模型，也常被称为RBAC0或RBAC1，核心思想是通过“角色（Role）”这个中间层来连接“用户（User）”和“权限（Permission）”。用户不再直接拥有权限，而是通过被赋予一个或多个角色来间接获得这些角色所包含的所有权限。

#### 1. 核心实体与关系

*   用户（User）：系统的使用者。
*   角色（Role）：权限的集合，代表一个身份或职责，如“管理员”、“文章编辑”、“普通会员”。
*   权限（Permission）：一个具体的操作许可，通常是原子化的。比如“创建文章”、“删除用户”、“查看报表”。我倾向于采用 `资源:操作` 的格式来定义，例如 `article:create`, `user:delete`。

它们之间的关系是：
*   一个用户可以拥有多个角色。（多对多）
*   一个角色可以包含多个权限。（多对多）

#### 2. 数据库表结构设计

根据上述关系，我们需要五张表来构建这个模型：

1.  `users` 表：存储用户信息。
    *   `id` (PK)
    *   `username`
    *   `password_hash`
    *   ... (其他业务字段)

2.  `roles` 表：存储角色定义。
    *   `id` (PK)
    *   `role_name` (e.g., 'admin', 'editor')
    *   `description`

3.  `permissions` 表：存储权限定义。
    *   `id` (PK)
    *   `permission_code` (e.g., 'article:create', 'article:edit')
    *   `description`

4.  `user_roles` 关联表：连接用户和角色。
    *   `user_id` (FK to users.id)
    *   `role_id` (FK to roles.id)
    *   (PRIMARY KEY on `user_id`, `role_id`)

5.  `role_permissions` 关联表：连接角色和权限。
    *   `role_id` (FK to roles.id)
    *   `permission_id` (FK to permissions.id)
    *   (PRIMARY KEY on `role_id`, `permission_id`)

#### 3. 权限校验流程

当一个用户尝试执行某个操作时，后端的校验逻辑如下：
1.  接收到请求，获取当前用户的 `user_id` 和需要校验的 `permission_code`。
2.  通过 `user_id` 在 `user_roles` 表中查询该用户拥有的所有 `role_id`。
3.  拿着这些 `role_id`，在 `role_permissions` 表中查询这些角色所对应的所有 `permission_id`。
4.  根据这些 `permission_id` 去 `permissions` 表中查询出具体的 `permission_code` 集合。
5.  判断用户请求的 `permission_code` 是否存在于这个集合中。如果存在，则授权访问；否则，拒绝。

这个模型解决了“用户能否进行某类操作”的问题。例如，角色为“文章编辑”的用户，拥有 `article:edit` 权限，他就可以进入编辑文章的页面。

### 增加数据权限设计

经典RBAC的局限性在于，它只能控制到“操作”级别，而不能控制到“数据行”级别。比如，“文章编辑”可以编辑文章，但他应该只能编辑自己创建的文章，而不是所有人的文章。这就是数据权限要解决的问题。

按照您的要求，我们通过增加表来实现这个功能。

#### 1. 设计思路

我们的核心思路是引入一个新的“策略”层，这个策略直接将“主体（Principal）”和它能操作的“具体资源实例（Resource Instance）”以及“权限”绑定起来。

*   主体：可以是单个用户，也可以是一个角色。这样设计更灵活。
*   资源实例：指代数据库中某张表的某一行数据，比如ID为123的文章。

#### 2. 新增数据权限表

我将设计一张名为 `data_access_policies` 的表：

6.  `data_access_policies` 表：
    *   `id` (PK)
    *   `principal_type` (ENUM/VARCHAR: 'user', 'role')：指明主体是用户还是角色。
    *   `principal_id` (BIGINT)：对应 `users.id` 或 `roles.id`。
    *   `resource_type` (VARCHAR: 'article', 'order')：指明是哪种类型的资源。
    *   `resource_id` (BIGINT)：具体资源的ID。**这是数据权限的核心**。我们可以用一个特殊值（如0或NULL）来代表“所有资源”。
    *   `permission_id` (FK to permissions.id)：允许在此资源上执行的操作。

#### 完整的权限校验流程

现在，当一个用户（比如 `user_id=101`）尝试编辑ID为 `456` 的文章（`article:edit`）时，校验流程会变得更加严谨：

1.  **第一步：操作权限校验（经典RBAC检查）**
    *   系统首先检查用户`101`的角色是否拥有 `article:edit` 这个通用权限。
    *   这是个快速的前置检查，如果用户连编辑文章的资格都没有，就直接拒绝，无需进行下一步。

2.  **第二步：数据权限校验（如果第一步通过）**
    *   系统现在需要判断，用户`101`是否有权编辑**这篇ID为456的特定文章**。
    *   此时，我们会去查询 `data_access_policies` 表，查询条件会是：
        ```sql
        SELECT 1 FROM data_access_policies
        WHERE
          permission_id = (SELECT id FROM permissions WHERE permission_code = 'article:edit')
          AND resource_type = 'article'
          AND (
            -- 策略1: 允许操作所有文章
            resource_id IS NULL OR resource_id = 0
            OR
            -- 策略2: 明确允许操作这篇特定的文章
            resource_id = 456
          )
          AND (
            -- 策略3: 权限直接授予给了该用户
            (principal_type = 'user' AND principal_id = 101)
            OR
            -- 策略4: 权限授予给了该用户所属的某个角色
            (principal_type = 'role' AND principal_id IN (SELECT role_id FROM user_roles WHERE user_id = 101))
          )
        LIMIT 1;
        ```
    *   如果上述查询能返回结果，说明存在一条策略允许此操作，校验通过。
    *   如果查询没有返回结果，即使他有 `article:edit` 的通用权限，也会因为没有具体这篇文稿的数据权限而被拒绝。

#### 示例场景：
*   **编辑自己文章**：当用户创建文章时，系统可以自动在 `data_access_policies` 表中插入一条记录：`('user', user_id, 'article', new_article_id, 'article:edit')`。
*   **部门经理查看本部门数据**：可以为“部门经理”这个角色，批量插入他所管辖部门所有员工创建的订单的查看策略。
