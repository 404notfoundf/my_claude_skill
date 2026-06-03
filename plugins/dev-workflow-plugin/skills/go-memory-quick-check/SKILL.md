---
name: go-memory-quick-check
description: 快速摸排 Go 项目里可能导致内存泄露或内存持续上涨的高风险代码点。适用于用户想先快速梳理风险位置、暂时没有 pprof、只想做结构性排查，或希望在正式深挖前先列出最可疑模块的场景。
---

# Go Memory Quick Check

## 概览

用最少步骤快速找出 Go 代码里最可能导致内存上涨的问题点。这个 skill 不负责严格证明泄露，只负责先把高风险位置筛出来。

## 快速流程

1. 先找长生命周期状态。
   优先看包级变量、单例 service、后台任务、常驻 manager、全局 cache、全局 map。

2. **特征码敏感搜索：**
   重点扫描并审查以下高频高危模式：
   - `go func(` (特别是循环或请求上下文中开启的、缺乏退出机制的协程)
   - `time.NewTicker(` (检查是否漏掉 `Stop()`)
   - `for ... { select { case <-time.After(` (循环内频繁创建 Timer 的死穴)
   - `make(chan ` / `chan ` (是否存在无缓冲且可能阻塞写、或者无消费的通道)
   - `map[` / `sync.Map` (是否存在只增不减、或按业务实体线性膨胀的 Key)
   - `io.ReadAll(` / `bytes.Buffer` (处理大文件、区块链大日志或 HTTP 大 Body 时是否一次性加载)
   - `append(` (是否存在大 Slice 的高频扩容，或从大 Slice 切出小 Slice 导致底层大数组无法释放)

3. **按四类经典根因归类：**
   - **Goroutine 泄露：** 协程因 Channel 阻塞、死锁或 Context 未 Done 导致无法退出。
   - **无界常驻集合：** Map/Cache 缺乏 TTL、LRU 或主动 Delete 机制，导致内存随时间只增不减。
   - **时间对象残留：** Ticker 未释放，或在 `for` 循环内滥用 `time.After` 导致创建大量临时定时器。
   - **长生命周期持有大对象：** Reslice 导致底层大数组无法被 GC；或者是大 Buffer 放入 `sync.Pool` 后被意外扩容并常驻。

4. 优先指出“结构上危险”的点。
   即使还没有 runtime 证据，只要满足下面任一条件，也要直接标成高风险：
   - map 没有上限
   - cache 没有 TTL 或淘汰机制
   - goroutine 没有退出条件
   - ticker 创建后没有 `Stop`
   - channel 可能只有生产没有消费
   - 大对象被放进全局状态或长生命周期对象

5. **输出结果优先级：**
   按危害程度倒序（最可能引起持续 OOM 的排最前），每个风险点必须输出：
   - **位置：** 文件名、行号/函数名。
   - **根因：** 为什么这里危险。
   - **性质定性：** 属于“确定性的真实泄露”还是“高概率的隐式内存上涨风险”。

## 判断规则
- 如果发现无界 map、无界 cache、未停止 ticker、未退出 goroutine，要明确提醒。
- 如果某个 map、cache、队列或对象集合明显可能随着 wallet、address、request id、tx hash、timestamp 之类唯一 key 增长，要直接标成重点风险。
- 用户只需“梳理排查、摸底代码”时，优先使用此方案，不要一上来就强制索要完整的 pprof 报告。
- 一旦用户能够提供 pprof (heap)、runtime 指标或线上 Grafana 内存趋势图，立即切换到更严谨的 `$go-memory-growth-check` 规则进行精准定量定位。

## 输出要求

输出尽量精简，直击痛点：
1. 精选出最可疑的 **3 到 5 个** 核心风险点。
2. 每个点标明所属的风险类型。
3. 给出是否需要跟进抓取 pprof 验证的明确判断。
