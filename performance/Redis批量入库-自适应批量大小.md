# Redis 批量入库：自适应批量大小（攒批 + 在线寻优）

> 性能优化 · 实战案例。对应 GitHub 工程：[chaos-java/jdk8-platform/jdk8-batch-ingest-demo](https://github.com/xieqiansong/chaos-java/tree/master/jdk8-platform/jdk8-batch-ingest-demo)。

高吞吐数据汇聚场景（探针/日志/埋点这种），每条数据一次 Redis 写命令。命令数随数据量线性放大，网络往返和 Redis CPU 双双成了瓶颈。想靠固定批量 + Pipeline 优化，又卡在最优批量依赖负载和并发、很难人工定死。

这篇的做法是「内存桶攒批 + 在线寻优」：命令数从「=条目数」降到「≈批次数」，再用探索-反馈-平滑的机制自动逼近最优批量，不用人肉调参。

## 现状和瓶颈

- 实现：`legacy` 每条数据一次 `HSET`，命令数 = 条目数。
- 瓶颈：8 汇聚线程压测下吞吐约 12k/s，`redis/s = 条目数`（1.2 万），命令往返是天花板。批量收益是真实的，但**静态批量有最优值**——单线程 flood 下 2048 比 512 高 14%，双线程下 512 反而反超 2%，人工定死没法兼顾。

## 自适应批量入库引擎

### 设计

- **一级内存桶**：有界 `ArrayBlockingQueue`，消费线程直取攒批，给存储天然削峰。
- **水位触发**：队列积压够时 `drainTo` 瞬取攒满目标批量 → flush；空转超 `idle-flush-ms` 兜底 flush。
- **批量自适应**（核心）：候选 = 当前批量 × {0.5, 0.8, 1.0, 1.25, 2.0}；按 `explore-prob` 概率试探候选做性能采样；反馈用**指数衰减加权平均**得到各候选吞吐；每 `sample-window` 次 flush 挑「加权最优」并以平滑系数过渡，收敛稳定。
- **固定并发消费线程**：`writer-threads` 个线程直取内存桶并发写（不设第二级任务队列）。
- **核算闭合**：数据只在内存桶一个队列里流转，丢弃只在 `offer` 失败时计入 `dropped`，`written + dropped + errors = 总投递`，可审计。

### 踩过的坑

1. **尾批卡死丢失**：`drainLoop` 在「队列空 + 本地残批不满」时，`if(!batch.isEmpty())` 每次循环都刷新空闲计时 → `idleFlushNs` 兜底永不触发 → 尾批滞留。改成只按 `drainTo` 实际新取条数刷新计时，`while` 退出后把残批刷出去。
2. **legacy 全量 ClassCastException**：`opsForHash().put(key, System.nanoTime(), item)` 的 hashKey 传了 `Long`，而 `StringRedisSerializer` 只接受 String → 每条直接抛异常、legacy 显示 0 吞吐。改成 `String.valueOf(...)`。
3. **static 丢弃数采集缺失**：`BenchEngine` 原只在 `instanceof AdaptiveBatchWriter` 时读 `dropped()`，static 的丢弃恒为 0（假象）。给 `BatchWriter` 接口加了 `default dropped()` 统一读取。
4. **dropped 假零黑洞（最大一个坑）**：曾引入「单承包人 + 写线程池」双层队列，写池任务队列无界 → 承包人把内存桶快速转存进写池、内存桶永远"满"不了 → `dropped` 恒 0；生产(约8M/s) >> 消费(约23k/s) 的差额全积压在写池队列，`close()` 只消化了极小部分，**几千万条随 JVM 退出无声丢了**。**最终方案**：去掉写池和第二级队列，消费线程直接取内存桶——单队列核算天然闭合，`dropped` 恢复真实值。
5. **压测环境隔离**：本地端口转发隧道偶发抖动 + 遗留大 hash 拖慢后续场景 → 每场景前 `DEL key` 隔离；3 分钟级长跑对受限环境压力太大，收敛专项取 60s。

## 压测方案

基准做成 **SpringBootTest 一键跑**：场景矩阵在工程 [BenchMarkTest](https://github.com/xieqiansong/chaos-java/tree/master/jdk8-platform/jdk8-batch-ingest-demo/src/test/java/lan/chaos/batchwriter/bench/BenchMarkTest.java) 定义，`mvn test` 跑完 6 场景 + 60s 收敛专项并自动写出 markdown（`target/bench-results.md` + 工程 `TEST_REPORT.md`）。

- 指标：`items/s`（写入吞吐）、`redisCmds/s`（Redis 往返数，legacy=条目数、批量≈批次数）、`avgBatch`（平均批量）、`dropped`（队列满丢弃）、`errors`（写失败）。
- 三实现对照：`legacy`（逐条直写）/ `static`（固定批 512 + Pipeline）/ `adaptive`（攒批 + 在线寻优）。
- 场景：匀速（rate=5000，贴近真实负载）、flood（灌爆测吞吐上限）、收敛专项（flood 60s 观察 batchSize 寻优轨迹）。

### 实测一：1 线程（writer-threads=1，static 固定批 512）

| 场景 | 模式 | items/s | redisCmds/s | avgBatch | dropped |
|---|---|---|---|---|---|
| 匀速 | legacy | 4999 | 4999 | 1.0 | 0 |
| 匀速 | static(512) | 4963 | 10.2 | 484 | 0 |
| 匀速 | adaptive | 4948 | 2.4 | 2048 | 0 |
| flood | legacy | 11815 | 11815 | 1.0 | 0 |
| flood | static(512) | 20221 | 39.5 | 512 | 91.3M |
| flood | adaptive | **24052** | 13.6 | 1771 | 86.3M |

要点：

- **命令量骤降**：匀速下 `redisCmds/s` 从 4999 → adaptive 2.4（约 **2000 倍**）；flood 下 11815 → 13.6（约 **870 倍**）。命令数从「=条目数」降到「≈批次数」。
- **吞吐**：flood 下 adaptive 是 legacy 的 **2.0 倍**、static(512) 的 1.2 倍。
- **核算闭合**：flood 丢弃（生产约8M/s >> 消费约24k/s）真实计入 `dropped`，匀速 dropped=0。

### 实测二：2 线程（writer-threads=2，static 固定批 512）

| 场景 | 1 线程 items/s | 2 线程 items/s | 提升 |
|---|---|---|---|
| flood static(512) | 20221 | **37160** | **+84%** |
| flood adaptive | 24052 | **36456** | **+52%** |
| flood legacy | 11815 | 11466 | -3%（无消费线程，波动） |
| 匀速 adaptive | 4948 | 4778 | -3.4%（波动） |

要点：

- 批量 Pipeline 与 Redis **并发执行**，2 消费线程在 flood 下吞吐 +52%~84%，逼近单实例上限；匀速低负载下线程数影响小（±3%）。

### 实测三：固定批量 512 vs 2048（最优批量随负载/并发动态变化）

| 场景 | static-512 | static-2048（历史） | 差异 |
|---|---|---|---|
| flood 单线程 | 20221 | 23548 | **-14%**（大批量优） |
| flood 双线程 | 37160 | 36513 | **+2%**（持平，并发流水线补偿） |

要点：最优批量不是常数——单线程下 2048 更优、双线程下 512 反而略好。**靠人工定死没法兼顾**，这正是 `adaptive` 在线寻优的价值（自适应两组均约等于 static 最优值）。

### 实测四：自适应收敛专项（flood 60s）

| 线程 | 寻优次数 | 批量轨迹 | items/s | errors |
|---|---|---|---|---|
| 1 线程 | 7 次 | 2048 → 1331 → 3481 → … → 1638（围绕 ~1600 震荡） | 25852 | 0 |
| 2 线程 | 11 次 | 2048 → 2406 → 1563 → … → **3841**（后半程持续爬升） | 40176 | 0 |

要点：

- **动态引擎核心实证**：批量大小随实测吞吐反馈持续「探索-反馈」调整，而不是定死一个值；低/高候选交替正是探索特性。
- 双线程批量爬到 3841、吞吐 40176/s、errors=0，命令量较逐条降约 600 倍。

## 结论

- **高性能达成**：flood 下 `adaptive`（1 线程）T/s=24052，是 `legacy` 的 **2.0 倍**、与 `static` 最优相当；2 线程再提升 +52%（36456/s）。
- **命令量骤降**：`redisCmds/s` 从「=条目数」降到「≈批次数」，匀速降约 **2000 倍**、flood 降约 **870 倍**。
- **在线寻优价值**：最优批量随负载/并发动态变化（实测三），`adaptive` 自动逼近、无需人工调参（实测四）。
- **核算闭合**：单队列模型下 `written+dropped+errors=总投递`，flood 大量丢弃是容量必然且真实可审计；匀速 dropped=0。
- **两轮全场景 errors=0**：内存测试 30 万条 100% 排空，引擎逻辑独立于 Redis 稳定性。

## 后续可做的

- **背压策略升级**：当前内存桶满即丢弃计数；可提供「降级直写 / 阻塞背压」模式，把丢失换成延时，适配"不可丢"场景。
- **按 key 分桶多线程**：当前 `writer-threads` 个消费线程直取内存桶并发写；更细粒度可按 key 分桶并发，提升多租户写入吞吐。
- **批量写 MySQL**：复用同一攒批/寻优骨架，把 `storage()` 换成批量 upsert，验证批量收益在关系型存储的迁移性。

## 参考来源

- 关联工程：[chaos-java/jdk8-platform/jdk8-batch-ingest-demo](https://github.com/xieqiansong/chaos-java/tree/master/jdk8-platform/jdk8-batch-ingest-demo)
- 实现要点全部来自该工程真实开发与压测过程（含踩坑记录，见工程 `TEST_REPORT.md`）
