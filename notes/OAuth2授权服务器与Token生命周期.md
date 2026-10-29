# OAuth2 授权服务器与 Token 生命周期

工程依托：[microservice/oauth2-demo](https://github.com/xieqiansong/chaos-java/tree/main/microservice/oauth2-demo)（`chaos-java` 仓库，Spring Boot 2.7.18 + JDK 8 + jjwt 0.11.5，**不引 `spring-authorization-server`**，手写最小可用 IdP）。

> 与 [`Spring Security 学习记录`](SpringSecurity学习记录.md) 的关系：那篇讲「**资源服务器**消费外部 IdP 签发的 JWT」；本篇讲「**授权服务器（IdP）本身**——发 Token + 管 Token 生命周期」。二者通过 `JWK Set URI` 衔接，是同一授权的两端。

## 1. 为什么手写而不是引库

本仓库定位「讲清机制」，所以不依赖 `spring-authorization-server`，把授权码、Token、刷新、撤销逐一显式实现。三个独立进程（形态二多模块）：

| 模块 | 端口 | 角色 |
|---|---|---|
| `oauth2-auth-server` | 18081 | 授权服务器（IdP）：发 Token + 管生命周期 |
| `oauth2-resource-server` | 18082 | 受保护资源，启动时从 IdP 拉 JWK 公钥验签 |
| `oauth2-client` | 18083 | 第三方客户端，自动跑通全链路 |

```bash
# 依次启动三个应用（终端 1/2/3）
mvn -o -pl microservice/oauth2-demo/oauth2-auth-server -am spring-boot:run
mvn -o -pl microservice/oauth2-demo/oauth2-resource-server -am spring-boot:run
mvn -o -pl microservice/oauth2-demo/oauth2-client -am spring-boot:run   # 启动即自动跑流程
```

内置样例身份（演示用，非真实）：用户 `alice/secret`、保密客户端 `demo_client_01/demo_secret_01`、公共客户端 `demo_public_client_01`（无 secret，强制 PKCE）。

## 2. 授权码 + PKCE（★★★）

标准授权码流程 + 公共客户端 PKCE，是 OAuth2 最核心的一环。

- 授权端点 `/oauth2/authorize`：校验 `client_id` / `redirect_uri` / `scope`（最小化），生成 `code`（绑定 `userId/scope/codeChallenge`，**单次消费**，TTL 10min）。
- 换 Token `/oauth2/token`：校验 code（单次消费）+ 客户端身份 + `code_verifier`（PKCE）。公共客户端必须带 `code_challenge(S256)`，换 Token 时验 `code_verifier`。

```bash
# 1) 取授权码（Basic 登录 alice）
curl -u alice:secret 'http://localhost:18081/oauth2/authorize?client_id=demo_client_01&redirect_uri=http://127.0.0.1:18083/cb&scope=user.basic&response_mode=json&auto_consent=true'
# 2) 换 Token（Basic 客户端身份 + code）
curl -u demo_client_01:demo_secret_01 -d 'grant_type=authorization_code&code=<上一步code>&redirect_uri=http://127.0.0.1:18083/cb' http://localhost:18081/oauth2/token
```

典型时序：`client → authorize(登录) → 拿到 {code} → token(code) → 拿到 {access, refresh} → 带 Bearer access 访问 resource → 资源服务器用 JWK 验签 → 返回受保护数据`。

## 3. Token 模型：JWT access + 不透明 refresh（★★★）

- **access_token = JWT**（RS256，含 `jti/exp/scope/client_id/sub`）：自包含，资源服务器可无状态验签，无需查库。
- **refresh_token = 不透明随机串**（落库 `TokenStore`）：便于撤销、单点/整族失效。

拆开看清楚职责：access 主打「无状态、可离线验签」，refresh 主打「可管控、可吊销」。两者分开放，才兼顾性能与可控。

## 4. 滚动刷新（★★★）

每次刷新签发【新 access + 新 refresh】，旧 refresh **立即 `revoked`**；用 `familyId` 串联令牌族——用户改密时可整族失效。

```
refresh(旧 refresh) → 校验旧 refresh 存在且未撤销 → 旧 refresh 立即失效
                    → 签发新 access + 新 refresh（沿用同一 familyId）
再用旧 refresh 重试 → 400 invalid_grant（已被滚动刷新作废）
```

要点：刷新不是「延长旧令牌」，而是「发一套新的、废掉旧的」，避免旧 refresh 被截获后长期可用。

## 5. 撤销 + 自省（黑名单）（★★★）

JWT 自包含、无法即时失效，这是它最大的矛盾点。解法是 **`jti` 黑名单**（`TokenStore`）：

- `/oauth2/revoke`：`revokeAccess(jti)` / `revokeRefresh` / `revokeByUser` / `revokeByClient` 批量失效。
- 资源服务器验签通过后，再查 `jti` 是否命中黑名单（本地缓存）→ 命中即 401。
- `/oauth2/introspect`（RFC 7662）：返回 `active/client_id/username/scope/exp`，供需要「在线校验」的场景。

> 生产可选 Redis 存黑名单：TTL 自然过期 + 黑名单集合，多实例共享。

## 6. JWK 公钥分发（★★★）

授权服务器暴露 `/.well-known/jwks.json`（`JwtKeyConfig` 暴露 `RSAPublicKey`），资源服务器启动时拉取并缓存公钥用于验签——这就是与 [`Spring Security 学习记录`](SpringSecurity学习记录.md) 资源服务器侧 `jwk-set-uri` 衔接的点。

## 7. Token 生命周期状态机

```
issue → active ──refresh──▶ active'（新 access/refresh；旧 refresh 立即失效）
          ├─ exp 到期 ─────▶ expired（验签/校验失败）
          └─ revoke(jti) ──▶ revoked（黑名单，即时失效，无需等 exp）
```

核心矛盾与解法：JWT 无法即时失效 → `jti` 黑名单兜底；access 短 TTL（30min）+ 黑名单，refresh 落库可单点/整族失效。

## 踩坑

- **授权码必须单次消费**：复用同一个 code 换 Token 第二次要失败，否则授权码泄露即被冒用。
- **JWT 难吊销**：别指望 access 自己过期前的即时失效，必须配黑名单/短 TTL。
- **`redirect_uri` 强校验**：不校验就会被钓鱼站点截授权码；`state` 防 CSRF；scope 最小化。
- **客户端密钥 bcrypt 比对**：`/token`、`/revoke`、`/introspect` 手动解析 Basic 头并 bcrypt 比对，避开 Spring Security 全局 Basic 过滤器与用户登录链混淆（双链设计）。
- **全程 HTTPS**：生产环境明文传输授权码/Token 等于裸奔。

## 进阶方向

- 用 Redis 替代内存 `TokenStore`（分布式黑名单 / 多实例共享 refresh）。
- 完整 OIDC（id_token、userinfo 标准化 claims）。
- 设备码模式（Device Code）、动态 scope 审批 UI。
- 多租户 / 密钥 `kid` 轮换（KMS 托管私钥）。

## 参考来源

- 工程：[microservice/oauth2-demo](https://github.com/xieqiansong/chaos-java/tree/main/microservice/oauth2-demo)（内含 `DESIGN.md` 开发文档）
- [OAuth 2.0 授权框架 RFC 6749](https://datatracker.ietf.org/doc/html/rfc6749)
- [PKCE RFC 7636](https://datatracker.ietf.org/doc/html/rfc7636)
- [Token 自省 RFC 7662](https://datatracker.ietf.org/doc/html/rfc7662)
