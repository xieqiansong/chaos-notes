# Java 内核与并发内功（学习进度 · 场景清单）

> 工程来源：[chaos-java/jdk8-base](https://github.com/your-username/chaos/blob/main/chaos-java/jdk8-platform/jdk8-base)
> 分类：学习类 · 记录覆盖的场景 / 功能清单 + 掌握程度
> 定位：这是「Java 内功」仓库，106 个可运行例子，讲清底层原理的实战底气来源。

## 一、已覆盖的能力地图

| 方向 | 工程落点 | 覆盖的场景 / 功能 |
|------|----------|------------------|
| 基础 | `java/base` | 反射（`HelloReflect`）、注解（自定义注解处理器）、集合源码级用法、泛型边界、`Exception` 体系 |
| IO | `java/io`（21 个例子） | BIO/NIO/`Selector`、`Channel`/`Buffer`、零拷贝思路、序列化 |
| 并发 JUC | `java/juc`（32 个例子） | 见下表 |
| JVM | `java/jvm`、`jvm/gc`、`jvm/agent` | 类加载机制（自定义 `ClassLoader`）、GC 演示、OOM 触发、Java Agent 字节码增强 |
| SPI | `java/spi` | `ServiceLoader`  SPI 机制 |
| 分布式内核 | `distributed/system` | Paxos / Raft 一致性算法最小实现、分布式 ID 发号 |

## 二、JUC 重点场景清单

- **AQS 体系**（`juc/aqs`）：手写 / 剖析 `ReentrantLock`、`Semaphore`、`CountDownLatch` 的同步器骨架
- **线程池**（`juc/thread`、`ScheduledThreadPoolDemo`）：`ThreadPoolExecutor` 参数取舍、`CallerRunsPolicy` 等拒绝策略、定时任务池
- **同步协作**：`CountDownLatch` / `CyclicBarrier` / `Exchanger` / `Phaser`（基础/动态/分层三套示例，体现对 Phaser 的深入理解）
- **锁的演进**：`ReentrantReadWriteLock`、JDK8 的 `StampedLock`（乐观读）、死锁演示 `LockDeadlockDemo`
- **并发容器**：`ConcurrentHashMap`、`ConcurrentLinkedQueue`、`CopyOnWriteArrayList` 用法与适用场景
- **异步**：`Future` / `Callable`（`FutureDemo` / `CallDemo`）、`ThreadLocal` 内存泄漏注意点

## 三、分布式内核（亮点）

- **Paxos / Raft**：在 `distributed/system` 下用纯 Java 实现了最小可用的提案/复制流程，不是调库而是理解「共识怎么达成」。
- **分布式 ID**：Snowflake 与号段（Leaf）两种发号器实现（短链系统即基于此，见[短链系统设计](./短链系统设计.md)）。

## 四、掌握程度自评

- ✅ 能讲清 AQS `state` + 队列 + CAS 的三件套原理
- ✅ 线程池参数能结合实际场景给出取值理由
- ✅ JVM 类加载双亲委派、Agent 插桩有落地代码
- 🔶 Paxos 多副本容错边界还需补一轮压测观察

## 五、后续计划
- 补 JMH 微基准对比 `synchronized` vs `StampedLock` 乐观读吞吐
- Agent 模块扩展：无侵入埋点 / 链路追踪探针 demo
