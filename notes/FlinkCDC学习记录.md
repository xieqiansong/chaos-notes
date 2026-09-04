# Flink CDC 学习记录

工程依托：[jdk8-flink-cdc-sync-demo](https://github.com/xieqiansong/chaos-java/tree/main/jdk8-platform/jdk8-tech/jdk8-flink-cdc-sync-demo)（`chaos-java` 仓库，基于 Flink CDC `flink-connector-mysql-cdc` 的 MySQL 增量同步示例，将源表实时同步到同实例目标表（表名加 `_new`））。

## 1. 安装

```bash
# 1) 准备 MySQL（开启 binlog，格式 ROW），建表并写入样例数据
mysql -h127.0.0.1 -P30100 -uroot -p<密码> demo < src/main/resources/schema.sql

# 2) 修改 application.properties 连接信息（地址/端口/账号/密码/时区）

# 3) 运行作业入口，或打包提交到 Flink
mvn -o package
flink run target/jdk8-flink-cdc-sync-demo-1.0.0.jar
```

> 运行环境 JDK 8（Flink 1.17 不兼容 JDK21，故放 `jdk8-tech` 而非 `jdk21-tech`）。

## 2. 功能

- **MySQL Binlog 实时捕获**：`MySqlSource` 读源库 binlog，捕获 INSERT/UPDATE/DELETE 变更。
- **全量 + 增量**：`StartupOptions.initial()`，启动先扫全量再无缝衔接增量 binlog，不丢历史。
- **断点续传**：开启 Flink Checkpoint（默认 3 秒一次，落本地文件系统），异常重启从位点续传，不重复不遗漏。
- **通用字段映射**：`IMapping` 接口描述「源→目标」列映射，支持字段改名（`nickname → display_name`）、字段派生（`state=2` → `finished=1`）。新增同步表只需实现一个 `IMapping`。
- **目标表写入**：Flink JDBC Sink，基于主键 `ON DUPLICATE KEY UPDATE` 幂等 upsert；删除事件对应 `DELETE`。

## 3. 内置示例

| 源表 | 目标表 | 说明 |
|---|---|---|
| `user` | `user_new` | 字段改名：`nickname → display_name`、`create_time → created_at` |
| `order` | `order_new` | 字段派生：`state` 推导 `finished`（`state=2` 视为已完成） |

## 踩坑

- **时区一致性**：`source.server.timezone` 必须与 MySQL 实际时区一致，否则 CDC 校验直接失败（须用 `UTC` / `Asia/Shanghai` 这类时区名）。
- **server-id 唯一**：CDC 客户端 server-id 在集群内须唯一，否则抢占 binlog 位点。
- **Checkpoint 是断点续传关键**：不开启就失去位点恢复能力，作业重启会重复消费。
- **元数据驱动扩展**：新增同步表只需加 `IMapping` 实现并注册到 `MysqlSyncJob.MAPPINGS`，无需改 source/sink 代码。

## 进阶方向

- 跨实例/跨库同步、多表正则匹配、schema 变更（DDL）处理。
- Exactly-Once Sink（如写入支持事务的外部存储）、监控与告警（LAG、Checkpoint 失败）。
- 替换为 Kafka + Flink SQL CDC，解耦源端与消费端。

## 参考来源

- [Flink CDC 官方文档](https://ververica.github.io/flink-cdc-connectors/)
- [MySQL Binlog](https://dev.mysql.com/doc/refman/8.0/en/binary-log.html)
