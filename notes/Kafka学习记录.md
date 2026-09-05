# Kafka 学习记录

工程依托：[jdk8-kafka-demo](https://github.com/xieqiansong/chaos-java/tree/main/jdk8-platform/jdk8-mq/jdk8-kafka-demo)（`chaos-java` 仓库，Spring Kafka，21 场景全带测试）。

## 1. 安装（KRaft 单节点）

用的是 `apache/kafka:latest` 官方镜像，KRaft 模式（不需要单独装 ZooKeeper），映射到宿主机 `30103`：

```yaml
services:
  kafka:
    image: apache/kafka:latest
    container_name: kafka
    ports:
      - "30103:9092"
    environment:
      # KRaft 模式配置
      - KAFKA_NODE_ID=1
      - KAFKA_PROCESS_ROLES=broker,controller
      - KAFKA_CONTROLLER_QUORUM_VOTERS=1@localhost:9093
      - KAFKA_LISTENERS=PLAINTEXT://:9092,CONTROLLER://:9093
      - KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://localhost:30103
      - KAFKA_LISTENER_SECURITY_PROTOCOL_MAP=PLAINTEXT:PLAINTEXT,CONTROLLER:PLAINTEXT
      - KAFKA_CONTROLLER_LISTENER_NAMES=CONTROLLER
      - KAFKA_INTER_BROKER_LISTENER_NAME=PLAINTEXT
      - KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR=1
      - KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR=1
      - KAFKA_TRANSACTION_STATE_LOG_MIN_ISR=1
      - KAFKA_LOG_DIRS=/var/lib/kafka/data
    volumes:
      - /data/kafka/data:/var/lib/kafka/data
    restart: unless-stopped
```

`docker-compose up -d` 起来就行。容器进程监听 `9092`，端口映射到宿主机 `30103`。

### 安装 Kafka 命令行客户端（在宿主机 ubuntu 上）

命令行脚本随 Kafka 发行包提供，下载解压即可（无需额外安装）：

```bash
cd ~/software
wget https://downloads.apache.org/kafka/4.3.1/kafka_2.13-4.3.1.tgz
tar -xzf kafka_2.13-4.3.1.tgz
cd kafka_2.13-4.3.1/bin
```

脚本在 `~/software/kafka_2.13-4.3.1/bin`，用 `.sh` 调用。因为 `30103` 已映射到宿主机，在 ubuntu 本机执行命令时连 `localhost:30103`（**不要连 `9092`**，那是容器内部端口，宿主机直连不通）。

> 注：命令行脚本必须在**运行 Kafka 的那台宿主机（ubuntu）**上执行；若在 Windows 等其他机器上跑，需把 `localhost` 换成 ubuntu 的 IP，且 Windows 下要用包里的 `bin/windows/*.bat`。

## 2. 简单收发

最基础的发和收：`KafkaTemplate.send` 发，`@KafkaListener` 收。

```java
// 发
kafkaTemplate.send("topic-simple", "hello");

// 收
@KafkaListener(topics = "topic-simple")
public void listen(String msg) {
    System.out.println("收到: " + msg);
}
```

```bash
# 对应 Kafka 指令（在宿主机 ubuntu 的 kafka/bin 下执行）
./kafka-console-producer.sh --bootstrap-server localhost:30103 --topic topic-simple
> hello

./kafka-console-consumer.sh --bootstrap-server localhost:30103 --topic topic-simple --from-beginning
```

## 3. 批量消费

要一次拿一批消息，得给 listener 配独立的 `ContainerFactory`，把 `setBatchListener(true)` 打开，否则还是一条条收。

```java
@Bean
public ConcurrentKafkaListenerContainerFactory<String, String> batchFactory(ConsumerFactory<String, String> cf) {
    ConcurrentKafkaListenerContainerFactory<String, String> f = new ConcurrentKafkaListenerContainerFactory<>();
    f.setConsumerFactory(cf);
    f.setBatchListener(true);          // 关键：开批量
    f.getContainerProperties().setAckMode(ContainerProperties.AckMode.BATCH);
    return f;
}

@KafkaListener(topics = "topic-batch", containerFactory = "batchFactory")
public void listen(List<String> msgs) {
    System.out.println("一批: " + msgs.size());
}
```

## 4. 消息过滤

消费端按条件扔掉不想要的。也可走 broker 端 `Header`/`Interceptor`，但消费端过滤最简单。

```java
@KafkaListener(topics = "topic-filter", containerFactory = "batchFactory")
public void listen(List<ConsumerRecord<String, String>> records) {
    records.stream()
        .filter(r -> r.value().contains("keep"))
        .forEach(r -> System.out.println("保留: " + r.value()));
}
```

```bash
# 对应 Kafka 指令（消费端过滤在代码里做，命令行只管收）
./kafka-console-consumer.sh --bootstrap-server localhost:30103 --topic topic-filter --from-beginning
```

## 5. 订单事件（事件驱动）

用一个 `OrderEvent` 模型把"下单"这件事作为事件发出来，消费者各自响应，体现解耦。

```java
// 发
kafkaTemplate.send("topic-order", new OrderEvent(orderId, amount));

// 收
@KafkaListener(topics = "topic-order")
public void onOrder(OrderEvent event) {
    System.out.println("处理订单: " + event.getOrderId());
}
```

```bash
# 对应 Kafka 指令
./kafka-console-producer.sh --bootstrap-server localhost:30103 --topic topic-order
> {"orderId":1,"amount":100}

./kafka-console-consumer.sh --bootstrap-server localhost:30103 --topic topic-order
```

## 6. 事务（exactly-once）

多条消息要原子地一起成功或一起失败，用 `executeInTransaction` 包起来。前提是 producer 开幂等（`enable.idempotence=true`）+ 配事务 id。消费者要设 `read_committed` 才只看已提交的消息。

```java
configs.put(ENABLE_IDEMPOTENCE_CONFIG, true);
DefaultKafkaProducerFactory<String, String> txFactory = new DefaultKafkaProducerFactory<>(configs);
txFactory.setTransactionIdPrefix("tx-demo-");
KafkaTemplate<String, String> txTemplate = new KafkaTemplate<>(txFactory);

txTemplate.executeInTransaction(ops -> {
    ops.send("topic-tx", "msg1");
    if (shouldFail) throw new RuntimeException("模拟异常");
    ops.send("topic-tx", "msg2");
    return null;
});
```

> 跟 RocketMQ 事务消息不是一回事：Kafka 是生产者侧事务（多分区原子写 + 只读已提交），Broker 不回查；RocketMQ 是半消息 + 本地事务回查，走最终一致。

```bash
# 对应 Kafka 指令（事务消息可用命令行观察）
# 仅消费已提交消息（事务相关）：加 --isolation-level read_committed
./kafka-console-consumer.sh --bootstrap-server localhost:30103 --topic topic-tx --isolation-level read_committed
```

## 7. 重试与死信（DLT）

消费报错想重试、重试耗尽进死信队列，给 container 注入 `DefaultErrorHandler` + `DeadLetterPublishingRecoverer`。DLT 目标 topic 必须显式指定（默认 `<topic>.DLT`），不然死信收不到。

```java
@Bean
public DefaultErrorHandler errorHandler(KafkaTemplate<String, String> kt) {
    DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(
        kt, (record, ex) -> new TopicPartition("topic-retry.DLT", record.partition()));
    return new DefaultErrorHandler(recoverer, new FixedBackOff(500L, 1L)); // 重试 1 次后进 DLT
}
```

```bash
# 对应 Kafka 指令（死信 topic 独立存在，可直接消费查看）
./kafka-console-consumer.sh --bootstrap-server localhost:30103 --topic topic-retry.DLT --from-beginning
```

## 参考来源

- 工程：[jdk8-kafka-demo](https://github.com/xieqiansong/chaos-java/tree/main/jdk8-platform/jdk8-mq/jdk8-kafka-demo)
- Spring Kafka 官方文档（错误处理、`KafkaAdmin` 自动建 Topic）
