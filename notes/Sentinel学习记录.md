# Sentinel 学习记录

工程依托：[jdk8-sentinel-demo](https://github.com/xieqiansong/chaos-java/tree/main/jdk8-platform/jdk8-sentinel-demo)（`chaos-java` 仓库，Sentinel 1.8.6 + Spring Cloud Alibaba，程序化 `SphU` + `@SentinelResource` 双轨，含 Dashboard 与 Nacos 持久化）。

## 1. 安装

```bash
# 单元测试（无需任何外部依赖，规则由代码动态加载）
mvn -pl jdk8-sentinel-demo -am test

# 完整 Dashboard 体验（需 Docker）
docker compose -f jdk8-sentinel-demo/docker-compose.yml up -d
# Dashboard: http://localhost:8080，账号 sentinel/sentinel
mvn -pl jdk8-sentinel-demo -am spring-boot:run
```

Dashboard 镜像：`bladex/sentinel-dashboard:1.8.6`。规则持久化预留 Nacos（默认关闭），开启后规则存配置中心。

## 2. 流控（Flow Control）

**三要素**：资源名、阈值类型（`FLOW_GRADE_QPS` / `FLOW_GRADE_THREAD`）、流控效果（快速失败 / WarmUp / 排队等待）。

- **直接限流**：`SphU.entry("qpsDemo")` 尝试获取通行证，超限抛 `BlockException`。
- **关联限流**：秒杀下单（被保护资源）与支付（触发者）共享读库，支付 QPS 过高时连带限制下单。
- **WarmUp**：冷启动阈值从 `count/3` 线性爬升到 `count`，避免刚启动的节点被瞬时流量打垮（网关/连接池初始化场景）。

## 3. 熔断降级（Degrade）

**状态机**：`CLOSED → OPEN → HALF_OPEN → CLOSED`（探测成功）或 `OPEN`（探测失败）。

| 策略 | 触发条件 | 适用 |
|---|---|---|
| 异常比例 | 异常占比 > 阈值 | 依赖外部服务，偶发超时可接受 |
| 异常数 | 一分钟内异常数 > 阈值 | 关键接口不容任何异常 |
| 慢调用比例 | RT 超阈值占比 > 阈值 | 数据库慢查询 |

**关键 API**：`Tracer.trace(ex)` 将异常上报 Sentinel 统计——**只有被 trace 的异常才计入熔断决策**。

## 4. 热点参数限流（Hotspot）

不只看资源整体 QPS，而是按参数值做精细化控制：

```
默认: productId -> QPS=2  (普通用户)
例外: productId="vip" -> QPS=100  (VIP 白名单)
```

程序化方式通过 `SphU.entry(resourceName, entryType, count, args)` 传入参数值。生产场景：单商品秒杀限流、单 IP 防刷、VIP 豁免。

## 5. @SentinelResource 注解

```
业务异常 → BlockException?
  ├── 是 → blockHandler（限流专用处理）
  └── 否 → fallback（通用降级处理）
```

| | blockHandler | fallback |
|---|---|---|
| 触发条件 | BlockException | 任何异常 |
| 签名要求 | `(BlockException)` | `(Throwable)` |
| 能否共存 | 是 | 是 |

**生产推荐**：两者均配，`blockHandler` 可主动调用 `fallback` 统一输出格式。

## 踩坑

- **`Tracer.trace(ex)` 的重要性**：直接抛异常不走 trace，不会被计入熔断决策，熔断永远不触发。
- **blockHandler ≠ fallback**：blockHandler 只处理 `BlockException`，业务异常要用 fallback 兜底。
- **规则默认只在内存**：应用重启即丢、Dashboard 改完不回写。上生产必接 Nacos 持久化（`sentinel.nacos.enabled=true`，五种规则各一个 dataId：flow/degrade/param-flow/system/authority）。
- **单测要隔离规则**：Sentinel 规则是 JVM 级全局状态，每个用例 `@BeforeEach` 清空再各自加载，避免交叉污染。
- **核心场景用 `SphU.entry()` 而非注解**：不需要 Spring AOP 代理，单测更稳定可控。

## 进阶方向

- 链路流控（`EntryType.IN` + 链路模式）、系统自适应限流（Load/CPU/RT）、Sentinel 网关限流（Gateway 集成）。
- 集群流控（Token Server 模式）、规则动态推送（Dashboard → Nacos → 应用热更新）。

## 参考来源

- [Sentinel 官方文档](https://sentinelguard.io/zh-cn/docs/introduction.html)
- [Spring Cloud Alibaba Sentinel](https://spring-cloud-alibaba-group.github.io/github-pages/2021/zh-cn/index.html#_sentinel)
