# Seata 学习记录

> 工程依托：[jdk8-seata-demo](https://github.com/xieqiansong/chaos-java/tree/main/jdk8-platform/jdk8-seata-demo)（`chaos-java` 仓库，AT + TCC 两种模式，多进程）。
> 记录分布式事务的核心思路、AT/TCC 差异与落地要点。

## 1. 为什么需要分布式事务

单体应用一个 `@Transactional` 就能回滚；微服务下「扣款 / 下单 / 扣库存」分属不同库，本地事务互不感知，需要**协调者**保证要么全成、要么全回滚。Seata 就是这个协调框架。

## 2. 角色与流程

- **TC（Transaction Coordinator）**：独立服务端，维护全局事务状态、下发提交/回滚。
- **TM（Transaction Manager）**：发起方（`@GlobalTransactional` 标注的方法）。
- **RM（Resource Manager）**：各分支事务，注册分支、执行本地事务、记录回滚日志。

全局事务流程：TM 向 TC 申请 XID → 各 RM 注册分支并执行（记 `undo_log`）→ 全成功 TC 通知提交（异步清 undo_log）/ 任一失败 TC 通知按 undo_log 反向补偿。

## 3. AT 模式（默认推荐）

`BusinessService.purchase` 用 `@GlobalTransactional` 编排：

```java
@GlobalTransactional(timeoutMills = 300000, rollbackFor = Exception.class)
public String purchase(String userId, String productId, double amount, int count) {
    accountService.deduct(userId, amount);   // RM：扣款 + 记 undo_log
    orderService.create(userId, productId, amount);
    storageService.deduct(productId, count); // RM：扣库存 + 记 undo_log
    return orderNo;
}
```

**AT 原理**：
- 执行前快照 `before image`、执行后快照 `after image`，写入 `undo_log`；提交时异步删 undo_log，回滚时按 undo_log 反向补偿。
- 对业务代码**侵入小**（只需加注解 + 建 undo_log 表），适合多数场景。

**坑点**：
- `purchaseFail` 中捕获异常后**必须继续向外抛**，否则 `@GlobalTransactional` 感知不到、不会回滚。
- AT 有全局锁，热点行竞争激烈时性能下降；跨库类型（如非关系型）不支持。

## 4. TCC 模式（手动三阶段）

`jdk8-seata-demo` 的 `tcc` 包用 `TwoPhaseBusinessAction` 实现 `Try/Confirm/Cancel`：

- **Try**：资源预留（如冻结金额、预占库存），不做真正扣减。
- **Confirm**：Try 全成功后真正提交（需幂等）。
- **Cancel**：任一 Try 失败释放预留（需幂等）。

`AccountTccAction` / `OrderTccAction` / `StorageTccAction` 分别对应三个服务的 TCC 动作，`BusinessTccService` 编排。

**AT vs TCC**：
| 维度 | AT | TCC |
|---|---|---|
| 侵入 | 低（注解 + undo_log 表） | 高（手写 Try/Confirm/Cancel） |
| 性能 | 有全局锁，适中 | 无锁，预留资源，高并发友好 |
| 一致性 | 最终一致（补偿） | 最终一致（补偿） |
| 适用 | 绝大多数内部 DB 事务 | 跨异构资源、高并发、长事务 |

## 5. 学习踩坑点

- **undo_log 表必须存在**：AT 模式依赖各分支库的 `undo_log`，漏建表事务无法回滚。
- **Confirm/Cancel 必须幂等**：网络重试会重复调用，重复执行不能产生副作用。
- **超时自动回滚**：`@GlobalTransactional(timeoutMills)` 默认 60s，长事务要调大，否则被 TC 强制回滚。
- **XID 透传**：跨服务调用要把 XID 传下去（Spring Cloud 用 `SeataFilter`/`RestTemplate` 拦截器），否则下游 RM 不在同一全局事务。

## 6. 参考来源

- 工程：[jdk8-seata-demo](https://github.com/xieqiansong/chaos-java/tree/main/jdk8-platform/jdk8-seata-demo)（AT: `at/`、TCC: `tcc/`）
- 与 RocketMQ 事务消息、Kafka 事务的关系：三者都在解决「跨资源一致性」，但层次不同（见 Kafka/ RocketMQ 学习记录）
