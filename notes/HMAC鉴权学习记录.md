# HMAC 鉴权学习记录

工程依托：[jdk8-hmac-auth-demo](https://github.com/xieqiansong/chaos-java/tree/main/jdk8-platform/jdk8-tech/jdk8-hmac-auth-demo)（`chaos-java` 仓库，纯 JDK8 零依赖，把「集中存储 Token + 每请求读 Redis 校验」演进为「HMAC 无状态签名 + 本地验签」，覆盖签名/防重放/密钥轮换/吞吐对比四场景）。

> 业务实体已泛化为 device / 上报请求，不含任何业务与公司隐私信息。

## 1. 安装

```bash
mvn -q -pl jdk8-tech/jdk8-hmac-auth-demo test     # 签名/篡改/时间窗/重放/轮换 单测
java -cp jdk8-tech/jdk8-hmac-auth-demo/target/classes lan.chaos.hmac.HmacAuthDemo   # 四场景演示 + 吞吐对比
```

## 2. 演进背景

```
早期链路（瓶颈）：
  ① 开机认证：AES-CBC 解密 + 固定 Token 校验 → 通过后签发 Token 存 Redis
  ② 后续上报：请求带 Token → 网关每请求去 Redis 读 Token 校验   ← 每请求 1 次网络往返，吞吐卡在这
后期演进：
  ③ HMAC 改造：认证与上报统一为 HMAC 无状态签名 → 网关本地验签，彻底去掉 ② 的 Redis 读
```

## 3. 四场景

| 场景 | 内容 | 对应问题 |
|---|---|---|
| A 签名/验签 | 签名串规范（method+path+ts+nonce+bodyDigest）+ HMAC-SHA256 + 常量时间比较；篡改/错误密钥被拒 | 设计 |
| B 防重放 | 时间戳时间窗（0 存储）+ 滑动窗口频率限制 + 业务幂等键去重（写侧兜底） | 防重放 |
| C 密钥轮换 | 双密钥过渡：新钥立即签发、旧钥宽限期内仍可验签、宽限期后丢弃 | 轮换 |
| D 吞吐对比 | 本地验签（0 往返）vs Token+Redis 读（模拟 1ms 往返），输出 QPS/avg/p99 | QPS 提升 |

## 4. 防重放为什么不上 Redis（容量账）

引入 HMAC 本为去掉每请求 Redis 读；若 nonce 防重放又去 Redis `SETNX`，等于把「读 Token」换成「读 nonce」，**瓶颈原地搬家**。

| 档位 | 机制 | 网络往返 | 适用 |
|---|---|---|---|
| 默认（高频上报） | 时间戳时间窗 + **业务幂等兜底**（deviceId+batchNo 唯一键，写侧去重） | 0 | 上报主链路 |
| 加强（低频敏感） | nonce + Redis SETNX | 1 | 配置/指令下发（分钟级，一次往返值得） |
| 不做 | 每设备内存 nonce / 全量缓存 | 0 | 跨节点与空间账不支持 |

内存账（1200 万设备）：每条约 216B → **≈2.6 GB**（即使优化成 byte[] ID+时间戳仍 ≈1.06 GB）。跨节点：设备出口 IP 是 NAT，按 IP 吸附必炸单点；按设备 ID 一致性哈希引入粘性会话与扩缩容漏洞。

**结论**：高频上报重放危害是「数据重复写」，用幂等唯一键在写侧去重，成本只发生在真正重复请求上；鉴权链路保持纯无状态。

## 5. 技术要点

- **签名串规范化**：按行拼接 `method\npath\ntimestamp\nnonce\nbodyDigest`，防拼接歧义；bodyDigest 用 SHA-256 保证请求体完整性。
- **常量时间比较**：验签用 `MessageDigest.isEqual`，防时序侧信道。
- **时间戳时间窗**：`|now - ts| <= skew`（默认 300s），无状态、天然水平扩展，仅需 NTP 同步。
- **双密钥轮换**：current 签发、previous 宽限期验签，平滑切换不中断在途设备；宽限期结束广播新密钥后丢弃旧钥。
- **吞吐主线**：消除每请求网络往返（1ms Redis 读 vs 0 往返）→ QPS 数量级提升（1000+ → 10000+）。

## 踩坑

- **防重放别又引入 Redis**：否则瓶颈平移，用写侧幂等唯一键兜底更优。
- **时间窗依赖 NTP**：各节点时钟须同步，否则合法请求被误拒。
- **密钥轮换要留宽限期**：旧钥在途设备仍要能验签，避免切换瞬间大规模失败。
- **常量时间比较**：绝用 `==`/`equals` 比签名，防时序攻击。

## 进阶方向

- 非对称签名（RSA/ECDSA）替代 HMAC 实现「服务端不持私钥验签」。
- 密钥托管 KMS、自动轮换、签名算法协商。
- 与 mTLS / API Gateway 鉴权体系结合。

## 参考来源

- [HMAC (RFC 2104)](https://datatracker.ietf.org/doc/html/rfc2104)
- [OWASP Auth Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
