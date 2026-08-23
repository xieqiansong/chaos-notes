# Linux 高负载：nf_conntrack 连接跟踪表溢出

> 分类：Linux / 网络 / 运维
> 踩坑耗时：长时间（服务器负载高，最终通过调大连接跟踪上限解决）
> 关键参数：`net.netfilter.nf_conntrack_max`
> 整理时间：2026-08-23
> 发生时间：2024-04

## 场景

一台承载**大量终端心跳/状态上报**的服务器（高并发、长连接场景），持续出现**高负载**。默认的 conntrack 连接跟踪配置不足以支撑海量连接。

## 表现

- `top` 显示系统负载持续偏高，CPU 利用率异常。
- 大量短连接/长连接同时存在时，新连接建立常超时或触发重传。
- 连接跟踪表达到上限，新连接被丢弃，表现为服务响应缓慢。

## 排查

查看当前连接跟踪状态与上限（确认是否被 conntrack 拖垮）：

```bash
# 当前连接跟踪表使用量 / 上限
cat /proc/sys/net/netfilter/nf_conntrack_max
# 当前正在跟踪的连接数
cat /proc/net/netfilter/nf_conntrack
# 也可以看全量统计，确认表是否已满
sysctl net.netfilter.nf_conntrack_max
sysctl net.netfilter.nf_conntrack_count
```

结合流量视角确认是否为高并发连接跟踪请求（例如按源 IP 统计高频访问方）。

## 根因

- 默认 conntrack 上限过低，在高并发的终端连接上报场景下，跟踪表被**占满**。
- 表满后无法建立新连接跟踪，转发/服务异常，最终表现为**服务器高负载**。
- 因此核心解法是把 `nf_conntrack_max` 从默认值提升到足够大的值（本案例提到最终调整到 **20000000** 量级才解决负载问题）。

## 修改

1. **临时生效**：
   ```bash
   sudo sysctl -w net.netfilter.nf_conntrack_max="20000000"
   ```
2. **持久化**到 `/etc/sysctl.conf`，同时配套关键内核调优：
   ```ini
   # 打开文件句柄数量
   fs.file-max = 655360

   # 最大 IP 连接跟踪数（本次核心）
   net.netfilter.nf_conntrack_max = 20000000

   # keepalive 时，TCP 发送 keepalive 频率：缺省 2 小时，改为 2 分钟
   net.netfilter.nf_conntrack_tcp_timeout_established = 120

   # TCP 收发缓冲区
   net.ipv4.tcp_rmem = 8760 256960 4088000
   net.ipv4.tcp_wmem = 8760 256960 4088000

   # 允许系统打开的端口范围，扩大端口数
   net.ipv4.ip_local_port_range = 1024 65535

   # LISTEN 队列上限
   net.core.somaxconn = 65535
   net.core.netdev_max_backlog = 262144

   # TIME_WAIT 处理
   net.ipv4.tcp_fin_timeout = 15
   net.ipv4.tcp_tw_reuse = 1
   net.ipv4.tcp_max_tw_buckets = 262144

   # 三类连接队列上限
   net.ipv4.tcp_max_orphans = 262144
   net.ipv4.tcp_max_syn_backlog = 262144
   net.ipv4.tcp_synack_retries = 1
   net.ipv4.tcp_syn_retries = 2
   ```
3. **使配置生效**：
   ```bash
   sudo sysctl -p
   ```
4. 若同时使用 iptables/firewalld，关注其 `conntrack` 模块影响；防火墙规则本身建议按来源 IP 白名单放行而非全放（避免无效连接占用跟踪表）。

## 复查

- `cat /proc/net/netfilter/nf_conntrack_count` 明显低于新上限，不再触顶。
- `top` 负载明显回落，新连接建立顺畅、不再超时/重传。
- 观察一段时间，确认高负载问题不再复现。

## 预防

- 高并发长连接（终端/心跳/上报）类服务，上线前**预估并发连接量并提前调大 conntrack 上限**。
- 同时间关注 `tcp_timestamps`/`tcp_tw_recycle` 的兼容性：在 NAT 环境下启用可能导致异常，需谨慎评估。
- 用防火墙白名单限制来源，从源头减少无效连接对跟踪表的占用。