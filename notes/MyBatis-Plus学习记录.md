# MyBatis-Plus 学习记录

工程依托：[jdk11-mybatis-plus-demo](https://github.com/xieqiansong/chaos-java/tree/main/jdk11-platform/jdk11-mybatis-plus-demo)（`chaos-java` 仓库，Spring Boot 2.7 + MyBatis-Plus 3.5.16，H2 内存库零外部依赖，覆盖 6 个高频实战能力）。

> 本 demo 原位于 `jdk8-platform/jdk8-mybatis-plus-demo`，因 MyBatis-Plus 3.5.16 的拦截器模块 `mybatis-plus-jsqlparser`（自 3.5.9+ 从 extension 拆分）需 JDK 11+ 字节码，**无法在 JDK 8 编译**，故整体迁移到 `jdk11-platform`。

## 1. 安装

```bash
# 在仓库根目录
mvn -pl jdk11-platform/jdk11-mybatis-plus-demo -am test
# 或看控制台分节输出
cd jdk11-platform
mvn -pl jdk11-mybatis-plus-demo -am spring-boot:run
```

数据库用 H2 内存库（`MODE=MySQL`，支持反引号、LIKE 大小写不敏感），无需 Docker。

## 2. 条件构造器 Wrapper

`apply("age between {0} and {1}", 18, 40)` 用占位符参数绑定防注入；`${ew.customSqlSegment}` 把 Wrapper 条件拼进手写 SQL（`UserMapper.selectByWrapper`）。

- `LambdaQueryWrapper` 在 VO 上无法建 lambda 缓存，联表分页用普通 `QueryWrapper` + 列名，且用表别名 `o.create_time` 避免歧义列。

## 3. 分页

`PaginationInnerInterceptor` 拦截 `IPage` 参数，自动 `limit` + 额外 `count`。

- 单表 `selectPage`；联表结果类型不限实体，`IPage<OrderUserVO>` 也行；排序列需用表别名限定。

## 4. 审计三件套（逻辑删除/乐观锁/自动填充）

- `@TableLogic` 把 DELETE 改写成 `SET deleted=1`。
- `@Version` 更新自动带 version 条件并自增（乐观锁）。
- `MetaObjectHandler` 在 insert/update 写入 `create_time/update_time/operator`。

## 5. 多租户隔离

`TenantLineInnerInterceptor` 仅对 `tenant_data` 注入 `tenant_id`（其余表 `ignoreTable` 返回 true 跳过），切换 `TenantContext` 即切换数据视野；INSERT 时也会自动写入当前租户值。

## 6. 动态表名分表

`DynamicTableNameInnerInterceptor` 按 `DynamicTableContext` 中的后缀把逻辑表 `log_record` 改写为 `log_record_2024/_2025`，业务只操作逻辑表。

## 7. 字段 AES 加密

`Member.phone` 标 `typeHandler=AesTypeHandler`，`autoResultMap=true` 保证「读取」也走 TypeHandler 解密；密钥硬编码仅为演示，生产用 KMS。

## 踩坑

- **JDK 版本坑**：MyBatis-Plus 自 3.5.9 起把分页/多租户等拦截器迁移到独立模块 `mybatis-plus-jsqlparser`，字节码要求 JDK 11+，无法在 JDK 8 编译运行——本 demo 因此落在 jdk11-platform。
- **联表分页用普通 QueryWrapper**：`LambdaQueryWrapper` 在 VO 上建不了 lambda 缓存，且排序列要用表别名限定避免歧义。
- **`autoResultMap=true` 必须开**：否则读取时不走 TypeHandler，解密失败拿到密文。
- **密钥硬编码仅演示**：生产字段加密密钥必须托管 KMS，且为按手机号查询额外存确定性哈希列。
- **演示逻辑放 main 而非 CommandLineRunner**：`@SpringBootTest` 加载上下文会自动执行所有 `CommandLineRunner`，某场景抛错会导致全部测试上下文初始化失败；放 `main()` 让单测干净加载、直接调用 Scenario 方法断言。

## 进阶方向

- 多租户：全局忽略表配置、跨租户 admin 操作、租户列统一治理。
- 动态表名：跨年/跨片查询的 union 聚合、历史数据归档。
- 分页：深翻页改用游标分页；联表分页 count 去重。
- 加密：改用 AES/GCM、密钥托管 KMS。
- 生产骨架：多数据源、读写分离、`mybatis-plus-generator` 代码生成器。

## 参考来源

- [MyBatis-Plus 官方文档](https://baomidou.com/)
