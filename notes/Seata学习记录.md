# Seata 学习记录

工程依托：[jdk8-seata-demo](https://github.com/xieqiansong/chaos-java/tree/main/jdk8-platform/jdk8-seata-demo)（`chaos-java` 仓库，AT + TCC 两种模式，多进程）。

微服务下「扣款 / 下单 / 扣库存」分属不同库，本地事务互不感知，需要协调者保证要么全成、要么全回滚。Seata 就是这个协调框架，核心是 TC（协调者）+ TM（发起方）+ RM（各分支）。

## 1. 安装

TC 服务端，`seata-server:1.6.1`，standalone 用 `file` 模式（注册/配置/存储都是 file，演示够用）。控制台 `7091` 映射 `30106`，事务协调端口 `8091` 映射 `30107`。

`docker-compose.yml`：

```yaml
services:
  # ==================== Seata Server（TC） ====================
  seata-server:
    image: seataio/seata-server:1.6.1
    container_name: seata-server
    restart: unless-stopped
    ports:
      - "30106:7091"   # Web 控制台（查看全局事务日志）  ← 外层统一端口
      - "30107:8091"   # 事务协调端口（TM / RM 通信）   ← 外层统一端口
    environment:
      - SEATA_REGISTRY_TYPE=file
      - SEATA_CONFIG_TYPE=file
      - STORE_MODE=file
      - SEATA_PORT=8091          # 容器内 TC 监听端口（默认即 8091，显式写明更清晰）
      # 控制台端口已由镜像内置为 7091，无需再通过 SEATA_PORT 覆盖
```

`docker-compose up -d` 起来，控制台 `http://<ubuntu-ip>:30106`。业务应用连 TC 用 `localhost:30107`（事务协调端口）。下面命令行在 ubuntu 宿主机执行，控制台 API 走 `30106`。

## 2. AT 模式（默认推荐）

AT 对业务侵入小：只需加 `@GlobalTransactional` 注解 + 各分支库建 `undo_log` 表。Seata 自动记前后镜像，提交时清 undo_log，回滚时按 undo_log 反向补偿。

```java
// TM 发起方：一个注解编排三个分支（扣款 / 下单 / 扣库存）
@GlobalTransactional(timeoutMills = 300000, rollbackFor = Exception.class)
public String purchase(String userId, String productId, double amount, int count) {
    accountService.deduct(userId, amount);   // RM：扣款 + 记 undo_log
    orderService.create(userId, productId, amount);
    storageService.deduct(productId, count); // RM：扣库存 + 记 undo_log
    return orderNo;
}
```

```bash
# 对应 Seata 指令（控制台 API，查看全局事务状态）
# 查看全局事务列表（state: Begin/Committing/Rollbacking）
curl -X GET 'http://localhost:30106/v1/transaction/global/list?pageNum=1&pageSize=10'

# 查看某全局事务详情（替换为实际 xid）
curl -X GET 'http://localhost:30106/v1/transaction/global?xid=xxx'
```

> 坑：异常被 catch 后**必须继续向外抛**，否则 `@GlobalTransactional` 感知不到、不会回滚；`undo_log` 表漏建则无法回滚。

## 3. TCC 模式（手动三阶段）

TCC 无全局锁、高并发友好，但要手写 `Try/Confirm/Cancel`。`tcc` 包里 `AccountTccAction` / `OrderTccAction` / `StorageTccAction` 对应三个服务的动作，`BusinessTccService` 编排。

```java
// 每个动作接口用 @TwoPhaseBusinessAction 声明两阶段方法
@LocalTCC
public interface AccountTccAction {
    @TwoPhaseBusinessAction(name = "accountTccAction", commitMethod = "commit", rollbackMethod = "rollback")
    boolean prepare(BusinessActionContext ctx, String userId, double amount); // Try：冻结金额
    boolean commit(BusinessActionContext ctx);   // Confirm：真正扣减（需幂等）
    boolean rollback(BusinessActionContext ctx); // Cancel：释放冻结（需幂等）
}

// 编排：调用各 prepare，全部成功 Seata 自动调 confirm，任一失败调 cancel
public String purchase(String userId, String productId, double amount, int count) {
    accountTccAction.prepare(ctx, userId, amount);
    orderTccAction.prepare(ctx, userId, productId, amount);
    storageTccAction.prepare(ctx, productId, count);
    return orderNo;
}
```

```bash
# 对应 Seata 指令（TCC 同样走全局事务，状态查看同上）
curl -X GET 'http://localhost:30106/v1/transaction/global/list?pageNum=1&pageSize=10'
```

> Confirm/Cancel 必须幂等（网络重试会重复调）；TCC 适合跨异构资源、高并发、长事务。

## 4. AT vs TCC

| 维度 | AT | TCC |
|---|---|---|
| 侵入 | 低（注解 + undo_log 表） | 高（手写 Try/Confirm/Cancel） |
| 性能 | 有全局锁，适中 | 无锁，预留资源，高并发友好 |
| 一致性 | 最终一致（补偿） | 最终一致（补偿） |
| 适用 | 绝大多数内部 DB 事务 | 跨异构资源、高并发、长事务 |

## 5. 几个注意点

- **undo_log 表必须存在**：AT 依赖各分支库的 `undo_log`，漏建表事务无法回滚。
- **Confirm/Cancel 必须幂等**：网络重试会重复调用，重复执行不能产生副作用。
- **超时自动回滚**：`@GlobalTransactional(timeoutMills)` 默认 60s，长事务要调大，否则被 TC 强制回滚。
- **XID 透传**：跨服务调用要把 XID 传下去（Spring Cloud 用 `SeataFilter`/`RestTemplate` 拦截器），否则下游 RM 不在同一全局事务。

## 参考来源

- 工程：[jdk8-seata-demo](https://github.com/xieqiansong/chaos-java/tree/main/jdk8-platform/jdk8-seata-demo)（AT: `at/`、TCC: `tcc/`）
- 与 RocketMQ 事务消息、Kafka 事务的关系：三者都在解决「跨资源一致性」，但层次不同（见 Kafka / RocketMQ 学习记录）
