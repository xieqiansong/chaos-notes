# RocketMQ 学习记录

工程依托：[jdk8-rocketmq-demo](https://github.com/xieqiansong/chaos-java/tree/main/jdk8-platform/jdk8-mq/jdk8-rocketmq-demo)（`chaos-java` 仓库，覆盖 RocketMQ 常用能力维度的 Spring Boot Demo，所有场景由 `DemoTest` 统一触发）。

版本：`rocketmq-client 4.9.8` + `rocketmq-spring-boot-starter 2.2.3` + `spring-boot 2.7.18` + JDK 8。

先说结论：
- **at-least-once 投递**：只保证"至少一次"，重复消费必然发生，消费端幂等是义务。
- **Topic / ConsumerGroup 是契约字符串**：生产/消费端拼错不报编译错，运行期静默收不到，务必收口到常量（demo 的 `MqConstant`）。
- **投递可靠性**：同步 > 异步 > 单向（吞吐反之）。

## 1. 安装

`apache/rocketmq:5.4.0`，包含 NameServer + Broker + Proxy + Dashboard 四件套。NameServer `9876`，Broker `10909/10911/10912`，Proxy `8080/8081`（5.x 客户端走 Proxy），Dashboard `8082`（容器内 `8080`）。

`docker-compose.yaml`：

```yaml
services:
  namesrv:
    image: apache/rocketmq:5.4.0
    container_name: rmqnamesrv
    ports:
      - "9876:9876"
    command: sh mqnamesrv
    networks:
      - rocketmq

  broker:
    image: apache/rocketmq:5.4.0
    container_name: rmqbroker
    ports:
      - "10909:10909"
      - "10911:10911"
      - "10912:10912"
    environment:
      - NAMESRV_ADDR=rmqnamesrv:9876
    volumes:
      - /data/rocketmq/conf/broker.conf:/home/rocketmq/conf/broker.conf
    depends_on:
      - namesrv
    networks:
      - rocketmq
    command: sh mqbroker -c /home/rocketmq/conf/broker.conf

  proxy:
    image: apache/rocketmq:5.4.0
    container_name: rmqproxy
    ports:
      - "8080:8080"
      - "8081:8081"
    environment:
      - NAMESRV_ADDR=rmqnamesrv:9876
    depends_on:
      - broker
      - namesrv
    networks:
      - rocketmq
    command: sh mqproxy
  dashboard:
    image: apacherocketmq/rocketmq-dashboard:latest
    container_name: rmqdashboard
    ports:
      - "8082:8082"
    environment:
      - "JAVA_OPTS=-Drocketmq.namesrv.addr=rmqnamesrv:9876"
    depends_on:
      - namesrv
    networks:
      - rocketmq

networks:
  rocketmq:
    driver: bridge
```

`conf/broker.conf`：

```properties
# 集群配置
brokerClusterName=DefaultCluster
brokerName=broker-a
brokerId=0
brokerRole=ASYNC_MASTER
flushDiskType=ASYNC_FLUSH

# IP 与端口
# 你的宿主机IP（如 192.168.x.x 或 host.docker.internal）
brokerIP1=192.168.x.x
listenPort=10911

# 存储路径
storePathRootDir=/home/rocketmq/store
logRootDir=/home/rocketmq/logs

# 清理策略
deleteWhen=04
fileReservedTime=48

# 功能开关（学习环境建议开启，生产环境关闭）
autoCreateTopicEnable=true
autoCreateSubscriptionGroup=true

enablePropertyFilter=true
```

`docker-compose up -d` 起来。Dashboard 控制台 `http://<ubuntu-ip>:8082`。

- Java 客户端 / Proxy 走 `localhost:8080`（5.x 推荐）。
- `mqadmin` 命令连 NameServer `localhost:9876`（需在 ubuntu 装 rocketmq 客户端，脚本在 `bin/`）。
- 注意 `brokerIP1=192.168.x.x` 要填宿主机真实 IP，否则客户端连不上 Broker（bridge 网络默认注册容器内 IP）。

## 2. 基础收发：同步 / 异步 / 单向

| 语义 | API | 特点 |
|---|---|---|
| 同步 | `syncSend` | 阻塞等确认，可靠性最高，务必处理异常 |
| 异步 | `asyncSend` + `SendCallback` | 不阻塞，回调处理结果 |
| 单向 | `sendOneWay` | 只发不收，吞吐最大，适合日志 |

```java
// 同步：阻塞等确认，可靠性最高
rocketMQTemplate.syncSend(MqConstant.TOPIC_BASIC, payload);

// 单向：只发不收，吞吐最大
rocketMQTemplate.sendOneWay(MqConstant.TOPIC_BASIC, MessageUtils.pack(body));
```

```bash
# 对应 RocketMQ 指令（mqadmin，在 ubuntu 宿主机执行，连 namesrv 9876）
# 查看 topic 列表
mqadmin topicList -n localhost:9876

# 发一条测试消息
mqadmin sendMessage -n localhost:9876 -t TOPIC_BASIC -p "hello"
```

## 3. 幂等消费（必做）

用 Broker 保证全局唯一的 `msgId` 去重：处理前查是否已处理，处理后标记。

```java
@RocketMQMessageListener(topic = MqConstant.TOPIC_BASIC, consumerGroup = MqConstant.GROUP_BASIC)
public class IdempotentConsumer implements RocketMQListener<MessageExt> {
    public void onMessage(MessageExt message) {
        String msgId = message.getMsgId();
        if (messageIdStore.isProcessed(msgId)) { return; } // 重复消息跳过
        // ... 业务处理 ...
        messageIdStore.markProcessed(msgId);
    }
}
```

```bash
# 对应 RocketMQ 指令（查消费组进度，验证消费情况）
mqadmin consumerProgress -n localhost:9876 -g GROUP_BASIC
```

> 坑：内存去重重启/多实例失效，生产换 Redis(SETNX+EX) 或 DB 唯一键；先处理成功再 `markProcessed`。

## 4. 重试与死信

消费失败自动重试（约 16 次），耗尽进死信 Topic `%DLQ% + 原消费组名`，必须由全新消费组消费。

```java
@RocketMQMessageListener(topic = MqConstant.DLQ_TOPIC_RETRY, consumerGroup = MqConstant.GROUP_DLQ_HANDLER)
public class DeadLetterConsumer implements RocketMQListener<String> {
    public void onMessage(String message) {
        // 人工补偿：落补偿表 / 告警 / 人工审核
    }
}
```

```bash
# 对应 RocketMQ 指令（查死信 topic 是否存在）
mqadmin topicList -n localhost:9876 | grep DLQ
```

## 5. 延迟消息

RocketMQ 原生固定延迟级别（`1s/5s/10s/.../2h`），只能选内置档位，不是任意秒数。

```java
rocketMQTemplate.syncSendDelayTimeMillis(MqConstant.TOPIC_DELAY, payload, 1000);
```

```bash
# 对应 RocketMQ 指令（无专用命令，在 Dashboard :8082 的「延迟消息」页查看）
# 或发带延迟级别的测试消息
mqadmin sendMessage -n localhost:9876 -t TOPIC_DELAY -p "delay" -d 1
```

## 6. 事务消息（半消息 + 回查，最终一致）

先发半消息 → 执行本地事务 → 提交/回滚；Broker 回查 `checkLocalTransaction` 拿最终结果。回查务必查库判定，未知返回 `UNKNOWN`。

```java
rocketMQTemplate.sendMessageInTransaction(TOPIC_TX, msg, orderId);
// executeLocalTransaction: 标记本地事务结果，返回 COMMIT
// checkLocalTransaction:   查库返回 COMMIT / ROLLBACK / UNKNOWN
```

```bash
# 对应 RocketMQ 指令（查事务处于半消息/回查状态的消息）
mqadmin queryMsgById -n localhost:9876 <msgId>
```

## 7. 顺序消息（分区顺序）

相同 `shardingKey`（如 `orderId`）路由同一队列，局部有序；消费用 `ORDERLY` 模式单队列串行。

```java
@RocketMQMessageListener(topic = TOPIC_ORDER, consumerGroup = GROUP_ORDER, consumeMode = ConsumeMode.ORDERLY)
public class OrderedConsumer implements RocketMQListener<String> { ... }
```

```bash
# 对应 RocketMQ 指令（查队列分布，确认同 key 落到同队列）
mqadmin topicRoute -n localhost:9876 -t TOPIC_ORDER
```

## 8. 消息过滤（Tag 常用 / SQL92 需开开关）

Tag 过滤零配置最常用；SQL92 按消息属性写类 SQL 表达式，**需 Broker 开 `enablePropertyFilter=true`**（你 broker.conf 已开）。

```java
// Tag 订阅：selectorExpression = "TagA"
@RocketMQMessageListener(topic = TOPIC_FILTER, consumerGroup = GROUP_FILTER, selectorExpression = "TagA")
public class TagConsumer implements RocketMQListener<String> { ... }
```

```bash
# 对应 RocketMQ 指令（确认 broker 是否开启属性过滤）
mqadmin getBrokerConfig -n localhost:9876 -b 192.168.x.x:10911 | grep enablePropertyFilter
```

## 9. 其余能力（一笔带过）

- **发送容错**：`setRetryTimesWhenSendFailed(3)` / `setSendLatencyFaultEnable(true)` 避峰切健康节点。
- **消息轨迹**：4.9.8 的 `DefaultMQProducer` 无 `setEnableMsgTrace`，只能构造函数 `new DefaultMQProducer(GROUP_TRACE, null, true, TRACE_TOPIC)` 开启，trace topic 须已存在。
- **批量发送**：`send(Collection<Message>)`，单批上限约 4MB，同批次须同一 Topic。
- **广播模式**：组内每人各一份，失败默认不重试，适合配置下发。
- **请求-应答 / 主动拉取(LitePullConsumer) / ACL 鉴权 / 消费限流调优**：按业务/治理选用。

## 10. 几个硬坑

- **Broker 注册地址**：bridge 网络默认注册容器内 IP，宿主机客户端连不上；用 `brokerIP1` 显式指定宿主机 IP（你已配 `192.168.x.x`）。
- **trace / DLQ / SQL92 前置条件**：trace topic 需存在、SQL92 需开 Broker 开关（你已开）、ACL 需配账号。
- **生产者组复用**：不同语义自建生产者应自建所属组，避免与共享 `RocketMQTemplate` 冲突。
- **幂等存储生产化**：内存实现重启/多实例失效，换 Redis 或 DB 唯一键。

## 参考来源

- [RocketMQ 官方文档](https://rocketmq.apache.org/)
- 关联工程：[jdk8-rocketmq-demo](https://github.com/xieqiansong/chaos-java/tree/main/jdk8-platform/jdk8-mq/jdk8-rocketmq-demo)
