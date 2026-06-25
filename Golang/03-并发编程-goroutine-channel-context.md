# 并发编程：goroutine、channel、context

## 并发与并行

- 并发：同时处理多个任务的能力，强调任务结构。
- 并行：多个任务在同一时刻真正同时执行，依赖多核 CPU。

Go 通过 goroutine 和 runtime scheduler 让并发编程成本较低，但并发程序仍然要处理数据竞争、资源释放、超时、取消和错误传播。

## goroutine

使用 `go` 关键字启动 goroutine：

```go
go func() {
    fmt.Println("run in goroutine")
}()
```

特点：

- 初始栈很小，可按需增长。
- 由 Go runtime 调度，不是一启动就对应一个 OS 线程。
- 适合 I/O 密集型和大量轻量任务。

注意点：

- goroutine 泄漏是 Go 服务常见问题。
- main goroutine 退出后，进程会结束，其他 goroutine 不会自动等待。
- goroutine 中 panic 如果没有 recover，会导致整个进程崩溃。

## channel

channel 用于 goroutine 间通信和同步：

```go
ch := make(chan int)

go func() {
    ch <- 1
}()

v := <-ch
fmt.Println(v)
```

### 无缓冲 channel

```go
ch := make(chan int)
```

发送和接收必须同时准备好，否则会阻塞。适合强同步场景。

### 有缓冲 channel

```go
ch := make(chan int, 10)
```

缓冲未满时发送不阻塞，缓冲非空时接收不阻塞。适合削峰、任务队列、并发控制。

## channel 关闭

关闭 channel 表示不会再发送数据：

```go
close(ch)
```

读取关闭后的 channel：

```go
v, ok := <-ch
if !ok {
    // channel 已关闭且数据已读完
}
```

规则：

- 只有发送方应该关闭 channel。
- 向已关闭的 channel 发送会 panic。
- 重复关闭 channel 会 panic。
- 接收方不需要关闭 channel。

## select

`select` 可同时等待多个 channel 操作：

```go
select {
case v := <-ch1:
    fmt.Println(v)
case ch2 <- 10:
    fmt.Println("sent")
case <-time.After(time.Second):
    fmt.Println("timeout")
}
```

常见用途：

- 多路复用。
- 超时控制。
- 取消信号监听。
- 非阻塞发送/接收。

非阻塞示例：

```go
select {
case v := <-ch:
    fmt.Println(v)
default:
    fmt.Println("no data")
}
```

## sync.WaitGroup

等待多个 goroutine 完成：

```go
var wg sync.WaitGroup

for i := 0; i < 10; i++ {
    i := i
    wg.Add(1)
    go func() {
        defer wg.Done()
        fmt.Println(i)
    }()
}

wg.Wait()
```

注意：

- `Add` 应在启动 goroutine 前调用。
- `Done` 通常用 defer 保证执行。
- WaitGroup 只负责等待，不负责错误收集和取消。

## Mutex / RWMutex

互斥锁保护共享资源：

```go
var mu sync.Mutex
var count int

mu.Lock()
count++
mu.Unlock()
```

读写锁适合读多写少：

```go
var rw sync.RWMutex
```

注意点：

- 加锁和解锁要成对出现。
- 不要复制已经使用过的锁。
- 锁保护的是临界区，不是变量名本身。
- 锁粒度过大会影响并发，过小会增加复杂度。

## sync.Once

确保某段逻辑只执行一次：

```go
var once sync.Once

func Init() {
    once.Do(func() {
        fmt.Println("init")
    })
}
```

常用于单例初始化、配置加载、连接池初始化。

注意点：

- `Do` 里的函数如果 panic，这次调用仍会被认为已经执行过，后续不会自动重试。
- 不要复制已经使用过的 `sync.Once`，和锁一样，复制会破坏内部状态。

## sync.Pool

`sync.Pool` 用于临时对象复用，减轻 GC 压力：

```go
var bufPool = sync.Pool{
    New: func() any {
        return new(bytes.Buffer)
    },
}
```

注意：

- Pool 中对象可能随时被 GC 清理。
- 不适合保存业务状态。
- 放回 Pool 前要重置对象。

## context.Context

`context` 用于传递取消信号、超时、截止时间和请求范围值：

```go
ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
defer cancel()

select {
case <-doWork(ctx):
case <-ctx.Done():
    return ctx.Err()
}
```

使用原则：

- 函数的第一个参数通常是 `ctx context.Context`。
- 不要把 context 存到 struct 中长期持有。
- 派生出的 cancel 一定要调用，避免资源泄漏。
- context value 只放请求范围元数据，不放可选参数和核心业务对象。

## errgroup

标准库没有内置 errgroup，但工程中常用 `golang.org/x/sync/errgroup` 来管理并发任务的错误和取消。

典型思路：

- 任一 goroutine 返回错误，取消其他 goroutine。
- 等待所有 goroutine 退出。
- 返回第一个错误。

面试中可以说：WaitGroup 只等待，errgroup 更适合“并发任务 + 错误传播 + 取消”的组合场景。

## 数据竞争

数据竞争指多个 goroutine 同时访问同一份共享数据，其中至少一个是写操作，并且这些访问之间没有同步保护。

常见风险：

- 结果不确定，例如多个 goroutine 同时 `count++`，最终计数可能偏小。
- 普通 `map` 并发读写不安全，严重时会 panic，例如 `fatal error: concurrent map writes`。
- 问题不一定每次复现，容易在线上变成偶发 bug。

检测方式：

```bash
go test -race ./...
go run -race main.go
```

注意：`-race` 适合开发和测试环境排查问题，会带来额外性能开销，一般不直接用于线上正式运行。

避免方式：

- 不共享内存，通过 channel 传递数据或转移所有权。
- 必须共享时，用 `sync.Mutex`、`sync.RWMutex`、`sync/atomic` 等同步机制保护。
- 简单计数器、状态位可以考虑 atomic，复杂业务状态优先用锁。
- 普通 `map` 并发访问要用锁保护；特殊读多写少场景可以考虑 `sync.Map`。
- 明确变量所有权，避免多个 goroutine 随意读写同一份状态。

### sync.Mutex 和 sync.RWMutex 区别

`sync.Mutex` 是互斥锁，同一时间只允许一个 goroutine 进入临界区，读和写都会互斥：

```go
var mu sync.Mutex
var count int

mu.Lock()
count++
mu.Unlock()
```

`sync.RWMutex` 是读写锁，允许多个读操作同时进行，但写操作会独占锁：

```go
var rw sync.RWMutex
var count int

rw.RLock()
v := count
rw.RUnlock()

rw.Lock()
count++
rw.Unlock()
```

核心区别：

| 类型 | 加锁方式 | 特点 | 适合场景 |
| --- | --- | --- | --- |
| `sync.Mutex` | `Lock` / `Unlock` | 读写都互斥，语义简单 | 临界区短、读写都比较频繁、业务逻辑复杂 |
| `sync.RWMutex` | 读：`RLock` / `RUnlock`；写：`Lock` / `Unlock` | 多个读可以并发，写会阻塞所有读写 | 读多写少、读操作耗时或并发读很多 |

使用注意：

- 两者都可以解决数据竞争，区别主要是并发性能和复杂度。
- 不确定时优先用 `Mutex`，代码更简单，不容易误用。
- `RWMutex` 只有在读明显多于写时才更有价值；写很多时收益不大，甚至可能因为锁管理成本更高而更慢。
- 加锁和解锁要成对出现，常用 `defer` 保证释放锁。
- 不要复制已经使用过的锁，也不要复制包含锁的结构体。

### 普通 map 的并发访问

普通 `map` 不是并发安全的。只要有并发写，或者读写同时发生，都应该加锁保护：

```go
type SafeMap struct {
    mu sync.RWMutex
    m  map[string]int
}

func NewSafeMap() *SafeMap {
    return &SafeMap{
        m: make(map[string]int),
    }
}

func (s *SafeMap) Get(key string) (int, bool) {
    s.mu.RLock()
    defer s.mu.RUnlock()

    v, ok := s.m[key]
    return v, ok
}

func (s *SafeMap) Set(key string, value int) {
    s.mu.Lock()
    defer s.mu.Unlock()

    s.m[key] = value
}
```

要点：

- 锁保护的是访问共享 `map` 的临界区，而不是 `map` 变量名本身。
- 如果多个操作要保持一致性，例如“先判断再写入”“批量更新”，要放在同一把锁的同一个临界区里。
- 如果把内部 `map` 直接返回给外部，外部绕过锁访问，仍然会产生数据竞争。

### sync.Map

Go 标准库提供了并发安全的 Map 类型，名字是 `sync.Map`，注意 `Map` 的 `M` 是大写：

```go
var m sync.Map

m.Store("name", "go")
v, ok := m.Load("name")
m.Delete("name")
```

常用方法：

- `Store(key, value)`：写入。
- `Load(key)`：读取。
- `LoadOrStore(key, value)`：如果 key 存在就读取，否则写入。
- `Delete(key)`：删除。
- `Range(func(key, value any) bool)`：遍历，返回 `false` 停止。

适用场景：

- 读多写少。
- key 相对稳定，写入后会被多次读取。
- 缓存、注册表、只写一次多读等场景。

不适合场景：

- 需要强类型 API，因为 `sync.Map` 的 key 和 value 都是 `any`。
- 需要维护复杂业务不变量，例如多个字段或多个 key 必须一起更新。
- 需要准确 `len`、批量一致性、复杂遍历逻辑。

面试回答可以说：Go 的普通 `map` 不是并发安全的，并发读写会产生数据竞争。常规业务优先用 `map + Mutex/RWMutex`，类型清晰、逻辑可控；`sync.Map` 是标准库提供的并发安全 Map，适合读多写少、key 稳定的特殊场景，但不是普通 `map` 的默认替代品。

## 原子操作

`sync/atomic` 适合简单计数器、状态位等低层场景：

```go
var n atomic.Int64
n.Add(1)
fmt.Println(n.Load())
```

注意：

- atomic 只能解决单变量原子读写，不适合复杂不变量。
- 复杂业务状态优先用锁，代码更清楚。

## 常见并发模式

### Worker Pool

```go
jobs := make(chan int)
var wg sync.WaitGroup

for i := 0; i < 5; i++ {
    wg.Add(1)
    go func() {
        defer wg.Done()
        for job := range jobs {
            fmt.Println(job)
        }
    }()
}

for i := 0; i < 100; i++ {
    jobs <- i
}
close(jobs)
wg.Wait()
```

### Fan-in

多个输入合并成一个输出。

### Fan-out

一个输入分发给多个 worker 并行处理。

### Pipeline

多个处理阶段通过 channel 串联。

## 面试追问

### channel 和 mutex 怎么选？

如果重点是所有权转移、任务通知、流水线，用 channel 更自然。如果重点是保护共享状态，用 mutex 更直接。不要为了“Go 风格”强行用 channel 替代简单锁。

### goroutine 泄漏有哪些场景？

- channel 永远没人接收或发送。
- 没有监听 context 取消。
- 后台 ticker 没有 Stop。
- worker 等待任务但任务通道永不关闭。
- 网络请求或数据库调用没有超时。

### 如何优雅关闭服务？

- 接收系统信号。
- 停止接收新请求。
- 通过 context 通知后台任务退出。
- 等待正在处理的请求完成。
- 设置最大等待时间，超时后强制退出。

## 文章导航
上一篇：[02-类型系统-接口-错误处理](02-类型系统-接口-错误处理.md)
下一篇：[04-GMP调度-GC-内存模型](04-GMP调度-GC-内存模型.md)
