# Redis 数据结构、持久化与缓存问题

## Redis 是什么

Redis 是高性能内存数据结构存储，可用作缓存、计数器、排行榜、分布式锁、消息队列、限流器和部分实时状态存储。

核心特点：

- 数据主要在内存中，读写速度快。
- 支持多种数据结构。
- 单线程执行命令模型简单，避免很多锁竞争。
- 支持持久化、主从复制、哨兵和集群。
- 丰富的原子命令和 Lua 脚本能力。

## 常见数据结构

| 类型 | 特点 | 常见场景 |
| --- | --- | --- |
| String | 字符串/整数/二进制安全 | 缓存、计数器、分布式锁 |
| Hash | field-value 映射 | 用户对象、商品属性 |
| List | 双端列表 | 简单队列、时间线 |
| Set | 无序去重集合 | 标签、共同好友、去重 |
| Sorted Set | 带分数的有序集合 | 排行榜、延迟队列 |
| Bitmap | 位图 | 签到、布尔状态统计 |
| HyperLogLog | 基数估算 | UV 统计 |
| Stream | 日志型消息结构 | 消息队列、事件流 |

## String

常用命令：

```text
SET key value
GET key
INCR key
DECR key
SETEX key seconds value
SET key value NX EX seconds
```

场景：

- 缓存对象 JSON。
- 计数器。
- 验证码。
- 分布式锁。

## Hash

```text
HSET user:1 name tom age 18
HGET user:1 name
HGETALL user:1
```

适合存储对象字段。相比把整个对象 JSON 存 String，Hash 可以单独更新某个字段。

注意：对象字段很多或 value 很大时，要关注内存和网络开销。

## List

```text
LPUSH queue task1
RPOP queue
BRPOP queue 5
```

适合简单队列，但功能不如 Stream、Kafka、RocketMQ 完整。

问题：

- 消费确认和重试能力弱。
- 消费者宕机时消息可靠性需要额外设计。

## Pub/Sub 发布订阅

Redis Pub/Sub 是发布订阅模型，发布者向 channel 发送消息，订阅者订阅 channel 后实时接收消息。

常用命令：

```text
SUBSCRIBE order_event
PUBLISH order_event "order paid"
PSUBSCRIBE order_*
```

适合场景：

- 实时通知，例如配置变更通知、缓存失效通知。
- 在线用户消息推送，例如 WebSocket 网关之间广播事件。
- 轻量级事件广播，例如后台任务状态变更。
- 多个服务实例之间做简单消息分发。

局限性：

- Pub/Sub 不持久化消息，订阅者离线时会丢消息。
- 没有消费确认、重试、死信队列等机制。
- 不适合订单、支付、库存扣减这类要求可靠投递的业务。
- 如果需要可靠消息队列，优先考虑 Redis Stream、Kafka、RocketMQ 等方案。

面试回答：

> Redis Pub/Sub 适合实时广播和轻量通知，特点是简单、延迟低，但消息不落盘、不保证离线可达，也没有 ACK 和重试。可靠消息场景不能只依赖 Pub/Sub，应该使用 Redis Stream 或专业 MQ。

## Stream

Redis Stream 是日志型消息结构，比 List/Pub/Sub 更适合做轻量可靠队列。

常见命令：

```text
XADD stream.orders * order_id 1001 status paid
XGROUP CREATE stream.orders group1 0 MKSTREAM
XREADGROUP GROUP group1 consumer1 COUNT 10 STREAMS stream.orders >
XACK stream.orders group1 message_id
```

核心概念：

- Stream：消息日志。
- Consumer Group：消费者组，同一组内一条消息通常由一个消费者处理。
- Pending List：已投递但未 ACK 的消息。
- ACK：消费者处理成功后确认。

注意：

- Stream 比 Pub/Sub 可靠，但仍要设计幂等、重试、死信或补偿。
- 消费者宕机后，Pending 消息需要被其他消费者认领处理。
- 大规模复杂消息系统仍优先 Kafka、RocketMQ 等专业 MQ。

## Set

```text
SADD tags go redis mysql
SISMEMBER tags go
SINTER set1 set2
```

适合去重和集合运算。

## Sorted Set

```text
ZADD rank 100 user1
ZREVRANGE rank 0 9 WITHSCORES
ZRANGEBYSCORE delay 0 1710000000
```

典型场景：

- 排行榜。
- 按时间分数做延迟任务。
- Top N。

实现原理面试版：

- 大集合通常使用跳表 + 字典。
- 字典提供成员到分数的 O(1) 查找。
- 跳表维护有序性，支持范围查询。
- 较小集合会使用紧凑编码节省内存，具体编码随 Redis 版本可能变化，不要死背 ziplist。

## Redis 为什么快

- 主要基于内存操作。
- 单线程命令执行，减少锁竞争和上下文切换。
- I/O 多路复用处理大量连接。
- 数据结构实现高效。
- pipeline 可减少网络往返。

注意：Redis 单线程指命令执行核心路径。持久化、异步释放、网络 I/O 等在新版本中可能有辅助线程。

## Redis 持久化

### RDB

RDB 是快照持久化，把某一时刻的数据生成快照文件。

触发方式：

- 手动 `SAVE`。
- 后台 `BGSAVE`。
- 配置 `save` 规则。
- 主从复制初次同步。
- 关闭服务时根据配置触发。

优点：

- 文件紧凑。
- 恢复速度较快。
- 对主线程影响相对较小，因为通常由子进程生成快照。

缺点：

- 可能丢失最近一次快照后的数据。
- fork 子进程可能造成瞬时资源压力。

### AOF

AOF 记录写命令日志，重启时重放恢复数据。

配置：

```text
appendonly yes
appendfsync everysec
```

fsync 策略：

- always：每次写都同步磁盘，最安全但最慢。
- everysec：每秒同步，常用折中方案。
- no：由操作系统决定，性能好但丢数据风险更高。

优点：

- 数据安全性通常高于 RDB。
- 日志可读性较强。
- 支持 AOF rewrite 压缩日志。

缺点：

- 文件通常更大。
- 恢复速度可能慢于 RDB。
- fsync 策略会影响性能。

### RDB + AOF

生产中常同时开启。恢复时通常优先使用 AOF，因为数据更完整。

## 过期删除策略

Redis key 设置 TTL 后，不是到期瞬间就一定被删除。

常见机制：

- 惰性删除：访问 key 时发现已过期，再删除。
- 定期删除：后台周期性抽样检查一批过期 key。
- 内存淘汰：内存达到上限时，根据 maxmemory-policy 淘汰 key。

易错点：

- 过期删除不等于内存淘汰，前者看 TTL，后者看内存压力。
- key 过期时间集中会造成缓存雪崩，通常要加随机 TTL。
- 过期 key 没被访问时可能短暂占用内存，不能认为到点就立刻释放。

## 缓存雪崩、击穿、穿透

### 缓存雪崩

大量 key 同时过期或 Redis 故障，导致请求集中打到数据库。

解决：

- 过期时间加随机值。
- 热点 key 不同时过期。
- 多级缓存。
- 限流降级。
- Redis 高可用。
- 后台预热。

### 缓存击穿

某个热点 key 过期，大量请求同时打到数据库。

解决：

- 热点 key 永不过期或逻辑过期。
- 互斥锁重建缓存。
- singleflight 合并请求。
- 提前异步刷新。

### 缓存穿透

请求查询不存在的数据，缓存和数据库都没有，导致每次都打数据库。

解决：

- 缓存空值并设置短 TTL。
- 布隆过滤器。
- 参数校验和风控。

## 缓存一致性

常见策略：先更新数据库，再删除缓存。

为什么不是更新缓存？

- 写多读少时频繁更新缓存浪费。
- 复杂对象缓存容易出现并发覆盖。
- 删除后下次读取再加载，更简单。

并发问题：

1. 线程 A 更新数据库。
2. 线程 B 读取旧数据并写入缓存。
3. 线程 A 删除缓存。

或者删除和写库顺序交错仍可能短暂不一致。

常见增强：

- 延迟双删。
- binlog 监听异步删缓存。
- 缓存设置合理 TTL。
- 关键读走数据库或加版本号。

## 淘汰策略

Redis 内存达到上限时，会根据策略淘汰 key。

常见策略：

- noeviction：不淘汰，写入报错。
- allkeys-lru：所有 key 中淘汰最近最少使用。
- volatile-lru：带过期时间的 key 中淘汰 LRU。
- allkeys-lfu：所有 key 中淘汰最不常用。
- volatile-lfu：带过期时间的 key 中淘汰 LFU。
- allkeys-random / volatile-random：随机淘汰。
- volatile-ttl：淘汰 TTL 最小的 key。

LRU 适合近期访问更可能再次访问的场景。LFU 适合访问频率稳定、热点长期存在的场景。

## Redis 高可用和集群

### 主从复制

主节点负责写，从节点复制数据并分担读请求。复制通常是异步的，所以主从切换时可能丢失少量最新写入。

### 哨兵 Sentinel

哨兵负责监控、故障发现、自动选主和通知客户端。它解决的是主从架构的自动故障转移，不负责数据分片。

### Redis Cluster

Cluster 通过 16384 个 hash slot 做数据分片，不同 slot 分布到不同主节点上。

面试踩坑：

- Cluster 不是简单主从，它解决的是水平扩展和分片问题。
- 多 key 操作要求 key 落在同一个 slot，否则可能失败或需要业务拆分。
- 主从复制是异步的，强一致场景不能只依赖 Redis 高可用。
- Redis Cluster 扩容、迁移 slot 时，要关注客户端重定向、热点 key 和迁移期间的性能波动。

## Redis 与 Memcached

| 对比项 | Redis | Memcached |
| --- | --- | --- |
| 数据结构 | 丰富 | 主要是 key-value |
| 持久化 | 支持 | 不支持或很弱 |
| 高可用 | 主从、哨兵、集群 | 依赖客户端和外部方案 |
| 原子操作 | 丰富 | 较少 |
| 内存模型 | 更复杂，功能多 | 简单高效 |
| 场景 | 缓存 + 数据结构 + 分布式能力 | 纯缓存 |

现在后端面试中 Redis 更高频。

## 面试追问

### Redis 单线程为什么还能高性能？

因为 Redis 大部分操作在内存中完成，命令执行很快；单线程避免锁竞争；网络层使用 I/O 多路复用；同时 Redis 的数据结构和编码优化做得很好。

### 大 key 有什么问题？

- 阻塞 Redis 主线程。
- 网络传输慢。
- 删除成本高。
- 主从同步和持久化压力大。

优化：拆分 key、限制 value 大小、使用异步删除、避免一次取全量。

### 热 key 怎么处理？

- 本地缓存。
- 多副本读。
- key 拆分。
- 限流降级。
- 提前预热。
- 监控热点访问。

## 文章导航
上一篇：[04-MongoDB-ES-搜索与日志分析](../数据库/04-MongoDB-ES-搜索与日志分析.md)
下一篇：[02-Redis分布式锁-限流-一致性](02-Redis分布式锁-限流-一致性.md)
