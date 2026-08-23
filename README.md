# chaos-notes

> 技术笔记仓库：**踩坑记录** + **技术学习笔记**。Chaos 本意保持混乱，把精力集中在当前要做的事上。

我的开源技术笔记集中地，聚焦两条主线：

## 内容主线

### 1. 踩坑记录 (Pitfalls)

记录真实踩过的 BUG、配置坑与排查过程。采用「场景 → 原因 → 解决」模板，方便回查与复用。

| 踩坑 | 主题 | 时间 |
|---|---|---|
| [MySQL 5.5 共享表空间改独立表空间](pitfalls/mysql-innodb-file-per-table.md) | InnnoDB 表空间 / 数据重建 | 约 2022-06 |
| [Linux 高负载：nf_conntrack 连接跟踪表溢出](pitfalls/linux-nf-conntrack-high-load.md) | 网络 / conntrack 调优 | 约 2025-04 |
| [ES 集群节点版本不一致导致数据不同步](pitfalls/elasticsearch-node-version-mismatch.md) | 搜索引擎 / 节点版本 | 约 2023-01 |

### 2. 技术学习笔记 (Notes)

中间件部署、示例代码、现有框架整合等学习沉淀。覆盖 Kafka / Redis / RocketMQ / Nacos / Seata 等。

> 内容整理中，低敏起步、逐篇沉淀。

## 目录规划

```text
chaos-notes/
├── pitfalls/        # 踩坑记录
├── notes/           # 技术学习笔记（中间件/框架整合/示例）
├── docs/            # 设计、索引、模板等
└── ...
```

## License

### 说明

- 内容发布前会清理日志、路径、内网 IP、账号等敏感信息。
- 引用的内容均标注参考来源，不搬运他人源码。