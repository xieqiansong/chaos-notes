# TCP-over-WebSocket 隧道：端到端性能基准与瓶颈分析

> 性能优化 · 实战案例。对应 GitHub 工程：[chaos-tcp-over-websockets](https://github.com/xieqiansong/chaos-tcp-over-websockets)。

场景是借用 HTTP 端口做端口转发（比如受限网络下访问内网 3306）。TCP-over-WebSocket 隧道把任意 TCP 流量经 WebSocket 帧透传。本篇记录一次「先把基准测准、再找瓶颈、再尝试优化」的完整过程。

先说结论：在受限网络借 HTTP 端口这个场景下，WebSocket 是唯一通用解，它的软件开销（loopback 下约 14 MB/s 上限）已经接近架构天花板。真正的瓶颈在网络链路，不在隧道软件微调——所以后面那些细粒度优化，都只有 0~20% 的小幅波动。

## 为什么是 WebSocket

虽然 HTTP 的 `Upgrade` 头是通用协议升级机制（理论上可以升级到任意协议），但中间设备（Nginx/防火墙/CDN/WAF）普遍只透传 `Upgrade: websocket`。所以在受限网络下，WebSocket 是唯一能同时做到通用穿透 + 全双工 + 有标准实现的方案。换 CONNECT / HTTP/2 / 自定义协议都不通用，中间设备根本不放行。

因此 WS 编解码开销是"换取穿透性必须付出的代价"，绕不过去。

## 先解决基准不可复现的问题

最初端到端测试 `realWorldThreeStrategies` 是在一次 JVM 内连续混跑：3 拷贝策略 × 4 包大小（12 个组合）。结果一团乱：

- retained 在 256KB/4MB 大包下偶发 `SocketTimeoutException`
- 4MB 那轮全部 `Connection refused`（隧道 server 没起来）
- `IllegalReferenceCountException: refCnt: 0` 刷屏

把主代码逐步改造（共享 EventLoop 等）后症状减轻，但 retained 还是偶发崩。后来做了单轮对照实验，发现关键事实：

- 单独跑 retained + 256KB / 4MB 完全正常（retained 4MB 单跑 11.80 MB/s）
- 崩溃只出现在连续混跑的第 3/4 轮

根因不是代码 bug，而是连续混跑时前面轮次的资源污染了后面。每轮都新建 TcpServer/WebsocketServer/TcpClient/WebsocketClient，每个连接各自 `new NioEventLoopGroup()` + `newFixedThreadPool(32)`，而且从不开新就开一堆，跨轮累积线程/端口/引用计数。到第 3/4 轮资源耗尽，新 server bind 失败、连接被拒、大包下偶发超时。

改法是把这个场景重构成单次单组合跑：

- `runSingle()` 通过 `-Dbench.strategy` / `-Dbench.payload` 指定，每次 JVM 只起一个隧道（一个策略 × 一个包大小）+ 一个直连 echo 对照，测完即清理
- 结果以标准 CSV 追加到 `target/bench-results.log`（字段：`timestamp,strategy,payloadBytes,tunnelMbPerSec,tunnelRttMs,tunnelReceived,tunnelError,directMbPerSec,directRttMs,directReceived,directError`），方便程序化解析，不用靠肉眼盯控制台
- 每个场景测 3 次取平均，抵消单次波动

```bash
mvn test "-Dtest=TunnelRealWorldBenchmarkTest#runSingle" -Dbench.strategy=retained -Dbench.payload=262144
```

改完 12 个组合全部稳定、可复现，无崩溃——说明崩溃根源是测试方法，不是代码。

## 瓶颈到底在哪

### 干净基线（3 次取平均，loopback）

| 包大小 | copied 吞吐 | copied RTT | retained 吞吐 | retained RTT | direct（直连基线） |
|---|---|---|---|---|---|
| 1KB | 2.53 MB/s | 1403 ms | 2.31 MB/s | 1506 ms | 16.61 MB/s |
| 16KB | 8.57 MB/s | 274 ms | 9.09 MB/s | 265 ms | 173.91 MB/s |
| 256KB | 13.85 MB/s | 103 ms | 13.53 MB/s | 101 ms | 364.81 MB/s |
| 4MB | 13.02 MB/s | 48 ms | 14.14 MB/s | 56 ms | 206.85 MB/s |

（duplicate 单跑就 CRASH——共享引用计数在异步链路下不安全，不能当默认策略）

几个观察：

1. copied 和 retained 端到端没有显著差异（差距 <10%，在波动内）。零拷贝在跨 EventLoop 转发下没体现优势，它的跨线程引用计数管理开销，差不多等于 loopback 下做一次内存拷贝。
2. 吞吐随包大小上升，但到 4MB 后就平台化了（1KB≈2.5 → 4MB≈14）。典型的「每包固定开销 + 每字节成本」双重主导，包大到一定程度固定开销被摊薄，每字节成本成了上限。
3. direct 远高于隧道（16~365 vs 2.5~14），隧道链路固有开销主导，大约 10~30 倍的衰减。

### 定位：每字节的数据处理成本

用控制变量实验排除外围因素：

| 尝试的优化 | 实测效果 | 结论 |
|---|---|---|
| 移除固定 4KB 收包（回退默认 Adaptive 64KB） | copied 4MB 12.54 → 15.44（+23%），RTT 73→33ms | 收包切片是次要因素 |
| 去掉固定线程池（构造时同步建连） | 吞吐持平 | 线程池不在数据转发路径，解决的是并发上限/死锁 |
| WS 帧上限 + 收包缓冲调大到 8MB | copied 4MB 13.02 → 14.26（约 +9.5%） | 帧数不是主要瓶颈 |

最说明问题的一个实验：把 4MB 从约 64 帧（64KB/帧）降到约 1 帧（8MB 帧上限），吞吐只提升约 10%。如果瓶颈是「帧数 × 每帧固定开销」，这里应该有数倍提升。所以瓶颈是**每字节的数据处理成本**：

- WS 帧编解码要处理每个字节（掩码、帧头、数据搬运）
- 转发路径每次 `writeAndFlush` 跨线程投递 + 引用计数管理，都要搬每字节
- 减少帧数不减少每字节成本，所以提升有限

补充一点：loopback 测试剥离了网络变量，测的是隧道软件本身的吞吐上限（约 14 MB/s）。真实部署里 client/server 是不同进程甚至不同机器，瓶颈更多由网络 RTT / 带宽 / 中间设备决定，软件开销往往不是主矛盾。所以软件层微调在真实高带宽网络里，收益会被网络瓶颈盖住。

## 改过的方案和取舍

| 改动 | 收益 | 代价/风险 | 是否保留 |
|---|---|---|---|
| 基准改单次单组合 + CSV | 测量可信化（最本质） | 每次单跑耗时增加 | ✅ 保留 |
| 去掉固定线程池 | 消除并发连接数上限（32→无）、消除每连接驻留线程、消除 `close()` 在 EventLoop 线程 `sync()` 死锁 | 构造时同步 connect 会短暂阻塞建连线程 | ✅ 保留（工程合理性） |
| WS 帧上限 + 收包缓冲调大到 8MB | 大包约 +9.5% | 内存占用上升（每帧最大 8MB） | ✅ 保留（方向性收益） |
| 批量写 + 周期 flush | 小包可能 1.5-2×，大包约 5-10% | 增加延迟；且不减少跨线程 write 次数 | ⚠️ 没采用（不符合当前场景方向） |
| 共享 EventLoop / 同线程绑定 | 浅共享不提升单连接吞吐；同线程绑定需 client/server 同进程 | 同线程绑定在 client/server 分离部署下没意义 | ❌ 不适用（架构前提不成立） |

## 落地结论

1. **测量方法比优化更重要。** 连续混跑基准不可复现，会让人得出"代码有 bug"的错误结论；单次单组合 + CSV + 取平均才有可信基线。
2. **这架构软件层已近天花板。** loopback 下隧道吞吐上限约 14 MB/s，瓶颈是每字节的 WS 编解码/跨线程投递成本，外围配置（缓冲、帧大小、线程池）只能带来 0~20% 的边际改善。
3. **WS 是受限网络场景的必然选择。** 它的编解码开销是换穿透性的代价，没法靠隧道内部优化绕开。
4. **真正要提吞吐得靠网络链路，不是隧道软件微调。** 要突破得走架构级重设计（去 WS / 换传输层 / 多路并行），不在当前代码优化范畴。

## 还没做的（架构级）

- **去 WebSocket 化**：如果中间件放行 HTTP CONNECT 或自定义升级，可以用纯 TCP 透传，省掉每字节 WS 开销（但受限网络通常不放行，所以不通用）
- **多路 WS 并行分片**：用多连接摊薄单流吞吐上限
- **真实跨机压测**：先确认网络链路是不是才是瓶颈，再决定要不要继续优化软件层

## 参考来源

- 工程：[chaos-tcp-over-websockets](https://github.com/xieqiansong/chaos-tcp-over-websockets)，其 `TEST_REPORT.md` 含完整基准与过程
- 原项目：`995270418L/Tcp_Over_websockets`（Netty TCP/WebSocket 编解码参考）
