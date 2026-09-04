# ZooKeeper 学习记录

工程依托：[jdk8-zookeeper-demo](https://github.com/xieqiansong/chaos-java/tree/main/jdk8-platform/jdk8-zookeeper-demo)（`chaos-java` 仓库，Spring Boot 2.7 + Apache Curator 5.5.0，覆盖分布式锁/Leader选举/配置中心三个协调原语）。

## 1. 安装

需先起一个单机 ZooKeeper（3.8，端口 2181）：

```bash
cd jdk8-zookeeper-demo
docker compose up -d
mvn -pl jdk8-zookeeper-demo -U test   # ZK 就绪后三场景执行并断言；无 ZK 时自动跳过
# 多个终端各跑一次 ZookeeperDemoApplication.main 可观察跨进程锁争抢/选主
```

客户端用 **Curator** 而非原生 ZK：原生 API 偏底层（Watcher 一次性、无重连），Curator 封装了重连、重试与分布式原语（锁/选主/缓存），生产首选。

## 2. 分布式锁（★★★）

多进程/多实例争抢同一把锁，保证临界区互斥。

- 机制：ZK 用「临时顺序节点」实现公平互斥锁，抢不到就监听前驱节点，前驱释放再抢（无羊群效应）。
- 关键 API：`InterProcessMutex#acquire` / `#release`。
- 生产坑：必须 `release`（否则靠临时节点在 session 断开时自动释放）；锁粒度要小；高并发短临界区更推荐 Redis 锁（性能更好），ZK 锁胜在强一致与自动释放。

## 3. Leader 选举（★★★）

多实例选一个 Leader 承担「只能一个干」的主任务（全局定时调度、主数据同步、主备切换），其余 standby，失败自动转移。

- 机制：顺序临时节点公平裁决谁当选；`autoRequeue()` 让失去领导权后自动重新排队。
- 关键 API：`LeaderSelector` + `LeaderSelectorListenerAdapter#takeLeadership`。
- 生产坑：`takeLeadership` 不返回就一直持有领导权，释放靠 `close()` 中断；领导权转移有「脑裂窗口」，业务要幂等或加租约。

## 4. 配置中心 + Watcher（★★★）

把配置放 ZK 节点，变更时「推」给所有监听方，动态生效，无需重启。

- 机制：原生 Watcher 一次性（触发后需重注册），Curator 的 `NodeCache` 帮我们持续监听。
- 关键 API：`client.setData()/getData()` + `NodeCache`。
- 生产坑：Watcher 一次性，漏注册会丢事件——用 NodeCache 省心；要有默认值与本地兜底；单节点有 **1MB 上限**，超大配置放对象存储 + ZK 存指针。

## 踩坑

- **ZK 锁 vs Redis 锁**：ZK 强一致/自动释放（胜在可靠），Redis 高性能（胜在吞吐），按一致性与性能要求取舍。
- **临时节点自动释放是双刃剑**：session 超时也会释放锁，网络抖动可能误释放，短临界区更稳。
- **Watcher 一次性**：漏重注册就丢事件，统一用 Curator Cache（NodeCache/PathChildrenCache/TreeCache）。
- **脑裂窗口**：Leader 切换间隙可能双主，业务必须幂等或加租约。
- **1MB 节点上限**：大配置别直接塞 ZK。

## 进阶方向

- 多进程观察（开多个终端跑 `main` 真实看锁等待/选主/配置广播）。
- Curator Cache 进阶（PathChildrenCache/TreeCache 监听子节点与整棵子树）。
- 生产部署奇数节点集群；客户端配 `connectionTimeout`/`sessionTimeout` 与合适重试策略；ACL/SASL 鉴权防任意客户端改协调数据。

## 参考来源

- [Apache Curator](https://curator.apache.org/)
- [ZooKeeper 官方文档](https://zookeeper.apache.org/doc/current/)
