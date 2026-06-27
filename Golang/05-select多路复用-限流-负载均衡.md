# select 多路复用、限流与负载均衡

## Go 中的多路复用

Go 的多路复用通常指用 `select` 同时监听多个 channel 事件：

```go
select {
case msg := <-ch1:
    fmt.Println("from ch1:", msg)
case msg := <-ch2:
    fmt.Println("from ch2:", msg)
case <-ctx.Done():
    return ctx.Err()
}
```

它适合处理：

- 多个数据源合并。
- 超时和取消。
- 非阻塞 channel 操作。
- fan-in/fan-out。
- 后台 worker 调度。

## select 的特性

- 如果多个 case 同时就绪，Go 会伪随机选择一个。
- 如果没有 case 就绪且没有 default，会阻塞。
- 如果有 default 且没有其他 case 就绪，会立即执行 default。
- 对 nil channel 的发送和接收会永久阻塞，可用于动态开关 case。

动态禁用 case：

```go
var ch <-chan int
if enabled {
    ch = source
}

select {
case v := <-ch:
    fmt.Println(v)
case <-ctx.Done():
    return
}
```

## 超时控制

简单写法：

```go
select {
case v := <-result:
    return v, nil
case <-time.After(time.Second):
    return 0, errors.New("timeout")
}
```

高频路径建议复用 timer，避免大量 `time.After` 创建临时定时器。

## Fan-in

多个 channel 合并为一个 channel：

```go
func merge(ctx context.Context, inputs ...<-chan int) <-chan int {
    out := make(chan int)
    var wg sync.WaitGroup

    for _, ch := range inputs {
        ch := ch
        wg.Add(1)
        go func() {
            defer wg.Done()
            for {
                select {
                case v, ok := <-ch:
                    if !ok {
                        return
                    }
                    select {
                    case out <- v:
                    case <-ctx.Done():
                        return
                    }
                case <-ctx.Done():
                    return
                }
            }
        }()
    }

    go func() {
        wg.Wait()
        close(out)
    }()

    return out
}
```

## Fan-out / Worker Pool

一个任务队列由多个 worker 消费：

```go
func startWorkers(ctx context.Context, n int, jobs <-chan Job) {
    var wg sync.WaitGroup
    for i := 0; i < n; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            for {
                select {
                case job, ok := <-jobs:
                    if !ok {
                        return
                    }
                    handle(job)
                case <-ctx.Done():
                    return
                }
            }
        }(i)
    }
    wg.Wait()
}
```

## 使用 channel 控制并发数

旧资料中“大量数据优化分页”实际主要讲的是用 `WaitGroup + buffered channel` 控制 goroutine 并发，这里归到 Go 并发控制。

```go
func run(tasks []Task, maxConcurrent int) error {
    sem := make(chan struct{}, maxConcurrent)
    var wg sync.WaitGroup
    var once sync.Once
    var firstErr error

    for _, task := range tasks {
        task := task
        sem <- struct{}{}
        wg.Add(1)

        go func() {
            defer wg.Done()
            defer func() { <-sem }()

            if err := handle(task); err != nil {
                once.Do(func() {
                    firstErr = err
                })
            }
        }()
    }

    wg.Wait()
    return firstErr
}
```

注意：

- 获取令牌和释放令牌要成对。
- goroutine 内要 recover 或返回错误，避免整个进程崩溃。
- 如果需要失败后取消其他任务，使用 context 或 errgroup 更合适。

## HTTP 请求限流

### 并发数限制

```go
func LimitMiddleware(max int, next http.Handler) http.Handler {
    sem := make(chan struct{}, max)

    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        select {
        case sem <- struct{}{}:
            defer func() { <-sem }()
            next.ServeHTTP(w, r)
        default:
            http.Error(w, "too many requests", http.StatusTooManyRequests)
        }
    })
}
```

这限制的是“同时处理的请求数”，不是每秒请求数。

### QPS 限制

QPS 限制通常使用令牌桶或漏桶算法。Go 官方扩展包 `golang.org/x/time/rate` 常用于单机限流：

```go
limiter := rate.NewLimiter(rate.Limit(100), 200)

if !limiter.Allow() {
    http.Error(w, "rate limited", http.StatusTooManyRequests)
    return
}
```

分布式限流通常放在 Redis、API 网关或专门的限流服务中实现。

## 负载均衡

Go 里用 goroutine/channel 可以做应用内任务分发，但不要把它和 Nginx、网关、注册中心的负载均衡混淆。

应用内负载均衡适合：

- 多 worker 处理任务。
- 多下游连接选择。
- 多消费者消费队列。

常见策略：

- Round-robin：轮询。
- Random：随机。
- Least connections：最少连接。
- Weighted：权重。
- Consistent hash：一致性哈希，适合需要会话或 key 粘性的场景。

## channel 实现简易 worker 负载均衡

```go
type Job struct {
    ID int
}

func worker(id int, jobs <-chan Job) {
    for job := range jobs {
        fmt.Printf("worker %d handles job %d\n", id, job.ID)
    }
}

func main() {
    jobs := make(chan Job, 100)

    for i := 0; i < 5; i++ {
        go worker(i, jobs)
    }

    for i := 0; i < 100; i++ {
        jobs <- Job{ID: i}
    }
    close(jobs)
}
```

这个模型由 worker 竞争同一个任务队列，天然实现了简单负载分摊。处理快的 worker 会拿到更多任务。

## 易错点

- channel 限流控制的是并发数，不等于 QPS。
- select 的 default 会让逻辑变成非阻塞，使用不当会造成空转。
- `time.After` 在高频循环里可能带来额外分配。
- worker pool 要考虑关闭任务队列和等待 worker 退出。
- 负载均衡不是只有代码层，通常还包括 Nginx、网关、注册中心、客户端负载均衡。

## 文章导航
上一篇：[04-GMP调度-GC-内存模型](04-GMP调度-GC-内存模型.md)
下一篇：[06-场景应用问题整合](06-场景应用问题整合.md)
