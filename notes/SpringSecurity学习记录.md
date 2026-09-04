# Spring Security 学习记录

工程依托：[jdk8-security-demo](https://github.com/xieqiansong/chaos-java/tree/main/jdk8-platform/jdk8-security-demo)（`chaos-java` 仓库，Spring Boot 2.7 + spring-boot-starter-security + jjwt 0.11.5 + oauth2-resource-server，用一个样例用户 `alice` 串起三类方案）。

## 1. 安装

```bash
# 单元测试（MockMvc，零外部依赖）
mvn -pl jdk8-security-demo test

# 启动应用用 curl 把玩四个端点
mvn -pl jdk8-security-demo spring-boot:run
curl localhost:8080/api/public
curl -u alice:secret localhost:8080/api/secure
curl -X POST "localhost:8080/api/token?user=alice"            # 返回 "Bearer <jwt>"
curl -H "Authorization: Bearer <jwt>" localhost:8080/api/jwt-secure
```

OAuth2 资源服务器需外部 IdP，提供 `docker-compose.yml`（Keycloak）参考，默认不启用。

## 2. Spring Security 基础防护（★★★）

痛点：每个接口手写鉴权重复又易漏。Spring Security 用「过滤器链 + 规则」统一保护，默认拦截一切、白名单放行、未认证返回 401。

- 关键 API：`SecurityFilterChain` Bean（2.7 推荐，替代旧 `WebSecurityConfigurerAdapter`）、`authorizeHttpRequests`、`httpBasic`、`SessionCreationPolicy.STATELESS`。
- 生产坑：无状态必须 `STATELESS`（否则仍建 Session）；密码应 `{bcrypt}` 加密（演示才 `{noop}`）；有 Session 的 Web 别乱关 CSRF。

## 3. JWT 无状态 Token（★★★）

痛点：传统 Session 需服务端存登录态、集群要共享（Redis），跨域/移动端不友好。JWT 把声明自包含进签名 Token，服务端**无状态**校验。

- 关键 API（jjwt）：`Jwts.builder().setSubject().setExpiration().signWith(key,HS256).compact()` 与 `parserBuilder().setSigningKey().build().parseClaimsJws(token)`。
- 本 demo：`JwtService` 签发/解析；`JwtAuthFilter` 从 `Authorization: Bearer <token>` 取 Token 写 `SecurityContext`；`/api/jwt-secure` 经它保护。
- 生产坑：密钥够强且放 KMS；JWT 无法主动吊销（短过期+刷新令牌/黑名单）；payload 非加密别放敏感；固定 HS256 防算法混淆。

## 4. OAuth2 资源服务器（外部 IdP）（★★★）

痛点：自己管登录/密钥轮换/注销成本高。生产更常见把认证外包给 IdP（Keycloak/Auth0/微信），本服务只做**资源服务器**：用 IdP 公钥校验其签发的 JWT。

- 关键 API：`spring-boot-starter-oauth2-resource-server` + `oauth2ResourceServer(o -> o.jwt())`，配 `jwk-set-uri` 指向 IdP 的 JWK 集合。
- **默认不启用**：`OAuth2ResourceServerConfig` 用 `@ConditionalOnProperty(jwk-set-uri)` + `@Order(0)`，仅当配置了 IdP 才激活并优先于本地链。
- 授权码模式：客户端 → 重定向 IdP 登录授权 → 拿 code → 换 access_token(JWT) → 带 Bearer 访问本服务 → 用 IdP 公钥验签。
- 生产坑：必须 HTTPS；公共客户端开 PKCE；权限从 JWT claims 取；IdP 不可达要有 JWK 缓存降级。

## 踩坑

- **无状态必须 `STATELESS`**：否则仍建 Session，分布式下状态不一致。
- **密码存储**：`{bcrypt}`/`Argon2`，绝明文（见 `jdk8-crypto-demo`）。
- **JWT 难吊销**：短 access_token + refresh_token 换发；或维护黑名单。
- **payload 非加密**：别放敏感信息，需要加密用 JWE。
- **OAuth2 公共客户端必开 PKCE**：防授权码拦截；务必 HTTPS。
- **方法级授权**：`@PreAuthorize("hasRole('ADMIN')")` 做更细粒度控制。

## 进阶方向

- Session vs Token 对比（Session 简单但需服务端状态；JWT 无状态但难吊销），按「是否多端/微服务」选型。
- 刷新令牌、方法级授权、`@PreAuthorize`、OAuth2 授权码 + PKCE、密码存储算法。

## 参考来源

- [Spring Security 官方文档](https://docs.spring.io/spring-security/reference/index.html)
- [jjwt](https://github.com/jwtk/jjwt)
- [OAuth 2.0 / PKCE](https://datatracker.ietf.org/doc/html/rfc6749)
