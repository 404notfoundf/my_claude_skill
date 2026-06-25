---
name: go-memory-quick-check
description: 快速摸排 Go 项目内存泄漏和内存上涨的高风险代码点。当用户说"检查内存""有没有内存泄露""Go 项目内存风险"或想结构性排查内存问题时触发。适用于无 pprof 的早期排查阶段。
---

# Go Memory Quick Check

## 概览

用最少步骤快速找出 Go 代码里最可能导致内存上涨的问题点。这个 skill 不负责严格证明泄露，只负责先把高风险位置筛出来。

## 快速流程

### 0. 快速诊断（可选）

在深入代码之前，先跑一个快速命令确认问题是否存在：

```bash
# 统计 goroutine 创建点数量（数量越多风险越高）
grep -rn "go func(" --include="*.go" | wc -l

# 统计全局 map 使用（可能无界增长）
grep -rn "var.*map\[" --include="*.go" | wc -l

# 如果能运行测试，观察内存趋势
go test -bench=. -benchmem -count=3 ./...
```

如果数字明显异常（如 goroutine 创建点 > 50，或 map 使用 > 30），直接进入深度排查。

### 1. 找长生命周期状态

优先看包级变量、单例 service、后台任务、常驻 manager、全局 cache、全局 map。

### 2. 特征码敏感搜索

重点扫描并审查以下高频高危模式：

- `go func(` — 特别是循环或请求上下文中开启的、缺乏退出机制的协程
- `time.NewTicker(` — 检查是否漏掉 `Stop()`
- `for ... { select { case <-time.After(` — 循环内频繁创建 Timer 的死穴
- `make(chan ` / `chan ` — 是否存在无缓冲且可能阻塞写、或者无消费的通道
- `map[` / `sync.Map` — 是否存在只增不减、或按业务实体线性膨胀的 Key
- `io.ReadAll(` / `bytes.Buffer` — 处理大文件或 HTTP 大 Body 时是否一次性加载
- `append(` — 是否存在大 Slice 的高频扩容，或从大 Slice 切出小 Slice 导致底层大数组无法释放
- `fmt.Sprintf` 在循环中 — 每次调用都生成新字符串，高频循环下内存持续增长
- `context.WithCancel` — 是否有对应的 `cancel()` 调用，否则 context 泄露
- `sync.Pool` — 放入的对象是否可能被意外扩容并常驻

### 3. 按四类经典根因归类

- **Goroutine 泄露：** 协程因 Channel 阻塞、死锁或 Context 未 Done 导致无法退出。
- **无界常驻集合：** Map/Cache 缺乏 TTL、LRU 或主动 Delete 机制，导致内存随时间只增不减。
- **时间对象残留：** Ticker 未释放，或在 `for` 循环内滥用 `time.After` 导致创建大量临时定时器。
- **长生命周期持有大对象：** Reslice 导致底层大数组无法被 GC；或者是大 Buffer 放入 `sync.Pool` 后被意外扩容并常驻。

### 4. 按严重度标记

对每个发现的风险点，按以下标准评级：

- **P0（高危）**：无界 map/cache 无 TTL、goroutine 无退出条件、循环内 time.After、大对象进全局状态
- **P1（中危）**：Ticker 未 Stop、channel 只有生产无消费、append 导致底层数组无法释放、context.WithCancel 未 cancel
- **P2（低危）**：fmt.Sprintf 在循环中（可优化）、sync.Pool 对象偏大、大 Buffer 未复用

即使还没有 runtime 证据，只要满足以下任一条件，直接标成 P0：
- map 没有上限
- cache 没有 TTL 或淘汰机制
- goroutine 没有退出条件
- ticker 创建后没有 `Stop`
- channel 可能只有生产没有消费
- 大对象被放进全局状态或长生命周期对象

### 5. 输出结果

按危害程度倒序（P0 → P1 → P2），精选最可疑的 **3 到 5 个**核心风险点。每个点必须输出：

- **位置：** `文件名:行号` 或函数名
- **根因：** 为什么这里危险（用 1-2 句话解释机制）
- **性质：** 属于"确定性的真实泄露"还是"高概率的隐式内存上涨风险"
- **修复方向：** 简要建议怎么改（如"加 LRU 淘汰""给 channel 加 buffer 或消费者""defer ticker.Stop()"）

最后给出是否需要跟进抓取 pprof 验证的明确判断。

## 危险模式示例

### Goroutine 泄露

```go
// ❌ 危险：没有退出机制，channel 无消费者时永久阻塞
func processEvents(ch chan Event) {
    for {
        ev := <-ch  // 如果 ch 没有发送者，这里永久阻塞
        handle(ev)
    }
}

// ✅ 安全：使用 context 控制退出
func processEvents(ctx context.Context, ch chan Event) {
    for {
        select {
        case <-ctx.Done():
            return
        case ev := <-ch:
            handle(ev)
        }
    }
}
```

### 无界 Map

```go
// ❌ 危险：按 wallet 地址线性增长，永不清理
var seenTxs = make(map[string]bool)

func OnTx(txHash string) {
    seenTxs[txHash] = true  // 只增不减
}

// ✅ 安全：使用带 TTL 的缓存或定期清理
var seenTxs = cache.New(5*time.Minute, 10*time.Minute)

func OnTx(txHash string) {
    seenTxs.Set(txHash, true, cache.DefaultExpiration)
}
```

### 循环内 time.After

```go
// ❌ 危险：每次循环创建新 Timer，旧 Timer 在触发前不会被 GC
for {
    select {
    case <-time.After(5 * time.Second):
        doSomething()
    }
}

// ✅ 安全：复用同一个 Ticker
ticker := time.NewTicker(5 * time.Second)
defer ticker.Stop()
for range ticker.C {
    doSomething()
}
```

### 大对象 Reslice

```go
// ❌ 危险：大 slice 切出小 slice，底层 1GB 数组无法释放
big := make([]byte, 1<<30) // 1GB
small := big[:100]          // small 引用底层数组，big 无法 GC

// ✅ 安全：拷贝到新 slice
small := make([]byte, 100)
copy(small, big[:100])
```

## 判断规则

- 如果发现无界 map、无界 cache、未停止 ticker、未退出 goroutine，要明确提醒。
- 如果某个 map、cache、队列或对象集合明显可能随着 wallet、address、request id、tx hash、timestamp 之类唯一 key 增长，要直接标成重点风险。
- 用户只需"梳理排查、摸底代码"时，优先使用此方案，不要一上来就强制索要完整的 pprof 报告。
- 一旦用户能够提供 pprof (heap)、runtime 指标或线上 Grafana 内存趋势图，立即切换到更严谨的定量定位方案。

## 输出要求

输出尽量精简，直击痛点：
1. 精选出最可疑的 **3 到 5 个**核心风险点。
2. 每个点标明 P0/P1/P2 严重度和风险类型。
3. 给出是否需要跟进抓取 pprof 验证的明确判断。
4. 如果用户要求保存，输出到 `docs/working_doc/memory/YYYYMMDD_memory_risk.md`。
