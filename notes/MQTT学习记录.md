# MQTT 学习记录

工程依托：[jdk8-mqtt-demo](https://github.com/xieqiansong/chaos-java/tree/main/mq/mqtt-demo)（`chaos-java` 仓库，JDK 8 + Spring Boot 2.7.18 + Eclipse Paho `org.eclipse.paho.client.mqttv3` 1.2.5，moquette 内存 Broker 自包含 + Testcontainers mosquitto 无 Docker 自动跳过）。

> MQTT 是「设备接入协议」，与 [`Kafka 学习记录`](Kafka学习记录.md)、[`RabbitMQ 学习记录`](RabbitMQ学习记录.md) 不在一个层：通常前端接 EMQX/mosquitto 做协议转换，再桥接到 Kafka/RabbitMQ 做后端业务。本篇聚焦它自身的语义。

## 1. 安装 / 部署

`docker-compose.yml` 用 `eclipse-mosquitto:2.0.18`。纯测试无需中间件：`mvn -pl jdk8-mqtt-demo -am test`（moquette 内存 Broker 自包含）。

```bash
cd jdk8-mqtt-demo && docker compose up -d
mvn -pl jdk8-mqtt-demo spring-boot:run
```

## 2. 基础发布订阅（★★★）

核心是「**发布者 → Broker（按主题路由）→ 订阅者**」的解耦模型，生产者与消费者只在「主题」上达成共识，彼此不知道对方地址/在线状态。

```java
MqttClient pub = MqttClients.create("tcp://localhost:1883", "pub-1");
pub.connect();
pub.publish("demo/sensors/temp", new MqttMessage("{\"v\":25}".getBytes()));
// 订阅者：client.subscribe("demo/sensors/temp", (t, m) -> System.out.println(new String(m.getPayload())));
```

差异：Kafka 偏日志流/分区重放、RabbitMQ 偏 Broker 侧复杂投递；MQTT 极轻量、面向「海量设备 ↔ 少量服务端」的遥测/指令，主题即路由，几乎没有 Broker 侧拓扑配置。

## 3. 通配符订阅 `+` 与 `#`（★★★）

主题用 `/` 分层：`+` 匹配**单层**、`#` 匹配**多层**。

- `demo/sensors/+/temp` 命中 `room1/temp`、`room2/temp`，**不命中** `room1/humidity`（层数不对）。
- `demo/sensors/#` 命中上述全部。

典型用途：温度监控订阅 `+/temp`；总控面板订阅 `#` 收全部遥测。

## 4. QoS 服务质量等级（★★★）

| QoS | 语义 | 确认机制 | 风险 |
|---|---|---|---|
| 0 | 至多一次 | 无确认，fire-and-forget | 可能丢 |
| 1 | 至少一次 | PUBACK | 不丢但**可能重复** |
| 2 | 恰好一次 | PUBREC/PUBREL/PUBCOMP 四段握手 | 不丢不重，最重 |

坑（面试常踩）：QoS 是「发布→Broker→订阅」**两段独立协商，取两者较小值生效**。发布 QoS2、订阅 QoS0，最终订阅者只按 QoS0 收到（可能丢）。

## 5. 保留消息 Retained（★★☆）

带 `retained=true` 的消息被 Broker **按主题留存最后一份**；之后才上线的新订阅者，订阅瞬间**立即收到**这份留存——不必苦等下一次发布。

典型用途：设备「在线/离线」状态、配置快照——新面板一进来就能看到当前值。坑：每主题一份，新发布会覆盖；清除以「发布一条**空载荷**的 retained 消息」实现。

## 6. 遗嘱消息 LWT（★★☆）

客户端连接时向 Broker 注册一条「遗言」；当客户端**异常断开**（网络断、进程崩、socket 被强行关闭，**不含**正常 DISCONNECT）时，Broker 自动把遗嘱发布到指定主题。

```java
MqttConnectOptions opt = new MqttConnectOptions();
opt.setWill("demo/status", "offline".getBytes(), 0, false);  // 注册遗嘱
client.connect(opt);
// 异常断开（MqttAsyncClient.close(true) 掐断 socket 且不发 DISCONNECT）→ Broker 代发 offline
```

典型用途：物联网设备掉线告警——连上时注册 `will=offline`，一掉线监控端立刻看到 offline。注意正常 `disconnect()` 会告知 Broker 主动离开，不会触发遗嘱。

## 踩坑

- **QoS 取小值生效**：发布/订阅两端 QoS 不一致时按低的来，别以为发了 QoS2 就一定恰好一次。
- **保留消息会覆盖**：每主题只留一份最新，清掉要发空载荷 retained。
- **LWT 只在异常断开触发**：正常 `disconnect()` 不发遗嘱，测试时要 force 掐断 socket。
- **安全**：生产务必开 TLS + `username/password` 认证 + `acl_file` 主题级授权，杜绝匿名。

## 进阶方向

- 会话与持久化订阅（`cleanSession=false` + 固定 clientId）：离线期间消息重连补发，面向「离线补偿」而非「最新值」。
- 共享订阅 `$share/group/topic`：订阅端负载均衡，类比 Kafka 消费组。
- MQTT 5 vs 3.1.1：Reason Code、User Property、消息过期、Topic Alias 等。
- 与 Kafka/RabbitMQ 分层：MQTT 做设备接入，桥接后端做业务，不是替代关系。

## 参考来源

- 工程：[jdk8-mqtt-demo](https://github.com/xieqiansong/chaos-java/tree/main/mq/mqtt-demo)
- [MQTT 3.1.1 协议规范](https://mqtt.org/mqtt-specification/)
- [Eclipse Paho](https://www.eclipse.org/paho/)
