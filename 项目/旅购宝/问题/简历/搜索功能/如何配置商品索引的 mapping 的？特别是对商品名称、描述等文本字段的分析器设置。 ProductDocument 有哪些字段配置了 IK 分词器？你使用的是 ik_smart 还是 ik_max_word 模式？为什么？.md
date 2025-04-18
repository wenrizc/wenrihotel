
## 商品索引的 Mapping 配置

使用了 Spring Data Elasticsearch 框架的注解方式配置索引 mapping：

```java
@Data
@NoArgsConstructor
@Document(indexName = "product")
public class ProductDocument {
    // 各字段配置
}
```

这种方式相比手动编写 JSON 配置更加简洁和类型安全，框架会自动将注解转换为 Elasticsearch 的索引定义。

## 文本字段的分析器设置

对于商品的文本字段，我特别注重分词效果，因此做了如下配置：

```java
@Field(type = FieldType.Text, analyzer = "ik_max_word", searchAnalyzer = "ik_smart")
private String name;

@Field(type = FieldType.Text, analyzer = "ik_max_word", searchAnalyzer = "ik_smart")
private String description;
```

这里采用了**"非对称配置"**策略：
- `analyzer = "ik_max_word"`: 索引时使用最细粒度分词
- `searchAnalyzer = "ik_smart"`: 搜索时使用智能分词

## 配置了 IK 分词器的字段

在 ProductDocument 中，只有两个字段配置了 IK 分词器：
1. **name** (商品名称)
2. **description** (商品描述)

其他字段采用了不同的类型映射：
- `id`、`shopId`、`categoryId`: 使用 FieldType.Long
- `cover`: 使用 FieldType.Keyword (不进行分词的精确匹配)
- `price`: 使用 FieldType.Double
- `status`、`sold`: 使用 FieldType.Integer

## 关于 ik_max_word 和 ik_smart 的选择

我选择**索引时使用 ik_max_word、搜索时使用 ik_smart** 的原因：

1. **索引时使用 ik_max_word**：
   - 最细粒度分词，会尽可能多地切分词语
   - 例如"中华人民共和国"会被分为"中华人民共和国、中华人民、中华、华人、人民、共和国、共和、国"等
   - 提高了召回率，使得用户搜索任何相关词都能匹配到商品

2. **搜索时使用 ik_smart**：
   - 智能分词，会做最粗粒度的切分
   - 例如"中华人民共和国"只会被分为"中华人民共和国"
   - 提高查询精确度，避免过多的无关匹配
