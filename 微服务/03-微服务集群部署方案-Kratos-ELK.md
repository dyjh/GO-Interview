# 微服务集群部署方案：Kratos、服务治理与 ELK

## 核心定位

严格说，HTTP、TCP、UDP、WebSocket 属于网络协议和接口通信基础，不完全等同于微服务。因此目录层面更适合叫“网络与微服务”：前面讲通信协议，后面讲服务拆分、注册中心、集群部署、可观测性和稳定性治理。

一套比较稳的 Go 微服务集群推荐基线：

```text
客户端 / 前端
  -> CDN / WAF / SLB
  -> API Gateway / Ingress / Nginx
  -> Kratos 服务集群（HTTP + gRPC）
  -> Redis / MySQL 或 PostgreSQL / MQ / 对象存储
  -> ELK 日志 + Prometheus 指标 + OpenTelemetry 链路追踪
```

面试里可以概括为：用 Kratos 统一微服务工程结构和 HTTP/gRPC 接入，用 Kubernetes 做部署、弹性伸缩和滚动发布，用注册中心维护服务实例目录并解决实例寻址，用 Redis、数据库、MQ 处理状态和异步解耦，用 ELK 做日志检索，再补 Prometheus/Grafana 和 tracing 做完整可观测性。

## 推荐部署架构

| 模块 | 推荐方案 | 作用 |
| --- | --- | --- |
| 服务框架 | Kratos | 统一 HTTP/gRPC、middleware、配置、错误码、工程分层 |
| 容器编排 | Kubernetes | Deployment、Service、ConfigMap、Secret、HPA、滚动发布 |
| 流量入口 | Ingress / API Gateway / Nginx | TLS、路由、鉴权前置、限流、灰度、统一入口 |
| 注册中心 | K8s Service/DNS；非 K8s 可用 Consul/Nacos | 维护实例目录，承载注册、健康检查和实例查询 |
| 缓存 | Redis Sentinel / Redis Cluster | 热点缓存、分布式锁、限流计数、会话辅助 |
| 数据库 | MySQL / PostgreSQL | 核心业务数据，配合主从、备份、迁移和连接池 |
| 消息队列 | Kafka / RocketMQ / RabbitMQ | 异步解耦、削峰、最终一致、事件驱动 |
| 配置 | K8s ConfigMap/Secret；或 Apollo/Nacos/Consul KV | 环境配置、密钥、动态开关 |
| 日志 | ELK / Elastic Stack | 日志采集、检索、聚合分析、排障 |
| 指标 | Prometheus + Grafana | QPS、延迟、错误率、资源和依赖指标 |
| 链路追踪 | OpenTelemetry + Jaeger/Tempo/Elastic APM | 跨服务调用链路和慢请求定位 |
| CI/CD | GitLab CI / GitHub Actions / Jenkins + Argo CD | 构建、镜像、部署、回滚、环境一致性 |

## Kratos 在集群里的位置

Kratos 适合放在服务内部标准层：

- `api`：用 proto 定义 HTTP/gRPC 接口契约。
- `service`：处理协议对象和业务对象转换。
- `biz`：业务规则、用例和事务边界。
- `data`：数据库、Redis、MQ、RPC client 等基础设施访问。
- `server`：HTTP/gRPC server、middleware、路由和启动配置。

推荐实践：

- 对内服务优先 gRPC，对外或 BFF 提供 HTTP REST。
- 所有服务统一 middleware：recovery、logging、tracing、metrics、auth、rate limit。
- 业务错误统一错误码，映射到 HTTP status 和 gRPC code。
- Kratos 服务只保留无状态计算，状态放到数据库、缓存、MQ 或对象存储。
- 服务启动时完成配置校验和依赖探活，就绪后再接入流量。

## 注册中心

### Kubernetes 场景

如果服务都部署在 Kubernetes 内，注册中心能力优先使用 K8s 原生 Service 和 DNS：

1. 每个 Kratos 服务用 Deployment 管理多个 Pod。
2. Service 通过 label selector 选择健康 Pod。
3. 调用方通过 `user-service.default.svc.cluster.local` 这类 DNS 名访问服务。
4. readinessProbe 失败的 Pod 不进入 Service endpoint。
5. 滚动发布时旧 Pod 先摘流，新 Pod ready 后再接流量。

优点是简单、原生、运维成本低，不一定需要额外部署 Consul。Kubernetes 自身使用 etcd 存储集群状态，但业务服务通常不直接把 etcd 当注册中心用。

### 非 Kubernetes 或混合部署

如果存在 VM、裸机、多机房、跨 K8s 集群等混合环境，可以考虑 Consul/Nacos 这类注册中心：

- 服务启动后通过 Kratos registry 把实例写入注册中心。
- 注册中心负责健康检查和服务目录。
- 客户端或网关从注册中心拉取健康实例。
- 配合负载均衡、超时、重试和熔断。

选型建议：

- 单 K8s 集群：K8s Service/DNS 足够承担注册中心能力。
- 多集群、多数据中心、VM + K8s 混合：Consul/Nacos 更合适。
- 已经使用 Nacos 做注册/配置中心，或 Apollo 做配置中心的团队，可以沿用现有体系，但要统一健康检查和下线流程。

## 网关和入口层

入口层常见职责：

- TLS 终止。
- 域名和路径路由。
- 身份认证、签名校验、租户识别。
- 限流、黑白名单、WAF。
- 灰度发布、流量镜像、金丝雀。
- 请求日志和 traceId 注入。

常见组合：

- Kubernetes Ingress + Nginx Ingress Controller：简单稳定。
- API Gateway：适合多团队、多租户、插件化鉴权和限流。
- BFF：前端聚合接口和页面场景适配，内部再调用 Kratos 服务。

注意：网关不要承载复杂业务逻辑，复杂规则应该沉到服务层或专门的策略服务。

## 缓存、数据库和消息队列

### Redis 缓存

推荐用法：

- Cache Aside：先查缓存，未命中查数据库，再写缓存。
- 热点数据设置合理 TTL，避免永久缓存造成脏数据。
- 防穿透：缓存空值、布隆过滤器、参数校验。
- 防击穿：互斥锁、singleflight、逻辑过期。
- 防雪崩：TTL 加随机抖动、分批预热、限流降级。
- 分布式锁只用于短时间、低争议场景，强一致协调优先考虑 etcd/ZooKeeper。

部署建议：

- 小规模高可用：Redis Sentinel。
- 大规模容量和吞吐：Redis Cluster。
- 应用侧配置连接池、读写超时、最大重试次数和熔断。

### 数据库

核心业务数据仍建议落 MySQL 或 PostgreSQL：

- MySQL：互联网 OLTP、团队经验成熟、分库分表生态强。
- PostgreSQL：复杂查询、JSONB、GIS、强 SQL 表达和复杂约束。

最佳实践：

- 应用使用连接池，限制最大连接数和空闲连接数。
- 所有 SQL 设置超时，慢查询进入监控。
- DDL 用 migration 工具，不要让服务启动时自动改生产表。
- 核心表设计备份、恢复演练和主从切换预案。
- 读写分离要处理复制延迟，强一致读回主库。

### 消息队列

MQ 适合：

- 下单后异步通知、积分、优惠券、搜索索引同步。
- 秒杀削峰。
- 跨服务最终一致。
- 日志、行为事件、数据同步。

选型：

- Kafka：高吞吐日志流、事件流、大数据生态。
- RocketMQ：事务消息、延迟消息、业务事件可靠投递。
- RabbitMQ：传统队列、路由灵活、规模中等。

注意：消费者要幂等，消息要有重试、死信、告警和补偿任务。

## ELK 日志方案

ELK 通常指 Elasticsearch、Logstash、Kibana。现在生产里也常用 Filebeat / Fluent Bit 采集容器日志，再送到 Logstash 或直接进 Elasticsearch。

推荐链路：

```text
Kratos structured log -> stdout
  -> Filebeat / Fluent Bit DaemonSet
  -> Logstash pipeline（清洗、脱敏、解析）
  -> Elasticsearch index / data stream
  -> Kibana 查询和仪表盘
```

日志规范：

- 输出 JSON 结构化日志。
- 必带字段：`trace_id`、`span_id`、`service`、`env`、`version`、`method`、`path`、`latency`、`status`、`error_code`。
- 错误日志记录堆栈和内部原因，但对外响应脱敏。
- 日志分级：debug、info、warn、error。
- 高频成功日志可采样，错误日志完整保留。
- 日志保留周期和索引生命周期要配置，避免 Elasticsearch 磁盘被打满。

ELK 解决“发生了什么”，但不完全解决“系统现在是否健康”和“请求慢在哪里”。因此要补指标和链路追踪。

## 指标和链路追踪

### Prometheus + Grafana

建议暴露这些指标：

- QPS、P95/P99 延迟、错误率。
- HTTP/gRPC 状态码分布。
- Redis、数据库、MQ 调用耗时和错误率。
- 连接池使用率、排队时间。
- goroutine、内存、GC、CPU。
- Pod 重启次数、HPA 扩缩容、节点资源。

### OpenTelemetry tracing

链路追踪用于跨服务排查：

- 网关注入 traceId。
- Kratos middleware 传递 trace context。
- HTTP/gRPC/DB/Redis/MQ client 自动或手动打 span。
- Jaeger、Tempo 或 Elastic APM 展示调用链。

面试里可以说：日志看细节，指标看趋势和告警，trace 看一次请求跨服务的路径，三者要通过 traceId 打通。

## 发布、扩缩容和稳定性

### Kubernetes 部署建议

每个服务至少包含：

- Deployment：多副本、滚动发布。
- Service：稳定访问入口。
- ConfigMap / Secret：配置和密钥。
- readinessProbe：是否可以接流量。
- livenessProbe：是否需要重启。
- resource requests / limits：资源申请和上限。
- HPA：按 CPU、QPS 或自定义指标扩缩容。
- PodDisruptionBudget：控制主动驱逐时最小可用副本。

### 稳定性策略

- 所有外部调用设置超时。
- 幂等接口才能自动重试，重试要加退避和最大次数。
- 下游异常时熔断、降级、限流。
- 服务下线时先摘流，再 graceful shutdown。
- 配置变更和发布走灰度，保留快速回滚能力。
- 关键业务有补偿任务和对账任务。

## 不同规模的推荐方案

### 小团队 / 初期项目

- Kratos + Docker Compose 或单 K8s 集群。
- K8s Service/DNS 承担注册中心能力。
- Redis 单主从或云 Redis。
- MySQL/PostgreSQL 云数据库。
- Filebeat + Elasticsearch + Kibana，或直接使用云日志服务。
- Prometheus + Grafana 做基础指标。

目标是先把规范建立起来，不要过早引入过多基础设施。

### 中大型微服务

- Kratos + Kubernetes 多副本部署。
- Ingress / API Gateway 统一入口。
- K8s Service/DNS 或 Consul/Nacos 做注册中心。
- Redis Sentinel/Cluster。
- MySQL/PostgreSQL 主从、高可用、备份恢复。
- Kafka/RocketMQ 做异步事件。
- ELK + Prometheus/Grafana + OpenTelemetry。
- GitOps / Argo CD / Jenkins 等完成可回滚发布。

目标是把服务治理、可观测性、扩缩容、故障隔离和数据可靠性都补齐。

### 多机房 / 多云

- 服务分区部署，优先本地调用。
- Consul/Nacos 或服务网格处理跨集群服务目录和流量治理。
- 数据层明确主写区域和容灾策略。
- MQ、缓存、数据库跨地域同步要接受延迟和一致性权衡。
- 灾备演练和降级预案必须常态化。

## 面试追问

### 微服务集群推荐怎么部署？

推荐 Kratos + Kubernetes 作为服务运行底座，HTTP/gRPC 统一接入，K8s Service/DNS 或 Consul/Nacos 做注册中心，Redis 做缓存，MySQL/PostgreSQL 存核心数据，Kafka/RocketMQ 做异步解耦，ELK 做日志检索，Prometheus/Grafana 做指标告警，OpenTelemetry 做链路追踪。这样能覆盖部署、注册中心、数据、缓存、异步、可观测性和稳定性治理。

### 注册中心用 K8s、Consul/Nacos 还是 etcd？

单 Kubernetes 集群优先 K8s Service/DNS，最简单。跨集群、VM + K8s 混合、多数据中心可以考虑 Consul/Nacos。etcd 是 Kubernetes 的核心状态存储，不建议普通业务服务直接依赖 etcd 做注册中心，除非你明确需要强一致协调能力并能承担运维复杂度。

### ELK 是否就等于完整可观测性？

不是。ELK 主要解决日志采集和检索；指标告警更适合 Prometheus/Grafana；跨服务慢请求定位需要 tracing，例如 OpenTelemetry + Jaeger/Tempo/Elastic APM。日志、指标、链路追踪要通过 traceId 关联起来。

### 为什么推荐 Kratos？

Kratos 比 Gin 更偏微服务工程化，适合 HTTP/gRPC 双协议、proto 契约、middleware、错误码、配置、注册中心和分层工程。服务数量多、团队多人协作时，Kratos 的约束能降低长期维护成本。

## 文章导航
上一篇：[02-注册中心-ZooKeeper-Consul-etcd](02-注册中心-ZooKeeper-Consul-etcd.md)
下一篇：[01-高并发秒杀系统](../系统设计/01-高并发秒杀系统.md)