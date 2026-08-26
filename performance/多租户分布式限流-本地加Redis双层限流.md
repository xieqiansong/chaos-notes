# 多租户分布式限流：Redis+Lua → 本地+Redis 双层限流

> 性能优化 · 实战案例（有取舍、有量化）。对应 GitHub 工程：[chaos-java/jdk8-platform/jdk8-ratelimiter-demo](https://github.com/xieqiansong/chaos-java/tree/master/jdk8-platform/jdk8-ratelimiter-demo)。

多租户场景，每个租户有独立全局限额（QPS），集群多节点共同承担流量，需要**跨节点**保证不超限。

原来的做法是每请求打一次 Redis（`EVAL` 令牌桶 Lua），全局精确、原子。但代价是每请求一次网络 RTT，高 QPS 下 Redis 单机 CPU/连接直接成瓶颈。**限流本该是最廉价的守卫，反而成了热点依赖。**

这篇改成「本地拦截 + 周期校准」：用一点窗口级精度损失，换热路径零网络开销，并且能量化精度损失的上界。

## 本地 + Redis 双层限流

### 设计

- **第一层（本地）**：每节点每租户一个内存令牌桶，热路径零网络。
- **第二层（Redis）**：全局窗口计数为权威，节点每 `windowMs` 校准一次。
- **校准语义**：滚动窗口计数 `count += 本节点消耗` → `remaining = windowQuota - count` → 返回 `allocated = remaining / N` → 本地桶重置为 `capacity = allocated × burst`（配额直接可用，而不是慢速补发）。

### 精度损失上界

- 本地容量 = 每节点窗口配额 × burst（每节点窗口配额 = `windowQuota / N`）。
- 最坏情况：N 节点同时打满本地桶，全局超限约 `(burst - 1) × windowQuota`。
- `burst = 1` 零超限，但流量倾斜时单节点只能服务自己的分片 → **欠用**（浪费配额）。

## 踩过的坑

1. 本地令牌桶用时间戳补发，多线程 `synchronized` 保证原子；校准即预填本期配额。
2. **Redis 8 返回整数**：RESP3 下 Lua 返回整数会触发 `ValueOutput.set(long)` 不支持报错 → 脚本统一 `return tostring(...)`，Java 端解析字符串，跨 RESP 版本稳健。
3. **Redis 抖动容错**：校准失败保留上一窗口配额、下窗口重试，本地不受影响，天然 fail-soft。
4. `redis-lua`（每请求 1 EVAL）作基准，`local-only`（纯本地）作性能下界参考，三个实现并列讲清取舍。

## 压测方案

基准已做成 **SpringBootTest 一键跑**：场景矩阵在工程 [BenchMarkTest](https://github.com/xieqiansong/chaos-java/tree/master/jdk8-platform/jdk8-ratelimiter-demo/src/test/java/lan/chaos/ratelimiter/BenchMarkTest.java) 定义，`mvn test` 即跑完并自动写出 markdown，对应工程 `TEST_REPORT.md`。

- 指标：`avg / p99`（µs）；`redis/s`（Redis 调用频率，含 EVAL/EVALSHA）；`overLimit%`（实际放行相对理论限额偏差，正=超限、负=欠用）。
- 三档实现对照；场景：未超限（A）、超限（B）、burst 敏感、流量倾斜（skew）、满速 flood。
- 每场景前自动 `flushDb` 隔离 key 残留；压测引擎 [BenchEngine](https://github.com/xieqiansong/chaos-java/tree/master/jdk8-platform/jdk8-ratelimiter-demo/src/main/java/lan/chaos/ratelimiter/bench/BenchEngine.java) 供命令行与测试共用。

### 实测一：三实现基准对照（本机 Redis 8、8 线程）

| 场景 | 模式 | avg(µs) | p99(µs) | redis/s | overLimit% |
|---|---|---|---|---|---|
| A 未超限（qps=500） | redis-lua | 6775 | 13622 | 499 | -49.9 |
| A 未超限（qps=500） | local-redis | 33 | 42 | 4.8 | -49.9 |
| B 超限（qps=2000） | redis-lua | 7661 | 13007 | 1042 | +4.4 |
| B 超限（qps=2000） | local-redis | **8** | **16** | 4.8 | +22.8 |
| B burst=1.0 | local-redis | 8 | 8 | 5.0 | +13.9 |
| B burst=3.0 | local-redis | 7 | 3 | 5.0 | +39.7 |
| B skew=0.8 | local-redis | 11 | 34 | 5.6 | -24.3 |
| flood | redis-lua | 8164 | 16241 | 978 | -2.0 |
| flood | local-redis | **0.5** | 0.6 | 5.6 | +78.1 |

几个观察：

- **热路径近零开销**：超限 B 场景 `local-redis` avg 8µs / p99 16µs（对照 `redis-lua` 7661µs / 13007µs），flood 下更是 0.5µs vs 8164µs——差 3~4 个数量级。
- **Redis 负载骤降**：调用频率从「约请求数/秒」（约 1000/s）降到「每窗口 × 租户 × 节点」（约 5/s）。
- **精度可解释可调**：`overLimit%` 随 burst 单调（1.0→+14%、1.5→+23%、3.0→+40%）；倾斜 skew=0.8 → -24.3%（欠用），印证「均分 N」的短板，引导按负载加权分配。

### 实测二：window-ms 对精度的取舍（local-redis，nodes=4，burst=1.5，qps=2000>limit=1000）

| window-ms | avg(µs) | redis/s | overLimit% | 说明 |
|---|---|---|---|---|
| 250 | 6 | 17.0 | **-2.1** | 校准最勤，几乎不超限；Redis 成本最高 |
| 500 | 7 | 9.0 | +2.6 | — |
| 1000（默认） | 8 | 4.8 | +22.8 | 成本/精度均衡 |
| 2000 | 6 | 3.2 | +32.4 | 窗口变大，超限上升 |
| 4000 | 6 | **2.6** | **+70.3** | 校准最稀，Redis 成本最低，超限明显放大 |

几个观察：

- **成本随窗口单调下降**：`redis/s` 从 17.0 → 2.6（约 6.5×），窗口越大越省 Redis——这是调大窗口的直接动机。
- **精度随窗口单调恶化**：`overLimit%` 从 -2.1% → +70.3%，窗口越大、错配持续越久、权威纠正越晚，超发越重。
- **热路径延迟几乎不变**（6~8µs）：窗口只作用于「精度-成本」轴，不进热路径，调参不牺牲首层性能。
- **不是越大越好**：4000ms 超发严重（+70%）且对突发响应迟钝；250ms 欠用兼成本高 → 默认 1000ms 是便宜又稳的折中点。

## 结论

- **高性能达成**：超限场景 `local-redis` avg 8µs / p99 16µs（对照 `redis-lua` 呈 3~4 个数量级优势），flood 下 0.5µs vs 8164µs。
- **Redis 负载骤降**：调用频率从「每请求」（约 1000/s）降到「每窗口 × 租户 × 节点」（约 5/s）。
- **精度可解释可调**：超限上界与 `burst`、`window-ms` 均强相关且单调（实测一 burst、实测二 window 均证），决策可量化。

## 后续可做的

- 严格模式：本地满回落 Redis 再判一次，精度接近 100%。
- 动态节点发现：`nodeCount` 由注册中心实时更新。
- 配额按历史负载加权分配，缓解倾斜欠用。

## 参考来源

- 关联工程：[chaos-java/jdk8-platform/jdk8-ratelimiter-demo](https://github.com/xieqiansong/chaos-java/tree/master/jdk8-platform/jdk8-ratelimiter-demo)
- 令牌桶 Lua 实现参考 Redis 官方/通用令牌桶脚本写法（仅思路，无源码搬运）
