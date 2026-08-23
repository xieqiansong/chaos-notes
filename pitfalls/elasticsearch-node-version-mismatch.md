# Elasticsearch 集群节点版本不一致导致数据不同步

> 分类：搜索引擎 / Elasticsearch / 运维
> 踩坑耗时：较长（主机下线 + 新节点加入，伴随数据同步问题）
> 关键点：**集群节点小版本号不一致，导致分片/数据无法正常同步**

> 整理时间：2026-08-23
> 发生时间：约 2023-01
> 说明：本条目依据个人回忆整理，原始操作记录未在归档检索到，细节可能与实际有偏差，使用前请结合当时的 ES 版本核对。

## 场景

对 ES 集群进行调整：**下线一台主机，接入一台新节点**（用于替代/扩容）。新节点与原集群版本存在**小版本差异**（如主版本相同、次版本不同）。

## 表现

- 新节点加入集群后，一直处于**未恢复/未就绪**状态（`_cat/nodes` 可见但状态异常）。
- `_cat/shards` 出现大量 `UNASSIGNED`，索引分片**无法同步/分配**到新节点。
- 数据无法在新旧节点间复制，集群整体 `status: yellow` 或 `red`。

## 排查

```bash
# 各节点版本是否一致
curl -s "$ES_HOST:9200/_nodes?filter_path=**.version">
# 查看分片分配情况，定位 UNASSIGNED
curl -s "$ES_HOST:9200/_cat/shards?v" | grep UNASSIGNED
# 查看具体未分配原因（大版本 REST 支持 explain，小版本结果字段不同）
curl -s "$ES_HOST:9200/_cluster/allocation/explain?pretty"
```

## 根因

- Elasticsearch 节点间的**版本兼容性有约束**：跨越大版本差异的节点无法加入集群；即便是**同主版本的不同小版本**，在节点加入、分片复制时也可能出现数据不同步/无法分配（例如集群 `minimum_master_nodes` 判定、序列化/协议、mapping 版本差异、新老节点对分片数据的兼容差异）。
- 因此**在"主机下线 + 添加新节点"这类扩容/替换操作中，节点间版本必须对齐**，小版本不一致是数据无法同步的直接诱因。

## 修改

1. **统一版本**：将新旧节点调整到**完全一致**的 ES 版本，再执行加入/替换操作。
2. 若需下线旧机、接入新机，先让新节点以**正确版本**加入集群，待分片完整分配后再下线旧节点：
   ```bash
   # 将数据迁移到新节点（按节点名），再下线旧节点
   curl -XPUT "$ES_HOST:9200/_cluster/settings?pretty" -d '{
     "transient": { "cluster.routing.allocation.exclude._name": "OLD_NODE_NAME" }
   }'
   ```
3. 补充确认：`cluster.initial_master_nodes` / `discovery.seed_hosts` 中把旧节点改为新节点，避免旧配置干扰。
4. 对长时间 `UNASSIGNED` 且需手工恢复的分片，可用 `_cluster/reroute` 的 `allocate_stale_primary` / `allocate_empty_primary`（注意 `accept_data_loss` 需谨慎）。

## 复查

- `curl -s "$ES_HOST:9200/_cat/nodes?v"` 确认各节点版本一致、节点状态正常。
- `curl -s "$ES_HOST:9200/_cat/shards?v"` 无 `UNASSIGNED`，分片均衡分布。
- `curl -s "$ES_HOST:9200/_cluster/health"` 状态为 `green`，副本已就位。

## 预防

- **扩容/替换前先核对所有节点版本完全一致**，这是加入与数据同步的前提。
- 维护 `discovery` 相关配置清单，替换节点时同步更新 `initial_master_nodes` / `seed_hosts`。
- 大版本升级务必走标准升级路径（滚动升级），避免不同版本节点长期共存。