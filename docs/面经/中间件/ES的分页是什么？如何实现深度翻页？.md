
#### ES 的分页是什么
- **定义**：
  - ES 的分页是指在搜索结果中按指定大小（`size`）和偏移（`from`）返回部分数据，适合展示大量结果的分页浏览。
- **默认机制**：
  - 使用 `from` 和 `size` 参数：
    - `from`：起始位置（偏移量）。
    - `size`：每页返回的文档数。
  - 示例：
```json
GET /users/_search
{
  "from": 0,
  "size": 10,
  "query": { "match_all": {} }
}
```
  - 返回第 1 页的 10 条记录。

#### 如何实现深度翻页
- **问题**：
  - 深度翻页（如 `from: 10000`）会导致性能下降，因 ES 需扫描并排序所有前序文档。
- **解决方案**：
  1. **Search After**：
     - 使用最后一条记录的排序值（`sort`）继续翻页，避免偏移。
  2. **Scroll API**：
     - 为批量处理设计，维护快照，适合全量数据遍历。
  3. **Point-in-Time（PIT） + Search After**：
     - 结合快照和排序，优化实时翻页。

#### 核心点
- 默认分页适合浅层，深度翻页用 Search After 或 Scroll。

---

### 1. ES 分页详解
#### (1) 默认分页（from/size）
- **原理**：
  - ES 在所有分片执行查询，收集 `from + size` 条记录，排序后返回 `size` 条。
- **流程**：
  1. 协调节点分发查询到各分片。
  2. 每个分片返回前 `from + size` 条排序结果。
  3. 协调节点合并，跳过 `from` 条，取 `size` 条。
- **局限**：
  - 深度翻页（如 `from: 10000`）需扫描大量数据，内存和 CPU 消耗大。
  - 默认限制：`index.max_result_window` 为 10000。

#### (2) 示例
```json
GET /users/_search
{
  "from": 20,
  "size": 10,
  "query": { "match": { "name": "Alice" } },
  "sort": [
    { "age": "desc" },
    { "_id": "asc" }
  ]
}
```
- 返回第 3 页（21-30 条），按 `age` 降序。

---

### 2. 深度翻页实现
#### (1) Search After
- **原理**：
  - 使用最后一条记录的排序值（`search_after`）作为游标，跳过偏移，直接获取下一页。
- **优点**：
  - 性能稳定，避免扫描前序数据。
  - 适合实时翻页。
- **要求**：
  - 必须指定 `sort` 字段，且排序唯一（如加 `_id`）。
- **流程**：
  1. 首次查询返回结果和最后一条的排序值。
  2. 下次查询用 `search_after` 指定排序值。
- **示例**：
```json
// 首次查询
GET /users/_search
{
  "size": 10,
  "query": { "match_all": {} },
  "sort": [
    { "age": "desc" },
    { "_id": "asc" }
  ]
}
// 返回结果
{
  "hits": [
    { "_id": "1", "age": 30 },
    ...
    { "_id": "10", "age": 25 } // 最后一条
  ]
}

// 下一页
GET /users/_search
{
  "size": 10,
  "query": { "match_all": {} },
  "search_after": [25, "10"], // [age, _id]
  "sort": [
    { "age": "desc" },
    { "_id": "asc" }
  ]
}
```

#### (2) Scroll API
- **原理**：
  - 创建一个查询快照（Scroll Context），维护结果集，逐批返回数据。
- **优点**：
  - 适合全量导出（如日志分析）。
- **缺点**：
  - 快照固定，不反映新写入数据。
  - 占用内存，需定期清理。
- **流程**：
  1. 发起 Scroll 查询，获取 `scroll_id`。
  2. 用 `scroll_id` 获取下一批。
  3. 结束后清除 Scroll。
- **示例**：
```json
// 首次 Scroll
POST /users/_search?scroll=1m
{
  "size": 100,
  "query": { "match_all": {} }
}
// 返回 scroll_id 和 100 条

// 下一批
POST /_search/scroll
{
  "scroll": "1m",
  "scroll_id": "DXF1ZXJ5QW5kRmV0Y2gB..."
}
```

#### (3) Point-in-Time（PIT） + Search After
- **原理**：
  - PIT 创建轻量快照，结合 `search_after` 翻页，优化实时性。
- **优点**：
  - 比 Scroll 轻量，支持动态数据。
- **流程**：
  1. 创建 PIT。
  2. 用 PIT 和 `search_after` 查询。
  3. 删除 PIT。
- **示例**：
```json
// 创建 PIT
POST /users/_pit?keep_alive=1m
{
  "id": "46ToAwMD..."
}

// 首次查询
POST /_search
{
  "size": 10,
  "pit": { "id": "46ToAwMD...", "keep_alive": "1m" },
  "sort": [
    { "age": "desc" },
    { "_id": "asc" }
  ]
}

// 下一页
POST /_search
{
  "size": 10,
  "pit": { "id": "46ToAwMD...", "keep_alive": "1m" },
  "search_after": [25, "10"],
  "sort": [
    { "age": "desc" },
    { "_id": "asc" }
  ]
}
```

---

### 3. 深度翻页对比
| **方法**         | **适用场景**         | **优点**                     | **缺点**                     |
|------------------|----------------------|------------------------------|------------------------------|
| from/size        | 浅层翻页（<10000）  | 简单易用                     | 深度翻页性能差               |
| Search After     | 实时深度翻页         | 高效、动态                   | 需唯一排序                   |
| Scroll API       | 全量数据导出         | 适合批量处理                 | 快照固定，内存占用高         |
| PIT + Search After | 实时深度翻页       | 轻量、支持动态数据           | 配置稍复杂                   |

---

### 4. 优化建议
- **限制深度**：
  - 业务上避免超深翻页（如限制 100 页）。
- **缓存**：
  - 用 Redis 缓存热点页。
- **索引优化**：
  - 精简 `_source`，减少传输。
- **排序唯一**：
  - 总加 `_id` 避免重复。

---

### 5. 延伸与面试角度
- **与性能**：
  - 深度翻页影响分片内存，需监控。
- **实际应用**：
  - 电商：商品列表翻页。
  - 日志：导出历史记录。
- **调试**：
  - `_cat/shards` 检查分片负载。
- **面试点**：
  - 问“分页”时，提 from/size 局限。
  - 问“深度”时，提 Search After。

---

### 总结
ES 默认分页用 `from/size`，适合浅层；深度翻页推荐 **Search After**（高效实时）或 **Scroll**（全量导出），PIT 结合 Search After 更灵活。面试时，可提实现代码或对比优劣，展示理解深度。