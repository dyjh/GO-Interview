# Kratos 微服务框架

## 核心定位

Kratos 是 Go 生态里的微服务框架，常用于构建较规范的服务端工程。相比 Gin 更偏 HTTP Web 接入层，Kratos 更强调工程化和微服务体系：项目布局、配置、日志、HTTP/gRPC transport、中间件、注册中心、错误处理、代码生成等。

这里的 Kratos 指 go-kratos 项目。如果面试里听到 “Kartos”，大概率是对 Kratos 的误写或误读，可以先确认对方语境。

一句话：Gin 更轻，适合快速做 HTTP API；Kratos 更工程化，适合中大型微服务、HTTP + gRPC 双协议、统一治理和团队规范。

## Kratos 常见能力

- 项目脚手架：生成标准目录结构。
- Transport：同时支持 HTTP 和 gRPC 接入。
- Middleware：统一处理 logging、recovery、tracing、metrics、auth 等横切逻辑。
- Config：支持配置加载和监听。
- Registry：可接入注册中心。
- Errors：统一错误码和错误响应。
- Protobuf：常用 proto 定义 API，再生成 HTTP/gRPC 代码。
- Wire：常配合依赖注入组织组件初始化。

Kratos 常见目录风格：

```text
api/        proto 和生成代码
cmd/        服务入口
configs/    配置文件
internal/
  biz/      业务领域逻辑
  data/     数据访问和外部依赖
  server/   HTTP/gRPC server 初始化
  service/  API 编排层
```

## 分层思想

Kratos 推荐把业务拆成更清晰的层：

| 层次 | 作用 |
| --- | --- |
| api | 定义接口协议，通常是 protobuf |
| service | 接收 transport 请求，做参数转换和调用 biz |
| biz | 业务规则、领域模型、用例编排 |
| data | 数据库、缓存、MQ、第三方服务访问 |
| server | HTTP/gRPC server、middleware、路由注册 |
| conf | 配置结构和配置加载 |

关键点：

- service 层不要写复杂业务，只做协议对象和领域对象转换。
- biz 层不要依赖具体数据库、HTTP 框架或 gRPC 框架。
- data 层实现 biz 层定义的 repository 接口。
- 依赖方向保持从外到内，业务核心不要被基础设施绑死。

## 最佳实践

### API 先行

Kratos 项目常见做法是先写 proto：

```proto
service UserService {
  rpc GetUser (GetUserRequest) returns (GetUserReply) {
    option (google.api.http) = {
      get: "/v1/users/{id}"
    };
  }
}
```

优势：

- HTTP 和 gRPC 可以共用一套接口定义。
- 客户端和服务端代码可以生成，减少手写样板代码。
- 接口变更更清晰，适合团队协作和版本管理。

注意：proto 是接口契约，不要把数据库表结构直接暴露成 API 结构。

### biz 和 data 解耦

biz 层定义接口，data 层实现：

```go
type UserRepo interface {
    Get(ctx context.Context, id int64) (*User, error)
}

type UserUsecase struct {
    repo UserRepo
}
```

这样业务层可以独立测试，也可以替换 MySQL、PostgreSQL、Redis、RPC 客户端等实现。

### 错误码和错误处理

- 业务错误要有稳定错误码，例如 `USER_NOT_FOUND`、`ORDER_STATUS_INVALID`。
- 内部错误不要直接暴露给客户端。
- HTTP 状态码、gRPC code 和业务错误码要有映射关系。
- 日志里记录内部细节，响应里给出可理解但不泄露实现的信息。

### 中间件治理

常见中间件：

- recovery：防止 panic 打垮服务。
- logging：记录请求、耗时、错误、traceId。
- tracing：链路追踪。
- metrics：暴露延迟、QPS、错误率。
- auth：认证和权限信息解析。
- validate：参数校验。
- rate limit：保护服务和下游。

中间件只做横切能力，不要承载具体业务流程。

### 配置管理

- 配置按环境区分：dev、test、prod。
- 敏感信息不要提交到仓库，用环境变量、密钥管理或配置中心。
- 配置结构要有默认值和校验，启动时尽早失败。
- 配置热更新只适合部分开关类参数，数据库连接、监听端口等核心配置要谨慎热更。

## 优化使用方式

### 性能优化

- HTTP/gRPC server 都要设置合理超时，避免慢请求长期占用资源。
- 数据库、Redis、RPC 客户端都配置连接池、超时和最大并发。
- 高频接口避免在 service 层做大量对象转换和重复序列化。
- 大列表接口分页，批量接口限制 batch size。
- 对读多写少数据使用缓存，但要设计失效、穿透、击穿和一致性策略。
- 对下游调用使用超时、重试、熔断和限流，避免故障扩散。

### 工程优化

- 统一 Makefile 或任务脚本，固定 proto 生成、lint、test、build 流程。
- proto 生成代码不要手改，业务代码放在 service / biz / data。
- 用接口隔离外部依赖，让 biz 层单元测试可以 mock repo。
- 对数据库迁移使用 migration 工具，不要依赖启动时自动改生产表。
- 每个服务提供 health check、metrics、pprof 或等价诊断能力。

### 可观测性优化

- 每个请求带 traceId，跨 HTTP/gRPC/RPC 传递。
- 日志按结构化字段输出，例如 service、method、trace_id、user_id、latency、error_code。
- 指标至少关注 QPS、P95/P99 延迟、错误率、依赖耗时、连接池使用率。
- 告警不要只看实例存活，也要看业务错误率和下游依赖异常。

### 发布和稳定性

- 服务启动时先完成配置校验、依赖初始化和健康检查，再标记 ready。
- 下线时先摘流，再 graceful shutdown，等待请求处理完成。
- 对 gRPC streaming、WebSocket 等长连接场景，要有心跳和最大连接生命周期。
- 发布失败要能快速回滚，数据库变更要兼容旧代码和新代码。

## Gin 和 Kratos 怎么选

| 场景 | 更推荐 |
| --- | --- |
| 简单 REST API、BFF、管理后台 | Gin |
| 小团队快速迭代，框架约束少 | Gin |
| HTTP + gRPC 双协议服务 | Kratos |
| 多服务协作，需要统一工程规范 | Kratos |
| 需要服务治理、注册中心、链路追踪、metrics 体系 | Kratos |
| 学习成本和接入成本敏感 | Gin |

选型建议：

- 只是做 HTTP API，不需要复杂治理，Gin 简洁直接。
- 服务数量多、团队多人协作、需要统一 proto、gRPC、注册中心和可观测性，Kratos 更合适。
- 也可以 Gin 做边缘 BFF，Kratos 做内部核心微服务。

## 常见坑

- 把 Kratos 当成 Gin 用，只写 HTTP handler，不利用 api / service / biz / data 分层。
- proto 设计过度贴近数据库表，导致接口难演进。
- biz 层直接依赖 GORM、Redis 客户端或 HTTP client，破坏可测试性。
- 所有错误都返回 internal error，客户端无法区分业务失败和系统失败。
- 依赖注入链路复杂但缺少启动校验，运行时才发现 nil 或配置缺失。
- 微服务拆得太细，服务治理成本超过业务收益。

## 面试追问

### Kratos 适合什么场景？

Kratos 适合中大型 Go 微服务，尤其是需要 HTTP/gRPC 双协议、proto 契约、统一中间件、注册中心、链路追踪、metrics 和工程规范的团队。简单 HTTP API 用 Gin 可能更轻。

### Kratos 的分层怎么理解？

api 定义接口契约，service 做协议转换和调用 biz，biz 放业务规则和用例，data 负责数据库、缓存、MQ、RPC 等基础设施。核心原则是业务层不要依赖具体框架和存储实现。

### Kratos 怎么做优化？

性能上配置 HTTP/gRPC 超时、连接池、缓存、分页和批量上限；稳定性上接入 recovery、限流、熔断、graceful shutdown；工程上统一 proto 生成、错误码、日志、trace、metrics 和 migration 流程。

## 文章导航
上一篇：[01-Gin-Web框架](01-Gin-Web框架.md)
下一篇：[01-MySQL基础-事务-锁](../数据库/01-MySQL基础-事务-锁.md)