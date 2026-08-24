# RocketMQ 学习记录

> 学习记录：以真实可运行工程为依托。
> **参照工程**：[jdk8-rocketmq-demo](https://github.com/xieqiansong/chaos-java/tree/main/jdk8-platform/jdk8-rocketmq-demo)（`chaos-java` 仓库），一个覆盖 RocketMQ 常用能力维度的 Spring Boot Demo（所有场景由 `DemoTest` 统一触发）。
> 本文不重述官方概论，只记录动手做出来的机制、关键 API 与踩坑点。

## 版本与适用

| 组件 | 版本 |
|---|---|
| rocketmq-client | 4.9.8 |
| rocketmq-spring-boot-starter | 2.2.3 |
| spring-boot | 2.7.18 |
| JDK | 8（`jdk8-platform`） |

运行方式：`docker-compose up -d` 起单节点 NameServer + Broker，再 `mvn test`（无 Broker 时测试用 `Assumptions` 优雅跳过，不误报）。

## 总体认知（先说结论）

1. **at-least-once 投递**：RocketMQ 只保证"至少一次"，重试 / 故障切换都会造成**重复消费**，消费端幂等是**义务**，不是可选项。
2. **Topic / ConsumerGroup 是契约字符串**：生产端与消费端必须严格对齐，拼错不报编译错，但运行期静默收不到消息。务必收口到常量中心（本 demo 的 `MqConstant`）。
3. **投递可靠性偏好**：同步发送可靠性最高；异步次之；单向吞吐最大但可丢。
4. **频率分级**：真正"几乎必写"的是 **同步发送 + 幂等消费**；**重试/死信、延迟、按 Key 检索、发送容错、消息轨迹** 属高频；顺序、事务、批量、过滤、广播、请求应答、主动拉取、ACL、限流调优按业务/治理选用；**SQL92 过滤**相对低频。

## 各能力维度（机制 + 关键 API + 坑）

### 1. 基础收发：同步 / 异步 / 单向

| 语义 | API | 特点 |
|---|---|---|
| 同步 | `syncSend` | 阻塞等 Broker 确认，可靠性最高，**务必处理异常**（Broker 不可达会抛异常） |
| 异步 | `asyncSend` + `SendCallback` | 不阻塞主线程，回调里处理结果，记 `msgId` 便于排查 |
| 单向 | `sendOneWay` | 只发不收，吞吐最大，适合日志等可丢场景 |

发送前统一 `MessageUtils.pack(body)` 拼 `时间戳|正文`，消费端借此打印"从发到消费"的端到端耗时（刻意不用 JSON，避免序列化开销污染耗时测量）。

```java
// 同步：阻塞等待确认，可靠性最高，务必处理异常
return rocketMQTemplate.syncSend(MqConstant.TOPIC_BASIC, payload);
// 单向：只发不收，吞吐最大，适合日志等可丢场景
rocketMQTemplate.sendOneWay(MqConstant.TOPIC_BASIC, MessageUtils.pack(body));
```

### 2. 幂等消费（高频，必做）

`RocketMQListener<MessageExt>` 才能拿到元数据（`msgId`）。**用 Broker 保证全局唯一的 `msgId` 做去重**：处理前查 `messageIdStore.isProcessed(msgId)`，处理后 `markProcessed`。

```java
@RocketMQMessageListener(topic = MqConstant.TOPIC_BASIC, consumerGroup = MqConstant.GROUP_BASIC)
public class IdempotentConsumer implements RocketMQListener<MessageExt> {
    public void onMessage(MessageExt message) {
        String msgId = message.getMsgId();
        if (messageIdStore.isProcessed(msgId)) { /* 重复消息，跳过 */ return; }
        // ... 业务处理 ...
        messageIdStore.markProcessed(msgId);
    }
}
```

> **坑**：内存实现的去重存储重启 / 多实例失效，生产必须换 Redis(SETNX+EX) 或数据库唯一键。
> 时序：处理成功后再 `markProcessed`，若先标记再中途失败会漏消费。

### 3. 重试与死信（高频）

消费失败自动重试（默认约 16 次），耗尽后进入死信 Topic `%DLQ% + 原消费组名`。

```java
@RocketMQMessageListener(topic = MqConstant.DLQ_TOPIC_RETRY, consumerGroup = MqConstant.GROUP_DLQ_HANDLER)
public class DeadLetterConsumer implements RocketMQListener<String> {
    public void onMessage(String message) {
        // 进入人工补偿：落补偿表 / 告警 / 人工审核
    }
}
```

> **坑**：死信必须由**全新消费组**消费，不能复用原组。用派生常量 `"${'%DLQ%'}" + GROUP_RETRY` 收口。

### 4. 延迟消息（高频）

RocketMQ 原生支持固定延迟级别（`1s/5s/10s/.../2h`），指定级别即可——这是相对 Kafka 需自建延迟方案的差异化能力。

```java
rocketMQTemplate.syncSendDelayTimeMillis(MqConstant.TOPIC_DELAY, payload, 1000);
```

> **坑**：**不是任意秒数**，只能选内置档位；当任意毫秒读会得到意外延迟。

### 5. 按 Key 检索（高频）

发送时设 `keys`（常为业务单号），可用 `queryMsgByKey` 反向检索（依赖 Broker 的 index 文件），按业务单号查消息排查。

### 6. 发送侧容错（高频）

生产默认就该显式配置的"抗抖动"开关，属于 `DefaultMQProducer` 全局配置，应写在初始化/配置类里一次性设定：

```java
producer.setRetryTimesWhenSendFailed(3);                 // 同步发送失败重试次数（默认 2）
producer.setSendMsgTimeout(3000);                        // 单次发送超时 ms
producer.setSendLatencyFaultEnable(true);                // 避峰：对慢 Broker 短期规避，切到健康节点
producer.setRetryAnotherBrokerWhenNotStoreOK(true);      // 落盘未 OK 时换 Broker 重试
```

> **坑**：不开启 `sendLatencyFaultEnable` 时，某个慢 Broker 会持续被选中，拖累整体发送耗时。

### 7. 消息轨迹（高频）

开启后 SDK 自动把"发送 → 存储 → 消费"各阶段上报到 trace topic，排查"发了但消费端没收 / 卡在哪"几乎必备。

> **坑**：4.9.8 的 `DefaultMQProducer` **没有 `setEnableMsgTrace`**，只能通过构造函数开启，所以自建带轨迹开关的生产者：
```java
producer = new DefaultMQProducer(GROUP_TRACE, null, true, MqConstant.TRACE_TOPIC);
```
> **坑**：trace topic（`RMQ_SYS_TRACE_TOPIC`）必须已存在于 Broker，否则仅丢轨迹，不影响主消息。

### 8. 顺序消息（分区 / 全局，中频）

- **分区顺序**：相同 `shardingKey`（如 `orderId`）路由到同一队列，局部有序——最常用。
- **全局顺序**：单队列 Topic，全局有序，吞吐受限，少用。

消费必须以 **ORDERLY** 模式，单队列串行：

```java
@RocketMQMessageListener(topic = TOPIC_ORDER, consumerGroup = GROUP_ORDER, consumeMode = ConsumeMode.ORDERLY)
public class OrderedConsumer implements RocketMQListener<String> { ... }
```

> **坑**：顺序消费下若某条消息抛异常，会**阻塞该队列后续消息直到成功**，所以顺序消费的业务逻辑要轻量、避免长耗时。

### 9. 事务消息（中频，最终一致）

半消息 + 本地事务 + 回查：先发半消息 → 执行本地事务 → 提交/回滚；Broker 回调 `checkLocalTransaction` 回查拿最终结果。

```java
rocketMQTemplate.sendMessageInTransaction(TOPIC_TX, msg, orderId);
```
```java
executeLocalTransaction: localTxStore.mark(orderId, true); return COMMIT;   // 本地事务结果
checkLocalTransaction:   return localTxStore.get(orderId) ? COMMIT : ROLLBACK; // null 返回 UNKNOWN，稍后再查
```

> **坑**：回查务必**查库**（订单表/事务表）判定最终状态，而非依赖易失内存变量；状态未知返回 `UNKNOWN` 让 Broker 稍后再查，而不是猜。

### 10. 批量发送（中频）

`send(Collection<Message>)` 一次发送多条，减少网络往返。**坑**：单批总大小默认上限约 4MB，且同批次须发往同一 Topic。

### 11. 消息过滤（Tag 中频 / SQL92 低频）

- **Tag 过滤**：按 `Tag` 订阅，零额外配置，最常用。
- **SQL92 过滤**：按消息属性写类 SQL 表达式，**需 Broker 开启 `enablePropertyFilter=true`**，否则监听启动失败。

> **坑**：SQL92 依赖 Broker 开关，demo 用 `@Profile("sql92")` 隔离，避免拖垮 Tag 消费者。

### 12. 广播模式（中频）

消息发给**组内所有消费者**各一份（每个消费者独立消费）。**广播模式下消费失败默认不重试**，可配"广播不重试"仅记日志。适合每个实例都要全量拿到的场景（如配置下发）。

### 13. 请求-应答 RPC（中频）

基于 `request-reply` 语义实现 RPC 风格同步往返：生产者发消息并**阻塞等待**消费者回包。适合同步拿结果、不想另起同步调用的场景。

### 14. 主动拉取（LitePullConsumer，中频）

除默认 Push 消费外，可主动控制拉取节奏，适合批处理、限速、与业务循环耦合。demo 用 `LitePullConsumer` 演示。

### 15. ACL 鉴权（中频）

客户端配 `AccessKey` / `SecretKey` 走 ACL 鉴权，控制 Topic 级读写权限。demo 只演示客户端写法。

> **坑**：需 Broker 开启 ACL 并配好账号，否则报 `No permission`。（真实密钥不得落入公开仓库）

### 16. 消费限流调优（中频）

通过并发线程数、拉取批次、消费耗时等参数调优消费吞吐，防止瞬时洪峰压垮下游。

## 部署与若干硬坑

1. **Broker 地址注册**：Broker 启动会向 NameServer 注册"对外地址"。默认 bridge 网络注册的是**容器内 IP**，宿主机 Java 客户端连不上。规避：`network_mode: host`（直接以宿主机 IP 注册），或不支持 host 时用自定义网络 + `brokerIP1` 显式指定宿主机 IP。
2. **trace / DLQ / ACL 前置条件**：trace topic 需存在、ACL 需配账号、SQL92 需开 Broker 开关——这三类"依赖外部配置"的场景要提前声明。
3. **生产者组复用**：不同语义自建生产者时应自建所属组，避免与共享 `RocketMQTemplate` 冲突。

## 生产化清单（本 demo 未实现，学习方向）

- [ ] 幂等存储 `MessageIdStore` / 事务存储 `LocalTxStore`：内存实现 → Redis(SETNX+EX) / 数据库唯一键
- [ ] 部署拓扑：多 Broker 副本高可用、配 `brokerIP1` 保证可达
- [ ] 可观测：消息轨迹接入、死信队列告警、消费堆积监控
- [ ] 客户端参数：发送容错 + 消费并发/拉取批次按下游能力调优

## 参考来源

- [RocketMQ 官方文档](https://rocketmq.apache.org/)
- 关联工程：[jdk8-rocketmq-demo](https://github.com/xieqiansong/chaos-java/tree/main/jdk8-platform/jdk8-rocketmq-demo)