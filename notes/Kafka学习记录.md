# Kafka 学习记录

> 工程依托：`chaos-java/jdk8-platform/jdk8-kafka-demo`（Spring Kafka，21 场景全带测试）。
> 记录 Kafka 的核心机制、关键 API 与学习踩坑点。配置/事务/重试部分直接对应 demo 代码。

## 1. 消息模型与核心概念

- **Topic / Partition / Offset**：Topic 是逻辑主题，物理上按 Partition 分片，每条消息在分区内有序、有唯一 offset；消费进度按 `(group, partition)` 维护。
- **Producer / Consumer / Consumer Group**：同一 group 内消费者分摊分区（一个分区同一时刻只被一个消费者持有）；不同 group 互不影响，各自独立消费。
- **位移提交**：消费语义取决于提交时机——自动提交可能「消息未处理就提交」导致丢消息，手动提交可控但需防重复消费。

## 2. 场景实践（对应 demo）

| 场景 | 类 | 要点 |
|---|---|---|
| 简单收发 | `SimpleProducer/Consumer` | 模板方法 `KafkaTemplate.send` + `@KafkaListener` |
| 批量消费 | `BatchConsumer` | 独立 `ContainerFactory`（`setBatchListener(true)` + `AckMode.BATCH`） |
| 消息过滤 | `FilterConsumer` | 消费端过滤（也可走 broker 端 `Header`/`Interceptor`） |
| 订单事件 | `OrderProducer/Consumer` | 用 `OrderEvent` 模型串联，体现「事件驱动」 |
| 事务 | `TransactionProducer` | exactly-once 语义 |
| 重试/死信 | `RetryConsumer` | `DefaultErrorHandler` + DLT |

## 3. 事务生产者（exactly-once）

`TransactionProducer` 用 `executeInTransaction` 把多条消息包成原子事务：

```java
configs.put(TRANSACTIONAL_ID_CONFIG, "tx-demo-producer");
configs.put(ENABLE_IDEMPOTENCE_CONFIG, true);          // 幂等是事务前提
DefaultKafkaProducerFactory<String, String> txFactory = new DefaultKafkaProducerFactory<>(configs);
txFactory.setTransactionIdPrefix("tx-demo-");
KafkaTemplate<String, String> txKafkaTemplate = new KafkaTemplate<>(txFactory);

txKafkaTemplate.executeInTransaction(ops -> {
    ops.send(TOPIC_TRANSACTION, key, msg1);
    if (shouldFail) throw new RuntimeException("模拟业务异常");
    ops.send(TOPIC_TRANSACTION, key, msg2);
    return null;
});
```

**要点**：
- 事务保证「跨分区/跨 Topic 的原子写入」，消费者需 `read_committed` 才只读到已提交消息，实现 exactly-once。
- 依赖 producer **幂等**（`enable.idempotence=true`）+ 事务 id。
- **与 RocketMQ 事务消息差异**：Kafka 是生产者侧事务（多分区原子写 + 仅读已提交），无服务端回查；RocketMQ 是半消息 + 本地事务回查，强调分布式事务最终一致。

## 4. 自动建 Topic 的设计取舍

`KafkaConfig` 用 `KafkaAdmin.NewTopics` 在代码中自动建 Topic（单分区、无副本）：

```java
@Bean
public KafkaAdmin.NewTopics demoTopic() {
    return new KafkaAdmin.NewTopics(
        topic(TOPIC_SIMPLE), topic(TOPIC_BATCH), /* ... */);
}
private NewTopic topic(String name) {
    return TopicBuilder.name(name).partitions(1).replicas(1).build();
}
```

**取舍**：Demo 场景避免依赖外部 `kafka-topics.sh`；**生产环境 Topic 多由运维统一管理，不应在业务代码里自动创建**。

## 5. 重试与死信（DLT）

`retryContainerFactory` 注入 `DefaultErrorHandler` + `DeadLetterPublishingRecoverer`：

```java
DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(
    kafkaTemplate,
    (record, ex) -> new TopicPartition(TOPIC_RETRY_DLT, record.partition())); // 显式指定 DLT
DefaultErrorHandler errorHandler = new DefaultErrorHandler(recoverer, new FixedBackOff(500L, 1L)); // 重试 1 次后进 DLT
```

**坑点**：
- Spring Kafka 错误处理器演进：`2.x` 用 `SeekToCurrentErrorHandler`（seek 重放），`2.8+` 用 `DefaultErrorHandler`（支持 `BackOff` + 自动 DLT）。
- DLT 目标 topic 必须**显式指定**，默认规则是 `<topic>.DLT`，否则收不到死信。
- 重试间隔 `FixedBackOff(500L, 1L)` 表示重试 1 次（共 2 次投递），教学足够。

## 6. 学习踩坑点

- **事务 template 不要注册成全局 bean**：Spring Boot 的 `@ConditionalOnMissingBean(KafkaTemplate.class)` 一旦发现任意 `KafkaTemplate` bean 就跳过默认非事务 template 的创建，会让 simple/batch 等场景全变成事务化。所以事务 template 在 `TransactionProducer` 内部自建。
- **分区数决定并发度**：单分区主题同一 group 只能有 1 个消费者干活，想提并发先加分区。
- **批量消费需独立 factory**：`setBatchListener(true)` 影响整个 container，必须隔离成专用 `batchContainerFactory`。

## 7. 参考来源

- 工程：`chaos-java/jdk8-platform/jdk8-kafka-demo`
- Spring Kafka 官方文档（错误处理、`KafkaAdmin` 自动建 Topic）
