# Paxos 与 Raft 一致性算法

工程依托：[distributed/consensus-demo](https://github.com/xieqiansong/chaos-java/tree/main/distributed/consensus-demo)（`chaos-java` 仓库，纯 JDK 实现、零外部依赖，从原 `jdk8-base` 拆分而来）。

> 本篇是 [`JDK8 内核与并发内功`](JDK8内核与并发内功.md) 里「Paxos / Raft 一致性算法最小实现」的**深读补充**——那篇只在能力地图一句话带过，这里把可运行的实现思路与验证讲清楚。

## 1. 为什么手写而不是调库

一致性算法是分布式系统的地基，但多数资料停留在「定理/证明」。本 demo 用纯 Java 把**共识怎么达成**跑出来：角色划分清晰、可断言、可观察，比读论文直观。

```bash
mvn -pl distributed/consensus-demo test          # Raft 6 条断言（内存状态机）
# IDE 运行 BasicPaxosDemo 观察一轮 Prepare/Accept/Chosen
```

## 2. Basic Paxos（最小可用）（★★★）

角色：`Proposer` / `Acceptor` / `Learner`，本地传输（进程内消息）。一轮共识走三步：

1. **Prepare**：Proposer 发 `(proposalId, value)` 给 Acceptor，Acceptor 承诺不再接受更小 proposalId 的提案，并返回已接受的最高 proposalId 的值。
2. **Accept**：若多数派 Acceptor 响应，Proposer 用「响应中最大 proposalId 对应的值」或自己的值发 Accept 请求。
3. **Chosen**：多数派 Accept 即达成决议，Learner 学习到最终值。

关键不变量：**一个值一旦被 Chosen，后续任何提案都必须沿用它**——这就是 Paxos 保证一致性的核心（"value 一旦被选中不可变"）。

```java
// 包结构：message / node / role / state / transport，角色划分清晰
// BasicPaxosDemo.main 观察一轮 Prepare/Accept/Chosen 共识过程
```

## 3. Raft 选举（★★★）

Raft 把共识拆成更易理解的「领导者选举 + 日志复制 + 安全性」三部分。本 demo 聚焦前两步：

- **选举**：节点三态 `Follower / Candidate / Leader`；Follower 收不到心跳转 Candidate，发 `RequestVote`，获多数票即成为 Leader。
- **日志复制**：Leader 接收写请求，先写本地日志再复制给 Follower，多数派落盘后提交（commit）并应用（apply）到状态机。

```java
// RaftElectionDemo：观察一轮选举（谁得多数票成为 Leader）
// RaftReplicationDemo：观察日志从 Leader 复制到多数派并提交
```

## 4. Raft 日志复制（★★★）

核心保证：**Leader 的日志是权威的，Follower 不匹配则截断对齐**。用**内存状态机**做确定性模拟，便于断言——每条日志 entry 带 `term` 与 `index`，提交点（commitIndex）推进后状态机 apply。

`RaftConsensusTest`（6 条断言）覆盖：选举出唯一 Leader、日志复制到多数派、提交后状态机一致、Leader 变更后日志不丢等。

## 踩坑 / 理解难点

- **Paxos 活锁**：多个 Proposer 互相抢占 proposalId，导致迟迟不 Chosen——生产用「选一个稳定 Leader 当 Proposer」化解（这也正是 Raft 的思路）。
- **Raft 日志匹配**：Follower 日志冲突必须截断对齐到 Leader，否则状态机分叉。
- **任期 term 单调**：任何 RPC 带 term，发现对方 term 更大立即退位为 Follower，这是 Raft 安全性的基石。
- **多数派才作数**：Prepare/Accept、选举、日志提交都靠「≥ N/2+1」，脑裂时少数派无法推进，保证安全性。

## 进阶方向

- 多副本容错边界：补充压测观察 Paxos/Raft 在丢包、延迟下的表现（原 `JDK8 内核` 笔记已记「多副本容错边界还需补一轮压测」）。
- Raft 成员变更（joint consensus / single-server change）。
- 日志压缩 / 快照（snapshot），避免日志无限增长。
- 与 Zab（ZooKeeper）、Viewstamped Replication 对比。

## 参考来源

- 工程：[distributed/consensus-demo](https://github.com/xieqiansong/chaos-java/tree/main/distributed/consensus-demo)
- [Paxos Made Simple (Lamport)](https://lamport.azurewebsites.net/pubs/paxos-simple.pdf)
- [In Search of an Understandable Consensus Algorithm (Raft)](https://raft.github.io/raft.pdf)
