# Spring Boot Starter 学习记录

工程依托：[jdk8-starter-demo](https://github.com/xieqiansong/chaos-java/tree/main/jdk8-platform/jdk8-starter-demo)（`chaos-java` 仓库，用一个轻量零依赖的 `token-spring-boot-starter` 把自动装配机制讲透，功能只是载体、机制才是重点）。

## 1. 安装

```bash
mvn -pl jdk8-starter-demo -am test             # 4 条可断言单元测试，零外部依赖
mvn -pl jdk8-starter-demo -am spring-boot:run  # 使用方启动，控制台打印生成 token
```

技术栈：JDK 8 + Spring Boot 2.7.18（2.7 起推荐 `AutoConfiguration.imports` 取代 `spring.factories`）。

## 2. 自动装配（核心）

`TokenAutoConfiguration` 经 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 被 Spring Boot 在启动时**主动加载**（非 `@ComponentScan` 扫到）。它 `@EnableConfigurationProperties(TokenProperties.class)` 注册配置绑定类，并 `@Bean` 提供 `TokenService`。

关键 API：`@Configuration` / `@EnableConfigurationProperties` / `@Bean`。

生产坑：自动配置类不要放在扫描包内，否则失去「按需装配」意义。

## 3. 外部化配置

`TokenProperties` 用 `@ConfigurationProperties(prefix = "token.starter")` 把 `application.yml` 的配置项绑定到字段，配合 `spring-boot-configuration-processor` 为 IDE 生成补全元数据。开箱即用默认值（length=32, charset=SIMPLE），用户可覆盖。

## 4. 条件装配（可开关 / 可被覆盖）

- `@ConditionalOnProperty(prefix="token.starter", name="enabled", havingValue="true", matchIfMissing=true)`：开关。
- `@ConditionalOnMissingBean`：仅在用户未自定义时提供默认 `TokenService`。

三个契约：**引依赖即用 / 可开关 / 可被覆盖**。

## 5. 命名约定（细节坑）

- 官方：`spring-boot-starter-xxx`（如 `spring-boot-starter-web`）。
- 第三方：`xxx-spring-boot-starter`（如本 demo 的 `token-spring-boot-starter`、MyBatis-Plus 的 `mybatis-plus-spring-boot-starter`）。
- 配置前缀用业务名（如 `token.starter`），不要占用 `spring.*`。

## 踩坑

- **自动配置类别放扫描包内**：放进去就变成无条件装配，失去 starter 的「按需」意义。
- **配置前缀别占 `spring.*`**：第三方用业务名前缀，避免和官方冲突。
- **可覆盖靠 `@ConditionalOnMissingBean`**：用户自定义 Bean 应让它生效，starter 默认实现让位。
- **开关用 `@ConditionalOnProperty`**：`matchIfMissing=true` 让默认开启、显式 false 关闭。

## 进阶方向

- 不可变绑定：`@ConstructorBinding` + `final` 字段，避免运行时被改。
- 装配顺序：`@AutoConfigureAfter` / `@AutoConfigureBefore` / `@AutoConfigureOrder` 控制多自动配置依赖。
- 元数据：在 `additional-spring-configuration-metadata.json` 补 `description`、默认值提示。
- 独立发布：把 `autoconfigure` + 实现拆成独立 starter 模块，使用方仅引依赖。
- 条件细化：`@ConditionalOnClass` 判断可选依赖是否存在（如仅当 Redis 在 classpath 才装配 Redis 实现）。

## 参考来源

- [Spring Boot 自动配置](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.developing-auto-configuration)
- [Creating Your Own Starter](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.developing-auto-configuration.custom-starter)
