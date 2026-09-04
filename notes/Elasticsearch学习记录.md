# Elasticsearch 学习记录

工程依托：[jdk8-elasticsearch-demo](https://github.com/xieqiansong/chaos-java/tree/main/jdk8-platform/jdk8-elasticsearch-demo)（`chaos-java` 仓库，ES 7.17.10 + Spring Data Elasticsearch 4.4，Testcontainers 集成测试，索引/文档/搜索/聚合全覆盖）。

## 1. 安装

```bash
# 单元测试（推荐）：Docker 可用时自动起 ES 7.17 容器并真实执行；无 Docker 时用例优雅跳过
mvn -pl jdk8-elasticsearch-demo -am test

# Docker 完整体验
docker compose -f jdk8-elasticsearch-demo/docker-compose.yml up -d
curl http://localhost:9200/_cluster/health
```

客户端：`RestHighLevelClient`；数据访问双轨：`ElasticsearchRestTemplate` + `ElasticsearchRepository`。

## 2. 索引管理（Index）

ES 是 schema-less 但有 mapping：字段类型一旦写入便难改。生产应**显式建索引**（settings + mappings），而非依赖首次写入的 auto-mapping（可能因首条数据误判类型）。

`IndexOperations` 支持基于实体类的「约定式」建索引（`create()` + `putMapping(Product.class)`），代码即文档。

## 3. 文档（Document）

基于 `ElasticsearchRepository` 的声明式 CRUD：

- `save`：未指定 id 时 ES 自动生成；指定 id 为 upsert。
- `saveAll`：批量写入，内部走 `_bulk`，比逐条 save 高效。
- **近实时陷阱**：默认写入后需 `refresh` 才能被搜索立即看到。演示中显式 `refresh()` 以便断言；**生产环境不要每次写入都 refresh**（有性能损耗），应依赖默认近实时刷新周期（1s）。

## 4. 搜索（Search）

| 查询 | 字段类型 | 说明 |
|---|---|---|
| `matchQuery` / `multiMatchQuery` | text | 先分词再匹配，适合商品名/描述全文检索 |
| `termQuery` | keyword | 不分词精确匹配，适合分类/状态枚举 |
| `rangeQuery` | numeric/date | 价格区间、时间区间 |
| `boolQuery` | 组合 | must（与）/ should（或，配 minimumShouldMatch）/ filter / mustNot |

## 5. 聚合（Aggregation）

类似 SQL 的 `GROUP BY`，典型场景是各分类商品数、平均价格。

- terms 聚合只能作用在 **keyword** 字段（text 需加 `.keyword` 子字段）。
- 聚合与搜索共用查询：先 query 过滤，再对过滤结果聚合。
- 只看聚合结果时 `setMaxResults(0)` 不返回明细，节省开销。

返回结构：`SearchHits → getAggregations() → 底层 ES Aggregations → terms/avg`。

## 踩坑

- **text vs keyword 选择**：全文检索用 text，精确匹配与聚合用 keyword——这是 ES 建模最易踩的坑。
- **近实时与 refresh**：演示为断言强制 refresh，生产应避免高频 refresh。
- **mapping 显式建**：别依赖 auto-mapping，首条数据可能误判类型导致后续写入失败。
- **terms 聚合只能 keyword**：对 text 字段聚合会被忽略或报错。
- **测试隔离**：ES 索引是全局状态，每个用例自建自毁（`@BeforeAll` 建、`@AfterEach` 删），避免交叉污染。

## 进阶方向

- 分词器（IK 中文分词、自定义 analyzer）、搜索结果高亮。
- 索引别名 + 重建实现零停机 reindex；集群分片/副本调优、冷热架构。
- ES 8.x + 新 Java API Client（`co.elastic.clients`）+ Spring Boot 3（需 JDK17+）；索引模板、ILM 生命周期、批量写入调优。

## 参考来源

- [Elasticsearch 官方文档](https://www.elastic.co/guide/index.html)
- [Spring Data Elasticsearch](https://docs.spring.io/spring-data/elasticsearch/docs/current/reference/html/)
