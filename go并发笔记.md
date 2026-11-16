## 1. 心智模型：Go 并发有两个世界

Go 的标语是"不要通过共享内存来通信，而要通过通信来共享内存"。**别把它当教条**——真实工程里两者都用，判断标准是这块状态长什么样：

| 状态形态 | 用什么 | 例子 |
|---|---|---|
| 被很多人同时改的"账本" | `Mutex` / `RWMutex` / `atomic` | 请求计数、session→backend 绑定表、后端健康状态 |
| 需要按顺序流动的数据 | `channel` + `select` | 请求转发、结果汇总、取消信号、优雅关闭 |

还有一个必须分清的概念：**并发 ≠ 并行**。goroutine 是"并发单位"（可以有很多个在逻辑上同时推进），`GOMAXPROCS` 才是"并行度"（同时真正跑 Go 代码的 OS 线程数）。你这台机器是 24。所以"开 1 万个 goroutine"和"用 1 万个核"完全不是一回事。

---

## 2. goroutine：便宜，但会泄漏

**为什么便宜**：初始栈只有 2KB 左右且按需增长，创建/切换/销毁全在用户态，由 runtime 调度到少量线程上。`01_goroutine` 里你实测能看到：1 万个 goroutine 同时活着毫无压力。OS 线程做不到这个量级。

**生命周期有一条铁律**：`main` 返回 = 进程结束，**其他 goroutine 被直接杀死，不会等它们**。这就是为什么你必须有 `WaitGroup` / channel / block 来等待——不是"礼貌"，是正确性。

**头号事故是泄漏**：卡在 channel、锁、网络上的 goroutine 永远不会退出，也不会被 GC 回收。demo 里我让你看 `runtime.NumGoroutine()` 从 1 跳到 6，`close(ch)` 后又回到 1。

排查手段记住两个：`runtime.NumGoroutine()` 做粗对比，`pprof` 的 goroutine profile（`debug=2`）看完整栈。12 月网关写 SSE 转发时，客户端一断开就泄漏一批 goroutine，这是最典型的线上问题。

---

## 3. channel：记住这张状态表就够了

面试最爱考的就是这张表，你把它背下来 + 能解释原因，就超过了大多数人：

| channel 状态 | 发送 | 接收 |
|---|---|---|
| 无缓冲 | 阻塞直到有接收方 | 阻塞直到有发送方 |
| 有缓冲未满 | 不阻塞 | 不阻塞（空则阻塞） |
| 有缓冲已满 | 阻塞 | 不阻塞 |
| **nil** | **永久阻塞** | **永久阻塞** |
| **已关闭** | **panic** | **立刻返回零值，ok=false** |

三个关键理解：

**① 无缓冲 channel 不只是"传数据"，它是同步点。** 发送方和接收方必须同时就绪，这叫 rendezvous（同步握手）。所以无缓冲 channel 天然可以用来做"确认对方准备好了"。

**② 缓冲只改变"发送何时阻塞"，不改变可见性语义。** 很多人误以为无缓冲 channel 有某种"更强的同步保证"——不是。channel 的 happens-before 保证两种都有。

**③ 已关闭的 channel 会立刻返回零值——这是 bug 温床。** 忘了检查 `ok` 的 `for { <-ch }` 在 channel 关闭后会变成 **CPU 空转的死循环**，不是阻塞、不是 panic，就是烧满一个核。

**谁关闭**：**发送方**。接收方永远不要 close 别人的 channel（会 panic，而且是别人背锅）。多个发送方时，用额外的 `done` channel 或 `WaitGroup` 统一关——这正是 `08_patterns` 里 `FanIn` 的做法。

最后记住：**`close` 是广播**。一次 `close(done)` 唤醒所有 `<-done` 的等待者。这就是 `context` 取消的底层形状，也是优雅关闭的常用手法。传信号用 `chan struct{}`（零大小），别用 `chan bool`。

---

## 4. select：多路复用

你之前只写过那个 fibonacci 的例子，那只是个玩具。真正的四个用法：

**① 多个 case 同时就绪 → 随机选。** demo 里 2000 次选择是 999 : 1001。这不是"不确定性"，是**故意设计的公平性**，防止某个分支被饿死。

**② `default` = 非阻塞尝试。** 这是你写限流、丢弃式指标采样、尝试获取名额的抓手。看 demo 里那句"尝试发送：失败"——不阻塞，直接走 default。

**③ 超时用 `time.After`，取消用 `ctx.Done()`。** 注意 `ctx.Done()` 本身就是一个只读 channel，所以它能直接进 select，没有任何魔法。

**④ 把 channel 变量置 `nil`，就"关掉"了这个分支**，因为 nil channel 永远不就绪。这个技巧在状态机里很好用。

**必须记住的反例**：`for + select + default` 是忙等。demo 实测 200ms 空转 **2034 万次**，CPU 100%，什么都没干。正确写法是去掉 `default`，真正阻塞等待。

---

## 5. Mutex / RWMutex：临界区的大小决定吞吐

Mutex 保护的是"读-改-写"这种不是一步完成的操作。三条规则：

**① 临界区只放读写共享状态。** 这是最重要的一条。demo 里的实测很直观：5 个各 100ms 的任务，**锁内做慢操作 = 503ms，锁外做慢操作 = 101ms**。锁里不放 IO、不放 RPC、不放 sleep、不放日志、不放可能阻塞的 channel 操作。

**② 含 Mutex 的结构体只能按指针传递。** 复制（`cc := *c`、按值传参、放进 `map[string]counter`）会把锁也复制一份，两个副本各锁各的，保护彻底失效。`go vet` 会报 `copies lock value`。

**③ RWMutex 只在读多写少时才有收益。** demo 实测：10 个读者各持锁 100ms，`Mutex` 要 1.005s，`RWMutex(RLock)` 只要 101ms。但写频繁时它反而更慢（RLock/RUnlock 本身有开销，写者还可能饥饿）。`RLock` 不可重入——同一个 goroutine 里先 RLock 再 Lock 会自锁死。

**死锁**（`go run ./04_mutex/deadlock` 单独跑，会 `fatal error`，这是预期）：两个 goroutine 以相反顺序获取两把锁。修法最常用也最便宜的是**全局规定加锁顺序**（比如永远按资源 ID 排序）。

这里有个生产相关的认知：runtime 那个 `all goroutines are asleep - deadlock!` 报错**只在所有 goroutine 都阻塞时才出现**。生产环境总有别的 goroutine 在跑，所以死锁的表现是"请求永久挂住、连接不释放、P95 毛刺"，而不是报错。排查要用 goroutine profile，不能等它报。

---

## 6. WaitGroup：Add 的位置就是 correctness

四个方法顺序，背下来：

```
Add(n)  →  go func(){ defer Done() ... }()  →  Wait()
```

**`Add` 必须在 `go` 之前调用**，因为 `Add` 和 `Wait` 之间需要 happens-before 关系。写在 goroutine 里会怎样？跑 `go run ./05_waitgroup/misuse` 看：这次是 `Wait` 提前返回（"完成任务"打在 Wait 之后），但另一种可能是直接 `panic: sync: WaitGroup misuse: Add called concurrently with Wait`。

**顺带说一个 vet 的实用价值**：如果你把 `wg.Add(1)` 直接写进 `go func(){}`，`go vet` 会当场报 `WaitGroup.Add called from inside new goroutine`。这是 vet 里少数能直接抓并发 bug 的检查，养成跑 `go vet` 的习惯。（我那个 demo 用了一层函数变量绕过了 vet，只为让你能观察到运行时行为。）

**收集结果时的一个高价值技巧**：每个 goroutine 写自己的下标，就**不需要任何锁**：

```go
out := make([]int, len(nums))
wg.Add(len(nums))
for i, v := range nums {
    go func() { defer wg.Done(); out[i] = v*v }()  // 只写 out[i]，无竞争
}
wg.Wait()
```

顺便：Go 1.22 起 for 的循环变量是**每轮独立**的，所以上面直接捕获 `i, v` 是安全的。你 `go.mod` 里写的是 `go 1.26.5`，没问题；但如果哪天写 `go < 1.22`，就必须 `go func(i, v int){...}(i, v)` 显式传参——这是老代码里最常见的坑之一。

**选型**：只关心"都结束"→ WaitGroup；要按完成顺序拿结果 / 流式产出 → channel；要"最快的一个，其余取消" → channel + select + context。第三种就是网关的 first-wins / hedged request 模式。

---

## 7. atomic：能做什么，不能做什么

**能用**：单个变量的计数器、标志位、指针整体替换。比 Mutex 快，不会死锁。类型有 `atomic.Int64`、`atomic.Uint64`、`atomic.Bool`、`atomic.Pointer[T]`。

**不能用**：把多步操作合成原子。这是我特意做成**确定性**的一个 demo：

```go
if n.Load() < limit {   // 检查
    runtime.Gosched()   // 中间做点事
    n.Add(1)            // 再改
}
```

期望最多 100，实测 **103 / 114 / 118**（跑三次）。而 **`-race` 完全不报错**——每次访问确实都是原子的，但组合起来逻辑不原子。这是本课最重要的一个认知：

> **race detector 能证明"有竞争"，不能证明"没有竞争"；它更证明不了业务正确性。**

正确做法是 CAS 循环（`CompareAndSwap`，乐观锁），demo 里结果恒为 100。但共享状态一旦超过一两个字段，CAS 循环会迅速变得难读难维护，那时请换回 Mutex。

**两个网关直接相关的点**：

- `atomic.Pointer[T]` 做**配置/后端列表的整体替换**。读者永远拿到一个完整自洽的快照，不会看到"一半旧一半新"。demo 里 4 个读者无锁读了 941 万次，期间后端列表被整体替换，零撕裂。
- 如果直接改被并发读写的 map/slice，会撞上 `fatal error: concurrent map read and map write`。**这个 fatal 不可 recover，直接杀死进程**——Go 里最贵的并发错误。
- **round-robin 负载均衡的全部核心就是 `next.Add(1) - 1` 取模。** 你 12 月网关的 baseline 就这 5 行。

---

## 8. 我看了你 v0.1 的代码，5 个待改点

`go:src/http/main.go` 整体是扎实的（middleware 顺序对、recover 位置对、WaitGroup 用得对）。但有几个点值得现在就知道：

1. **`responseRecorder.WriteHeader` 没防重复调用。** 如果 handler 已经写了 200 再 panic，`recoverPanic` 里的 `http.Error` 会触发 `superfluous response.WriteHeader call`。加个 `wroteHeader bool` 判断。
2. **`logRequest` 的 `slog.Info` 在 `next.ServeHTTP` 之后且没有 `defer`**，万一 panic 逃出 recover 层，整条访问日志会丢。改成 `defer`。
3. **`/count` 用的是全局 `requestCounter`，只统计 `/count` 自己**。如果本意是"所有请求数"，应该挪到 middleware 里。
4. **`http.TimeoutHandler` 会把整个响应体缓冲到内存再一次性写出**（我查了 Go 1.26 源码，内部是 `timeoutWriter.wbuf`）。所以 **SSE 流式转发绝对不能套它**，否则流式变批量、TTFT 直接废掉。网关里应该用 `context.WithTimeout` + 每个后端自己的 client timeout。这条对你 12 月的项目是硬伤，提前知道能省一周。
5. **没设 `ReadHeaderTimeout` / `IdleTimeout`。** 暴露在外的网关必须设，否则慢速请求能拖垮连接池。
