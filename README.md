# chaos-notes

> 技术笔记仓库：**性能优化** + **踩坑记录** + **技术学习笔记**。Chaos 本意保持混乱，把精力集中在当前要做的事上。

我的开源技术笔记集中地，聚焦三条主线：

## 内容主线

### 1. 性能优化 (Performance)

主动调优 / 压测 / 基准的实战沉淀，重视过程分析与量化对比（延迟、吞吐、负载、精度）。

> 定位：问题 → 分析 → 优化 → 效果（带量化数据），与「踩坑记录」以「性能问题排查修复」划界。

| 笔记 | 主题 | 时间 |
|---|---|---|
| [虚拟线程 vs 平台线程：IO 密集压测量化与落地边界](performance/虚拟线程vs平台线程-IO密集压测与落地边界.md) | 虚拟线程 / Tomcat 替换、并行编排、pinning、适用边界 | 2026-08 |
| [TCP-over-WebSocket 隧道：端到端性能基准与瓶颈分析](performance/TCP-over-WebSocket隧道-端到端性能基准与瓶颈分析.md) | 网络隧道 / 基准可信化、每字节开销瓶颈、边界结论 | 2026-08 |
| [多租户分布式限流：本地+Redis 双层限流](performance/多租户分布式限流-本地加Redis双层限流.md) | 限流 / 本地+Redis、精度换性能、压测 | 2024-02 |
| [Redis 批量入库：自适应批量大小](performance/Redis批量入库-自适应批量大小.md) | 批量入库 / 攒批、在线寻优、命令量降千倍 | 2023-09 |
| [秒杀链路压测与优化](performance/秒杀链路压测与优化.md) | 秒杀 / Lua 分桶扣减、Kafka 削峰、双层限流压测 | 2026-08 |
| [短链缓存命中率与防穿透压测](performance/短链缓存命中率与防穿透压测.md) | 短链 / 布隆过滤器防穿透、缓存命中率压测 | 2026-08 |
| [热路径 Servlet Filter 异步化：高频接口绕过 SpringMVC](performance/热路径Filter异步化-高频接口绕过SpringMVC.md) | 网关/接入层 / 提前返回、绕过 MVC、独立线程池异步落库 | 2026-09 |

### 2. 踩坑记录 (Pitfalls)

记录真实踩过的 BUG、配置坑与排查过程。采用「场景 → 原因 → 解决」模板，方便回查与复用。

| 踩坑 | 主题 | 时间 |
|---|---|---|
| [Linux 高负载：nf_conntrack 连接跟踪表溢出](pitfalls/Linux高负载nf_conntrack连接跟踪表溢出.md) | 网络 / conntrack 调优 | 2024-04 |
| [ES 集群节点版本不一致导致数据不同步](pitfalls/ES集群节点版本不一致导致数据不同步.md) | 搜索引擎 / 节点版本 | 2023-01 |
| [MySQL 5.5 共享表空间改独立表空间](pitfalls/MySQL共享表空间改独立表空间.md) | InnnoDB 表空间 / 数据重建 | 2022-06 |


### 3. 技术学习 (Notes)

中间件部署、示例代码、现有框架整合等学习沉淀。覆盖 Kafka / Redis / RocketMQ / Nacos / Seata 等。

| 笔记 | 主题 | 时间 |
|---|---|---|
| [Nacos 学习记录](notes/Nacos学习记录.md) | 注册中心 / 配置中心、动态刷新、编程式监听 | 2026-07-18 |
| [Seata 学习记录](notes/Seata学习记录.md) | 分布式事务 / AT 与 TCC 模式、undo_log、补偿 | 2026-07-19 |
| [Redis 学习记录](notes/Redis学习记录.md) | 缓存 / 分布式锁、限流、排行榜、管道、发布订阅 | 2026-07-19 |
| [Kafka 学习记录](notes/Kafka学习记录.md) | 消息队列 / 事务 exactly-once、批量消费、重试死信、自动建 Topic | 2026-07-15 |
| [RocketMQ 学习记录](notes/RocketMQ学习记录.md) | 消息队列 / 收发、幂等、重试死信、事务、轨迹 | 2026-07-15 |
| [高并发秒杀实战](notes/高并发秒杀实战.md) | 场景总结 / 限流+Lua 扣减+Kafka 削峰、要点 | 2026-07-02 |
| [短链系统设计](notes/短链系统设计.md) | 场景总结 / Snowflake+Base62、布隆防穿透、要点 | 2026-07-01 |
| [企业级微服务架构实践](notes/企业级微服务架构实践.md) | 场景总结 / SCA P0-P6 路线图、分布式事务/限流/鉴权 | 2026-05 |
| [JDK8 内核与并发内功](notes/JDK8内核与并发内功.md) | 学习进度 / JUC/JVM/Agent/Paxos、能力地图 | 2026-05-02 |
| [JDK8 到 25 新特性演进](notes/JDK8到25新特性演进.md) | 学习进度 / 虚拟线程、密封类、版本跟进 | 2026-04-15 |
| [MySQL UDF 插件实现 SM4 加解密](notes/MySQL-UDF插件实现SM4加解密.md) | MySQL 插件 / C 编写 UDF、SM4 ECB 加解密、编译与注册 | 2024-05-26 |
| [本地缓存学习记录](notes/本地缓存学习记录.md) | 本地缓存 / Caffeine、TTL、W-TinyLFU、@Cacheable | 2026-07-20 |
| [Sentinel 学习记录](notes/Sentinel学习记录.md) | 流控 / QPS·WarmUp·关联、熔断降级、热点参数、@SentinelResource | 2026-08-24 |
| [Elasticsearch 学习记录](notes/Elasticsearch学习记录.md) | 搜索引擎 / 索引、文档、搜索、聚合 | 2026-07-21 |
| [ZooKeeper 学习记录](notes/ZooKeeper学习记录.md) | 协调 / 分布式锁、Leader 选举、配置中心+Watcher | 2026-08-20 |
| [Spring Security 学习记录](notes/SpringSecurity学习记录.md) | 认证授权 / 过滤器链、JWT、OAuth2 资源服务器 | 2026-08-24 |
| [MyBatis-Plus 学习记录](notes/MyBatis-Plus学习记录.md) | ORM / Wrapper、分页、逻辑删除+乐观锁、多租户、动态表名、AES | 2026-07-23 |
| [加密与国密学习记录](notes/加密与国密学习记录.md) | 加密 / AES、RSA、SHA-256、SM2/SM3/SM4 | 2026-07-22 |
| [序列化学习记录](notes/序列化学习记录.md) | 序列化 / JDK 原生、Jackson、Kryo 对比 | 2026-07-22 |
| [定时任务学习记录](notes/定时任务学习记录.md) | 调度 / @Scheduled、Quartz、XXL-JOB | 2026-07-22 |
| [MapStruct 学习记录](notes/MapStruct学习记录.md) | 对象映射 / 基础、集合、嵌套、自定义映射 | 2026-08-29 |
| [Spring Boot Starter 学习记录](notes/SpringBootStarter学习记录.md) | 自动装配 / @ConfigurationProperties、条件装配、命名约定 | 2026-08-20 |
| [单元测试学习记录](notes/单元测试学习记录.md) | 测试 / Mockito、参数匹配、行为验证、切片测试 | 2026-07-22 |
| [PDF 处理学习记录](notes/PDF处理学习记录.md) | Office / PDFBox 绘制、字体嵌入、提取、合并拆分 | 2026-09-04 |
| [Excel 处理学习记录](notes/Excel处理学习记录.md) | Office / POI、EasyExcel、Hutool、大文件流式、横评 | 2026-09-04 |
| [Word 处理学习记录](notes/Word处理学习记录.md) | Office / POI XWPF、模板填充、跨 run 坑 | 2026-09-04 |
| [位图统计学习记录](notes/位图统计学习记录.md) | 统计 / Bitmap 压缩、大 key 拆分、HyperLogLog UV | 2026-09-04 |
| [Flink CDC 学习记录](notes/FlinkCDC学习记录.md) | 数据同步 / MySQL Binlog、全量+增量、断点续传 | 2026-08-26 |
| [HMAC 鉴权学习记录](notes/HMAC鉴权学习记录.md) | 鉴权 / HMAC 无状态签名、防重放、密钥轮换 | 2026-09-04 |
| [MCP 服务学习记录](notes/MCP服务学习记录.md) | AI / MCP 服务端、SSE 传输、工具共享 | 2026-09-02 |
| [Spring AI 学习记录](notes/SpringAI学习记录.md) | AI / 对话、流式、记忆、结构化输出、工具、RAG、MCP | 2026-09-02 |
| [接口幂等学习记录](notes/接口幂等学习记录.md) | 幂等 / 请求级、消费级、状态机级三层去重 | 2026-08-26 |
| [多级缓存学习记录](notes/多级缓存学习记录.md) | 缓存 / Caffeine+Redis Hash+版本号一致性 | 2026-08-26 |

> 以上 22 篇由 `chaos-java` 各 demo 模块提炼而成（工程依托均指向对应模块 GitHub 链接）。内容整理中，低敏起步、逐篇沉淀。

## 目录规划

```text
chaos-notes/
├── pitfalls/        # 踩坑记录
├── notes/           # 技术学习笔记（中间件/框架整合/示例）
├── performance/     # 性能优化（主动调优/压测/基准）
├── docs/            # 设计、索引、模板等
└── ...
```

## License

### 说明

- 内容发布前会清理日志、路径、内网 IP、账号等敏感信息。
- 引用的内容均标注参考来源，不搬运他人源码。