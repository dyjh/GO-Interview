# MongoDB、Elasticsearch、搜索与日志分析

## MongoDB 适用场景

MongoDB 是文档型 NoSQL 数据库，适合：

- 数据结构灵活、字段变化频繁的业务。
- JSON 文档天然表达的对象模型。
- 内容管理、配置、画像、事件数据。
- 写入量大且可水平扩展的场景。
- 部分实时分析和聚合场景。

不适合：

- 强事务、复杂 join 很多的核心关系模型。
- 高度规范化且依赖复杂 SQL 的场景。
- 需要复杂全文检索相关性排序的搜索场景。

## Elasticsearch 适用场景

Elasticsearch 是基于 Lucene 的分布式搜索和分析引擎，适合：

- 全文检索。
- 商品搜索。
- 日志检索。
- 聚合分析。
- 自动补全和拼写纠错。
- 多条件筛选和相关性排序。

ES 通常不是主数据库，而是搜索/分析索引。主数据仍建议放在 MySQL、PostgreSQL、MongoDB 等数据库中。

## ES 和传统数据库查询的区别

| 对比项 | MySQL | Elasticsearch |
| --- | --- | --- |
| 核心目标 | 事务和结构化查询 | 搜索和分析 |
| 数据结构 | 表、行、列 | index、document、field |
| 查询方式 | SQL | Query DSL |
| 全文检索 | 能做但能力有限 | 强项，倒排索引 |
| 事务 | 强 | 弱，不适合复杂事务 |
| 实时性 | 提交后立即可读 | 近实时，refresh 后可搜索 |
| Join | 支持 | 不擅长，通常反范式设计 |

纠错：不能简单说“数据库模糊查询一定全表扫描”。如果是前缀匹配并有合适索引，MySQL 也可能使用索引。但 `%keyword%` 这类左模糊通常难以利用普通 B+Tree 索引。

## ES 为什么适合商品搜索

ES 的优势：

- 倒排索引适合关键词检索。
- 分词器支持中文、拼音、同义词等搜索体验优化。
- 相关性评分可以按标题、品牌、类目、销量、价格等综合排序。
- 支持过滤、聚合、范围查询。
- 分布式扩展，适合大数据量搜索。
- 支持自动补全和搜索建议。

商品搜索常见字段设计：

- 商品 ID。
- 标题。
- 品牌。
- 类目。
- 价格。
- 库存状态。
- 销量。
- 上下架状态。
- 标签和属性。

## ES 常用接口

### 索引管理

```http
PUT /products
GET /products
DELETE /products
PUT /products/_mapping
GET /products/_mapping
```

### 文档操作

```http
POST /products/_doc/1
GET /products/_doc/1
POST /products/_update/1
DELETE /products/_doc/1
POST /_bulk
```

### 搜索和聚合

```http
POST /products/_search
POST /products/_count
GET /_cluster/health
GET /_cat/indices?v
```

## 日志分析用 ES 还是 MongoDB

一般日志检索和分析更倾向 ES 或专门日志系统。

ES 优势：

- 全文检索强。
- 时间范围过滤和聚合方便。
- Kibana 生态成熟。
- 适合按关键词、traceId、错误堆栈检索。

MongoDB 优势：

- 文档模型灵活。
- 写入和存储半结构化数据方便。
- 对简单按字段查询、少量聚合也能胜任。

选择建议：

- 日志检索、监控、错误排查：优先 ES/OpenSearch/Loki 等日志检索系统。
- 业务文档存储、灵活 JSON 查询：MongoDB 更合适。
- 复杂实时指标分析：考虑 ClickHouse、Druid、StarRocks 等 OLAP。

## ES 使用注意事项

- ES 是近实时，不适合强一致读写事务。
- mapping 要提前设计，字段类型错了后改动成本高。
- 大字段、无用字段不要全部建索引。
- 深分页要避免 `from + size` 过大，可用 `search_after` 或滚动查询。
- 分片数量不是越多越好，过多会增加集群管理成本。
- 写入高峰要用 bulk 批量写入。
- 查询要避免高成本 wildcard、script 和深层聚合。

## MongoDB 使用注意事项

- 文档不宜无限增大。
- 高基数查询字段要建索引。
- 数组字段索引要关注膨胀。
- 需要事务时要评估性能和复杂度。
- 分片键选择要避免热点和频繁迁移。

## 面试追问

### ES 为什么快？

对全文搜索来说，ES 使用倒排索引，将词项映射到文档列表，查询关键词时不需要扫描所有文档。同时通过分片并行、缓存、列式 doc values 等机制提升搜索和聚合性能。

### ES 数据和 MySQL 不一致怎么办？

接受短暂最终一致；用消息重试、幂等更新、版本号控制、死信队列、定时全量校验和补偿任务保证最终一致。关键场景可以查询 MySQL 兜底。

### MongoDB 和 MySQL 怎么选？

强关系、事务、多表一致性和复杂 SQL 优先 MySQL。结构灵活、文档天然聚合、字段变化频繁、横向扩展需求强时可考虑 MongoDB。

## 文章导航
上一篇：[03-MySQL分库分表-读写分离](03-MySQL分库分表-读写分离.md)
下一篇：[05-PostgreSQL介绍-MySQL对比-GORM兼容](05-PostgreSQL介绍-MySQL对比-GORM兼容.md)
