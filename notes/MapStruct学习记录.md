# MapStruct 学习记录

工程依托：[jdk8-mapstruct-demo](https://github.com/xieqiansong/chaos-java/tree/main/jdk8-platform/jdk8-mapstruct-demo)（`chaos-java` 仓库，MapStruct 1.5.5 常见用法速查，按能力分包覆盖基础/集合/嵌套/自定义映射，纯编译期零外部依赖）。

## 1. 安装

```bash
mvn -pl jdk8-platform/jdk8-mapstruct-demo test     # 12 条断言全过
# 控制台分节打印：mvn -pl jdk8-platform/jdk8-mapstruct-demo exec:java -Dexec.mainClass=lan.chaos.mapstruct.DemoApp
```

技术栈：MapStruct 1.5.5.Final + Lombok 1.18.30 + `lombok-mapstruct-binding` 桥接 + JUnit 5。

## 2. 基础同名映射（★★★）

`@Mapper` + 同名字段自动映射；嵌套对象（如 Address）字段同名自动递归。

- 反向映射：`@InheritInverseConfiguration` 复用正向配置（仅继承字段映射，表达式/常量需自行对齐）。
- 同类型深拷贝：`clone` 方法做对象深拷贝。

## 3. 集合映射（★★★）

定义单元素映射方法，List/Set 自动复用：`CollectionMapper` 调单元素 `toDto` 批量映射。集合元素类型须与单元素映射入参/出参一致。

## 4. 嵌套递归（★★★）

字段同名即自动递归，无需逐层声明，嵌套层级深时同样生效。

## 5. 自定义映射（★★☆）

| 能力 | 关键注解 / API | 生产坑点 |
|---|---|---|
| 重命名 | `@Mapping(source="username", target="account")` | 多个源字段映射到同一目标会编译报错 |
| 忽略 | `@Mapping(target="phone", ignore=true)` | 目标字段无源对应时**不写会报错**，必须显式 ignore |
| 常量 | `@Mapping(target="type", constant="SUMMARY")` | constant 与 source 互斥，不能同时写 |
| 默认值 | `@Mapping(source="level", target="grade", defaultValue="NORMAL")` | **仅在源字段为 null 时生效**；空串 `""`、0 不触发 |
| 表达式 | `@Mapping(target="displayName", expression="java(...)")` | 表达式内用 Java 全限定名，可读性差，慎用 |
| 日期格式化 | `@Mapping(dateFormat="yyyy-MM-dd")` | 源必须是 `Date`/`LocalDate`/`LocalDateTime`，否则转换失败 |
| 枚举转义 | `@Mapper(uses=MappingUtil.class)` + `@Named` + `qualifiedByName` | `MappingUtil` 方法须 `@Named` 标注，否则 `qualifiedByName` 找不到 |

## 踩坑

- **字段名/类型不一致不会自动映射**：必须 `@Mapping` 显式声明，否则目标字段为 null。
- **目标字段无源对应必须 ignore**：漏写会编译失败。
- **默认值只对 null 生效**：空串、0 不会触发 `defaultValue`。
- **constant 与 source 互斥**：不能同时写。
- **Lombok 桥接必引**：不引 `lombok-mapstruct-binding`，`@Data` 的 getter 在 MapStruct 处理时尚未生成，映射失败。
- **查看生成代码**：MapStruct 编译期生成 `BasicMapperImpl` 等，位于 `target/generated-sources`。

## 进阶方向

- 与 Spring 集成：真实项目用 `@Mapper(componentModel = "spring")`，后 `@Autowired` 注入（本 demo 用 `Mappers.getMapper(...)` + `INSTANCE` 免容器）。
- 选型对比：`BeanUtils.copyProperties`、ModelMapper 是运行时反射、性能差；MapStruct 编译期生成、零运行时反射、性能最佳，但需写 Mapper 接口。
- 拓展：`@AfterMapping`/`@BeforeMapping` 后置处理、多源对象合并映射、流式映射。

## 参考来源

- [MapStruct 官方文档](https://mapstruct.org/documentation/stable/reference/html/)
- [mapstruct-examples](https://github.com/mapstruct/mapstruct-examples)
