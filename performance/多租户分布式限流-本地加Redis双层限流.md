# 多租户分布式限流：Redis+Lua → 本地+Redis 双层限流

> 性能优化 · 实战案例（有意向的取舍，实测量化）。对应 GitHub 工程：[chaos-java/jdk8-platform/jdk8-ratelimiter-demo](https://github.com/xieqiansong/chaos-java/tree/master/jdk8-platform/jdk8-ratelimiter-demo)。

## 一句话

用「窗口级精度损失」换「热路径零网络开销」：分布式限流从「每请求打 Redis」改成「本地拦截 + 周期校准」，并能量化精度损失的上界。

## 版本与适用

- 框架：Spring Boot 2.7、Spring Data Redis（Lettuce）
- 中间件：Redis 8（RESP3 回值）
- 关联工程：[chaos-java/jdk8-platform/jdk8-ratelimiter-demo](https://github.com/xieqiansong/chaos-java/tree/master/jdk8-platform/jdk8-ratelimiter-demo)（可 clone 复现：`mvn clean package` 后 `--ratelimiter.bench.enabled=true` 跑压测）

## 背景

多租户场景，每租户有独立全局限额（QPS），集群多节点共同承担流量，需**跨节点**保证不超限。

## 现状（Redis+Lua）与瓶颈

- 实现：每请求一次 `EVAL` 令牌桶 Lua，全局精确、原子。
- 瓶颈：每请求 1 次网络 RTT 叠加延迟；高 QPS 下 Redis 单机 CPU/连接成瓶颈。**限流本应是最廉价的守卫，反而成了热点依赖。**

## 优化：本地 + Redis 双层限流

### 设计

- **第一层（本地）**：每节点每租户一个内存令牌桶，热路径零网络。
- **第二层（Redis）**：全局窗口计数为权威，节点每 `windowMs` 校准一次。
- **校准语义**：滚动窗口计数 `count += 本节点消耗` → `remaining = windowQuota - count` → 返回 `allocated = remaining / N` → 本地桶重置为 `capacity = allocated × burst`（配额直接可用，而非慢速补发）。

### 精度损失上界

- 本地容量 = 每节点窗口配额 × burst（每节点窗口配额 = `windowQuota / N`）。
- 最坏：N 节点同时打满本地桶，全局超限 ≈ `(burst - 1) × windowQuota`。
- `burst = 1` 零超限但流量倾斜时单节点只能服务分片 → **欠用**（浪费配额）。

## 关键实现要点（含踩坑）

1. **本地令牌桶用时间戳补发**，多线程 `synchronized` 保证原子；校准即预填本期配额。
2. **踩坑 · Redis 8 返回整数**：RESP3 下 Lua 返回整数会触发 `ValueOutput.set(long)` 不支持报错 → 脚本统一 `return tostring(...)`，Java 端解析字符串，跨 RESP 版本稳健。
3. **Redis 抖动容错**：校准失败保留上一窗口配额、下窗口重试，本地不受影响，天然 fail-soft。
4. `redis-lua`（每请求 1 EVAL）作基准，`local-only`（纯本地）作性能下界参考，三个实现并列讲清取舍。

## 压测方案与指标

- 三档实现对照；指标：吞吐（放行/s）、延迟 avg/p99、`redis/s`（Redis 调用频率）、`overLimit%`（实际放行相对理论限额偏差，负=欠用）、本地命中率。
- 场景：未超限、超限、burst 敏感、流量倾斜、满速 flood。

### 实测（本机 Redis，8 线程）

| 维度 | redis-lua | local-redis |
|---|---|---|
| 超限场景 avg / p99 | 1080µs / 2370µs | **169µs / 19.8µs** |
| 吞吐上限（flood） | 9.7 万 req/s | **7400 万 req/s（≈769×）** |
| Redis 调用 | 每请求 1 次（≈9666/s） | 每窗口×节点（≈4.4/s，**≈1/2200**） |
| 精度损失 overLimit% | +4.2% | +10.6%（burst 1.5） |

辅助结论：`overLimit%` 随 burst 单调（1.0 → -7.8%、1.5 → +10.6%、3.0 → +25.0%）；流量倾斜 skew=0.8 → -29.5%（欠用），印证均分 N 的短板，引导「按负载加权分配」。

## 结论

- **高性能达成**：热路径近零开销（avg 169µs / p99 19.8µs），flood 吞吐提升约两个数量级。
- **Redis 负载骤降**：调用频率从「每请求」降到「每窗口×租户×节点」（≈1/2200）。
- **精度可解释可调**：损失上界与 burst 强相关、随 burst 单调，决策可量化。

## 后续增强

- 严格模式：本地满回落 Redis 再判一次，精度近 100%。
- 动态节点发现：`nodeCount` 由注册中心实时更新。
- 配额按历史负载加权分配，缓解倾斜欠用。

## 参考来源

- 关联工程：[chaos-java/jdk8-platform/jdk8-ratelimiter-demo](https://github.com/xieqiansong/chaos-java/tree/master/jdk8-platform/jdk8-ratelimiter-demo)
- 令牌桶 Lua 实现参考 Redis 官方/通用令牌桶脚本写法（仅思路，无源码搬运）