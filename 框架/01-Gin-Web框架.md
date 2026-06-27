# Gin Web 框架

## 核心定位

Gin 是 Go 生态里常见的 HTTP Web 框架，主打轻量、高性能、路由和中间件机制清晰。它适合快速构建 REST API、网关 BFF、管理后台接口、小型 Web 服务，也可以作为微服务的 HTTP 接入层。

面试回答时可以这样定位：Gin 不是“大而全”的企业框架，它更像一个高效 HTTP 工具箱。它负责路由、参数绑定、JSON 响应、中间件、错误处理等 Web 层能力；配置、日志、链路追踪、鉴权、依赖注入、数据库访问等通常需要项目自己组合。

## Gin 常见能力

- 路由分组：按版本、模块、权限分组，例如 `/api/v1/user`。
- 中间件：统一处理日志、恢复、鉴权、限流、跨域、traceId。
- 参数绑定：支持 query、path、form、JSON 等参数绑定。
- 参数校验：结合 binding tag 做基础校验。
- JSON 响应：快速返回结构化响应。
- 静态文件和模板：能做简单 Web 页面，但 Gin 更常用于 API 服务。
- panic recovery：通过 `Recovery` 中间件避免单个请求 panic 打垮进程。

简单示例：

```go
r := gin.New()
r.Use(gin.Logger(), gin.Recovery())

v1 := r.Group("/api/v1")
v1.GET("/users/:id", func(c *gin.Context) {
    id := c.Param("id")
    c.JSON(200, gin.H{"id": id})
})

r.Run(":8080")
```

## 推荐项目分层

Gin 只应该出现在接入层，不建议把业务逻辑塞进 handler。

常见分层：

| 层次 | 作用 |
| --- | --- |
| router | 注册路由和中间件 |
| handler / controller | 解析请求、调用 service、组织响应 |
| service | 业务逻辑、事务边界、领域规则 |
| repository / dao | 数据访问，封装 SQL / ORM |
| middleware | 鉴权、日志、限流、trace、recover |
| response / error | 统一响应结构和错误码 |

推荐原则：

- handler 保持薄，只做参数绑定、校验、调用 service、返回响应。
- service 不依赖 `*gin.Context`，用 `context.Context` 传递超时、traceId、用户信息等。
- repository 不感知 HTTP，方便复用和单元测试。
- 错误在 service 中用业务错误表达，在 handler 或统一中间件里转换成 HTTP 状态码和错误码。

## 最佳实践

### 路由和版本管理

- 用 `Group` 按 API 版本和业务模块组织路由。
- 外部 API 保持稳定，破坏性变更放到新版本路径。
- 管理端、开放接口、内部接口要拆开中间件链路。

```go
api := r.Group("/api")
v1 := api.Group("/v1")
user := v1.Group("/users", AuthMiddleware())
```

### 参数绑定和校验

- JSON 请求使用 `ShouldBindJSON`，query/path 参数分别处理。
- 不要忽略绑定错误，要给出明确错误码。
- 复杂校验放到 service 或 validator，不要堆在 handler。

```go
type CreateUserReq struct {
    Name  string `json:"name" binding:"required"`
    Email string `json:"email" binding:"required,email"`
}
```

### 统一响应和错误处理

建议所有 API 返回统一结构：

```json
{
  "code": 0,
  "message": "ok",
  "data": {}
}
```

注意点：

- HTTP 状态码表达协议层结果，例如 400、401、403、404、500。
- 业务错误码表达业务语义，例如用户不存在、库存不足、重复提交。
- 不要直接把内部错误、SQL 错误、堆栈信息返回给客户端。

### 中间件顺序

常见顺序：

1. request id / trace id。
2. access log。
3. recovery。
4. CORS。
5. timeout。
6. auth。
7. rate limit。
8. business handler。

中间件要短小，避免在中间件里写复杂业务逻辑。

### context 使用

Gin 的 `*gin.Context` 是 Web 层上下文，不要传到 service 和 dao。推荐在 handler 中取出标准 `context.Context`：

```go
ctx := c.Request.Context()
user, err := svc.GetUser(ctx, id)
```

如果要传用户 ID、traceId，可以封装成业务上下文或通过 `context.WithValue` 谨慎传递。

## 优化使用方式

### 性能优化

- 生产环境使用 `gin.ReleaseMode`，减少 debug 日志和额外开销。
- 给 `http.Server` 设置 `ReadTimeout`、`WriteTimeout`、`IdleTimeout`，不要直接只用默认 `Run` 管全部生产配置。
- 大响应尽量分页，避免一次性返回超大 JSON。
- 文件上传限制大小，防止内存和磁盘被打满。
- 日志异步或采样，避免高 QPS 下同步日志成为瓶颈。
- 热路径避免频繁反射、重复 JSON marshal、大对象复制。

示例：

```go
gin.SetMode(gin.ReleaseMode)

srv := &http.Server{
    Addr:         ":8080",
    Handler:      r,
    ReadTimeout:  5 * time.Second,
    WriteTimeout: 10 * time.Second,
    IdleTimeout:  60 * time.Second,
}
```

### 稳定性优化

- 所有外部调用都设置超时，不要让请求无限挂住。
- 使用 recovery 捕获 panic，同时记录 traceId 和关键上下文。
- 接入限流、熔断、降级，保护下游依赖。
- 关闭服务时做 graceful shutdown，停止接收新请求，等待正在处理的请求完成。
- 健康检查区分存活检查和就绪检查，发布时配合负载均衡摘流。

### 安全实践

- 鉴权逻辑统一放中间件，权限校验放到业务层或专门 auth 模块。
- CORS 不等于安全控制，后端仍要做鉴权和权限校验。
- 请求体大小、上传文件类型、频率都要限制。
- 对外错误信息要脱敏。
- 使用 HTTPS，Cookie 注意 `HttpOnly`、`Secure`、`SameSite`。

## 常见坑

- 把所有业务逻辑写在 handler，后期难测试、难复用。
- 在 goroutine 里直接使用 `*gin.Context`，请求结束后可能出现数据竞争或上下文失效；需要复制必要值。
- 忘记设置超时，慢请求拖垮 worker、连接和下游资源。
- 中间件顺序混乱，导致日志缺 traceId、鉴权绕过或 panic 未恢复。
- 直接返回内部错误，泄露数据库结构或系统信息。
- 滥用全局变量保存请求状态，造成并发安全问题。

## 面试追问

### Gin 适合什么场景？

Gin 适合 Go HTTP API、BFF、管理后台、轻量 Web 服务和微服务 HTTP 接入层。它轻量高效，但不是完整企业框架，日志、配置、链路追踪、鉴权、数据库访问等需要项目自己组合。

### Gin 项目怎么分层？

handler 负责 HTTP 参数和响应，service 负责业务逻辑和事务边界，repository 负责数据访问。`*gin.Context` 不要向下传到 service 和 dao，业务层使用标准 `context.Context`。

### Gin 怎么做性能和稳定性优化？

生产环境开启 release mode，配置 HTTP server 超时，使用 recovery、限流、超时、日志和 trace 中间件；大请求要限制体积，大列表要分页，发布下线要 graceful shutdown。

## 文章导航
上一篇：[06-场景应用问题整合](../Golang/06-场景应用问题整合.md)
下一篇：[02-Kratos微服务框架](02-Kratos微服务框架.md)