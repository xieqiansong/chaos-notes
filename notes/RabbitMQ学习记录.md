# RabbitMQ 学习记录

工程依托：[jdk8-rabbitmq-demo](https://github.com/xieqiansong/chaos-java/tree/main/mq/rabbitmq-demo)（`chaos-java` 仓库，Spring Boot 2.7.18 + Spring AMQP 2.4.x + RabbitMQ 3.12，双测试层：rabbitmq-mock 内存 Broker 自包含 + Testcontainers 真实 Broker 无 Docker 自动跳过）。

> 与 [`Kafka 学习记录`](Kafka学习记录.md)、[`RocketMQ 学习记录`](RocketMQ学习记录.md) 互补：RabbitMQ 偏「Broker 侧路由 + 复杂投递语义 + 推模式」，本篇聚焦它的差异点。

## 1. 安装 / 部署

`docker-compose.yml` 用 `rabbitmq:3.12-management`，映射 5672（AMQP）/ 15672（管理界面，guest/guest）。

```bash
cd jdk8-rabbitmq-demo && docker compose up -d
mvn -pl jdk8-rabbitmq-demo spring-boot:run
# 浏览器 http://localhost:15672 看拓扑与消息流动
```

纯测试无需中间件：`mvn -pl jdk8-rabbitmq-demo -am test`（内存 Broker 自包含必过）。

## 2. Exchange 四种类型路由（★★★）

路由是 RabbitMQ 的立身之本，由 Broker 在**服务端**完成，消费者收不到非匹配消息（省带宽）。

- **Direct**：`routingKey` 完全相等才投递，点对点。
- **Topic**：`order.*` / `log.*` / `#` 通配，`order.created` 同时命中 `order.*` 与 `#`。
- **Fanout**：忽略 routingKey，广播到所有绑定队列（清缓存、事件通知）。
- **Headers**：按消息 `header` 匹配（忽略 routingKey），`where("type").equals("report")`。

```java
// Topic 示例：声明 + 绑定 + 发
rabbitTemplate.convertAndSend("demo.topic.exchange", "order.created", event);
// orders 队列绑 order.*，logs 队列绑 log.*，all 队列绑 # → order.created 进 orders 与 all
```

## 3. 生产者确认 Publisher Confirm（★★★）

开启 `publisherConfirms=true`，每条消息有 `deliveryTag`，Broker 落盘/路由成功后回 `confirm`。

```java
rabbitTemplate.invoke(t -> {
    rabbitTemplate.convertAndSend(exchange, key, msg);
    return t.waitForConfirms(timeout);   // 阻塞等全部确认
});
```

坑：confirm 只保证「到 Broker」，不保证「被消费」；路由失败也会 confirm 成功。要感知不可路由，配 `publisher-returns` / `mandatory`。

## 4. 消费者手动 Ack + nack 重入队（★★★）

容器 `AcknowledgeMode.MANUAL`，业务处理完 `basicAck`；失败 `basicNack(requeue=true)` 重入队重试，或 `requeue=false` 进死信。

```java
@RabbitListener(queues = "demo.ack.queue", ackMode = "MANUAL")
public void on(Message msg, Channel ch, @Header(AmqpHeaders.DELIVERY_TAG) long tag) throws IOException {
    try { handle(msg); ch.basicAck(tag, false); }
    catch (Exception e) { ch.basicNack(tag, false, true); }  // 重入队重试
}
```

## 5. TTL + DLX 死信（★★★）

工作队列声明 `x-message-ttl` + `x-dead-letter-exchange` + `x-dead-letter-routing-key`，消息超时**由 Broker 自动死信**到 DLT——无需业务代码抛异常（与 Kafka 代码层重试的本质区别）。

```java
// 延迟消息：缓冲队列(TTL=延迟时长) + DLX → 目标队列 = 延迟 N ms 送达
new Queue("demo.delay.buffer", true, false, false,
    Map.of("x-message-ttl", 1500, "x-dead-letter-exchange", "demo.delay.dlx",
           "x-dead-letter-routing-key", "demo.delay.target"));
```

局限：队列级 TTL 对所有消息同延迟；单条不同延迟需消息级 `expiration` 或延迟插件 `rabbitmq-delayed-message-exchange`。

## 6. 消息可靠性专题：不丢 / 不重 / 有序（★★★）

面试高频「消息怎么保证不丢、不重、有序」，本 demo 把三件事拆开：

| 维度 | 应用层（可演示） | 中间件层（说明） |
|---|---|---|
| 不丢·生产→Broker | Publisher Confirm（`waitForConfirms`） | `publisher-confirms` + 队列 `durable` + 消息 `PERSISTENT` + `publisher-returns` |
| 不丢·Broker→消费 | 手动 Ack（处理完才 ack，失败 nack 重入队） | 消费者 `MANUAL` + 仲裁队列（quorum）防单点 |
| 不重（幂等） | `orderId` 业务键去重，重复投递仅处理一次 | 生产用 Redis `SETNX+EX` / 数据库唯一键 |
| 有序 | 单队列天然 FIFO（默认行为） | 多消费者 + `prefetch>1` 破坏严格顺序；严格有序用「单队列+单消费者」 |

本质话术：不丢 = 确认机制 + 持久化 + 高可用队列；不重 = 消费端业务幂等（RabbitMQ 是 at-least-once）；有序 = 一个 queue 内本就 FIFO。

## 踩坑

- **confirm 不等于送达消费端**：只到 Broker，路由失败也 confirm 成功。
- **DLT 必须配告警/补偿**：否则死信积压无感知。
- **队列级 TTL 乱序**：早消息阻塞晚消息，单条延迟用消息级 `expiration` 或延迟插件。
- **多消费者 + prefetch>1 破坏顺序**：严格有序要单队列单消费者。

## 进阶方向

- 延迟插件 `rabbitmq-delayed-message-exchange`（单条任意延迟）。
- 优先级队列 `x-max-priority`、惰性队列 `x-queue-mode=lazy` 抗积压。
- 仲裁队列（quorum，替代 classic mirrored queue）高可用。
- 与 Kafka 选型：RabbitMQ 推模式、低延迟、Broker 侧路由/死信强；Kafka 拉模式、高吞吐、分区有序/日志重放。

## 参考来源

- 工程：[jdk8-rabbitmq-demo](https://github.com/xieqiansong/chaos-java/tree/main/mq/rabbitmq-demo)
- [RabbitMQ 官方文档](https://www.rabbitmq.com/documentation.html)
- [Spring AMQP 文档](https://docs.spring.io/spring-amqp/reference/html/)
