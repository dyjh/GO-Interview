# PostgreSQL 介绍、MySQL 对比与 GORM 兼容性

## 核心定位

PostgreSQL 常简称 PG 或 PgSQL，是功能很强的开源关系型数据库。它和 MySQL 一样支持 SQL、事务、索引、主从复制等关系型数据库能力，但整体风格更偏“标准 SQL、复杂查询、丰富数据类型和可扩展能力”。

面试里不要简单说“PostgreSQL 比 MySQL 强”或者“MySQL 比 PostgreSQL 快”。更好的回答是：

- MySQL 更常见于互联网业务的高并发 OLTP、简单查询、成熟运维和生态场景。
- PostgreSQL 更适合复杂 SQL、数据一致性要求高、模型复杂、JSON/地理位置/全文检索/扩展能力强的场景。
- 两者都能做核心业务库，真正选型要看团队经验、查询复杂度、运维体系、生态工具和未来扩展方向。

## PostgreSQL 常见特点

PostgreSQL 的优势通常体现在这些方面：

- SQL 能力完整，窗口函数、CTE、复杂子查询、聚合和表达式能力强。
- 数据类型丰富，支持数组、JSONB、范围类型、枚举、UUID、几何类型等。
- 索引类型丰富，除了 B-Tree，还常见 GIN、GiST、BRIN、Hash 等。
- JSONB 配合 GIN 索引，适合半结构化字段查询。
- 扩展能力强，例如 PostGIS 做地理空间，pg_trgm 做相似文本匹配。
- MVCC 实现成熟，读写并发体验好，但需要关注 vacuum / autovacuum 和表膨胀。
- 事务和约束能力强，外键、唯一约束、检查约束、部分索引、表达式索引都很常用。

常见注意点：

- 运维上要理解 autovacuum、连接数、WAL、复制延迟和膨胀治理。
- 默认连接模型是进程/连接占用资源较重，高并发短连接场景通常需要连接池，例如 PgBouncer 或应用侧连接池。
- 国内 MySQL 生态和经验更普遍，团队迁移到 PostgreSQL 要考虑学习和运维成本。

## PostgreSQL 和 MySQL 的区别

| 对比项 | MySQL | PostgreSQL |
| --- | --- | --- |
| 常见定位 | 互联网 OLTP、简单高频读写、成熟生态 | 复杂查询、强 SQL 能力、丰富类型和扩展 |
| SQL 能力 | 常用 SQL 足够，业务开发简单直接 | 更接近标准 SQL，复杂查询表达力强 |
| 事务与 MVCC | InnoDB 依赖 undo log、redo log、锁和 MVCC | MVCC 基于多版本行，旧版本回收依赖 vacuum |
| 索引能力 | B+Tree 最常用，也有全文、空间等能力 | B-Tree、GIN、GiST、BRIN、表达式索引、部分索引更常用 |
| JSON 能力 | 支持 JSON，常配合生成列或函数索引使用 | JSONB 生态成熟，配合 GIN 索引查询体验好 |
| 分库分表生态 | 互联网场景经验多，分库分表方案成熟 | 单库能力强，水平拆分常依赖业务、分区或 Citus 等方案 |
| 复制和高可用 | 主从复制、半同步、MGR、Proxy 生态丰富 | 流复制、逻辑复制成熟，HA 依赖 Patroni 等方案较常见 |
| 运维普及度 | 国内团队经验更普遍 | 需要团队掌握 vacuum、WAL、连接池等机制 |
| 典型易错点 | 隔离级别、间隙锁、字符集、索引失效 | autovacuum、表膨胀、连接数、大小写与引号 |

容易混的点：

- PostgreSQL 的未加引号标识符会默认转小写；MySQL 对大小写的表现还受系统和配置影响。跨库迁移时表名、字段名最好统一小写加下划线。
- MySQL 的 `AUTO_INCREMENT` 和 PostgreSQL 的 `serial` / identity 都能做自增主键，但 DDL 写法不同。
- MySQL 的 `ON DUPLICATE KEY UPDATE` 和 PostgreSQL 的 `ON CONFLICT DO UPDATE` 都能做 upsert，但语法不同。
- JSON 查询语法差异明显，例如 MySQL 常见 `JSON_EXTRACT`，PostgreSQL 常见 `->`、`->>`、`@>`。

## 不同业务场景怎么选

### 优先 MySQL 的场景

- 业务以简单 CRUD、高并发读写、订单、用户、库存、支付流水等 OLTP 为主。
- 团队 MySQL 经验丰富，已有完善的备份、监控、主从、分库分表和故障处理体系。
- 需要兼容大量历史系统、中间件、报表工具或云厂商默认方案。
- 查询模式清晰，复杂聚合和复杂 SQL 不多。
- 更看重生态成熟度、招聘和维护成本。

一句话：标准互联网业务、简单高频交易链路、团队 MySQL 能力强，优先 MySQL 更稳。

### 优先 PostgreSQL 的场景

- 查询复杂，涉及多表 join、窗口函数、递归 CTE、复杂聚合或强约束。
- 业务模型复杂，需要数组、JSONB、范围类型、UUID、枚举、地理空间等数据类型。
- 希望在一个数据库里同时处理关系数据和一部分半结构化数据。
- 需要表达式索引、部分索引、GIN/GiST 等更灵活的索引能力。
- 对数据一致性、约束表达、SQL 标准能力要求更高。

一句话：复杂业务模型、复杂查询、强约束、JSONB 或 GIS 场景，PostgreSQL 更有优势。

### 混合选型

大型系统里也可以混合使用：

- 核心交易链路用 MySQL，复杂报表或内部数据平台用 PostgreSQL / OLAP。
- 主业务库用 PostgreSQL，搜索场景同步到 ES/OpenSearch。
- 简单缓存和排行榜用 Redis，最终数据仍落 MySQL 或 PostgreSQL。
- 地理位置检索用 PostgreSQL + PostGIS，交易订单仍用 MySQL。

混用时要注意：数据同步、事务边界、最终一致、运维复杂度和团队能力。不要为了“技术更强”引入多个数据库，除非业务收益能覆盖复杂度。

## GORM 是否同时兼容 MySQL 和 PostgreSQL

GORM 支持 MySQL 和 PostgreSQL，分别使用不同 driver：

```go
import (
    "gorm.io/driver/mysql"
    "gorm.io/driver/postgres"
    "gorm.io/gorm"
)

func OpenMySQL(dsn string) (*gorm.DB, error) {
    return gorm.Open(mysql.Open(dsn), &gorm.Config{})
}

func OpenPostgres(dsn string) (*gorm.DB, error) {
    return gorm.Open(postgres.Open(dsn), &gorm.Config{})
}
```

### 哪些查询基本兼容

GORM 的链式 API 通常可以同时兼容两者：

```go
db.Where("status = ? AND created_at >= ?", 1, start).
    Order("created_at DESC").
    Limit(20).
    Find(&orders)
```

这类写法一般由 GORM 根据不同方言生成 SQL，常见的 `Create`、`First`、`Find`、`Where`、`Order`、`Limit`、`Offset`、`Joins`、`Preload`、`Transaction` 都比较容易复用。

GORM 的 `?` 占位符写法也通常可以复用，底层 driver 会处理实际占位符和参数绑定。不要自己拼接用户输入，仍然要使用参数绑定。

### 哪些查询不完全兼容

只要写到数据库方言特性，就不能认为 MySQL 和 PostgreSQL 完全兼容：

| 场景 | MySQL 写法 | PostgreSQL 写法 |
| --- | --- | --- |
| JSON 字段取值 | `JSON_EXTRACT(extra, '$.source')` | `extra->>'source'` |
| Upsert | `ON DUPLICATE KEY UPDATE` | `ON CONFLICT (...) DO UPDATE` |
| 字符串拼接 | `CONCAT(a, b)` | `a || b` 或 `concat(a, b)` |
| 随机排序 | `RAND()` | `RANDOM()` |
| 当前时间 | `NOW()` 常用 | `NOW()` / `CURRENT_TIMESTAMP` 常用 |
| 全文检索 | `MATCH ... AGAINST` | `to_tsvector` / `to_tsquery` |
| 锁语义 | InnoDB 间隙锁、临键锁常见 | 行锁、谓词锁、SSI 等机制不同 |

建议：

- 普通 CRUD 尽量用 GORM API，不直接写方言 SQL。
- upsert 尽量用 `clause.OnConflict`，让 GORM 生成不同数据库的写法。
- JSON、全文检索、锁、复杂分页、CTE、窗口函数等场景，可以按数据库封装 repository 或 scope。
- 迁移时不要只改 driver，要用 `DryRun` 或日志查看生成 SQL，并跑集成测试。
- `AutoMigrate` 适合开发期快速建表，生产环境跨 MySQL / PostgreSQL 迁移建议用专门 migration 工具并人工 review DDL。

### 字段类型和标签注意

GORM model 结构体可以复用，但字段标签可能要分数据库处理：

```go
type UserEvent struct {
    ID      uint64
    UserID  uint64
    Payload string `gorm:"type:json"`  // MySQL 常见
    // Payload datatypes.JSON `gorm:"type:jsonb"` // PostgreSQL 常见
}
```

常见差异：

- JSON：MySQL 常用 `json`，PostgreSQL 常用 `jsonb`。
- 自增主键：MySQL 是 `AUTO_INCREMENT`，PostgreSQL 可用 identity / sequence。
- 时间类型、时区、默认值函数要明确约定。
- 唯一索引、部分索引、表达式索引等高级索引在两边写法不同。

## 面试追问

### PostgreSQL 和 MySQL 怎么选？

简单高频 OLTP、团队 MySQL 经验成熟、生态和运维要求稳定时优先 MySQL。复杂查询、强 SQL 表达、JSONB、GIS、复杂约束和丰富索引能力要求高时优先 PostgreSQL。真正选型看业务查询复杂度、团队经验、运维体系和未来扩展，不要只看单点性能。

### PostgreSQL 一定比 MySQL 更适合复杂查询吗？

通常 PostgreSQL 在复杂 SQL、窗口函数、CTE、表达式索引、部分索引、JSONB 和扩展能力上更有优势，但不是所有复杂查询都应该直接压到 PostgreSQL。大规模分析、宽表聚合、实时指标仍可能更适合 ClickHouse、StarRocks、Doris 等 OLAP 系统。

### GORM 的查询语句能同时兼容 MySQL 和 PostgreSQL 吗？

普通 GORM 链式 API 大多可以兼容，GORM 会按 driver 生成对应方言 SQL。但 Raw SQL、JSON 查询、upsert、全文检索、锁、函数、字段类型和 DDL 不一定兼容。实际项目里应把通用 CRUD 和方言特性分层封装，并用集成测试验证两种数据库。

### 从 MySQL 切到 PostgreSQL 主要改哪里？

主要改 DSN 和 GORM driver、字段类型、迁移 DDL、JSON 查询、upsert、分页排序细节、锁相关逻辑、索引设计和 SQL 函数。简单 CRUD 改动少，依赖方言特性的复杂查询改动会明显增加。

## 文章导航
上一篇：[04-MongoDB-ES-搜索与日志分析](04-MongoDB-ES-搜索与日志分析.md)
下一篇：[01-Redis数据结构-持久化-缓存问题](../中间件/01-Redis数据结构-持久化-缓存问题.md)