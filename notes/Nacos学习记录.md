# Nacos 学习记录

> 工程依托：[jdk8-nacos-demo](https://github.com/xieqiansong/chaos-java/tree/main/jdk8-platform/jdk8-nacos-demo)（`chaos-java` 仓库，注册中心 + 配置中心，多模块）。
> 记录 Nacos 两大核心能力：服务注册发现、动态配置。

## 1. Nacos 的两个角色

- **注册中心**：服务启动时把自己的地址注册到 Nacos，消费者通过服务名发现实例、做负载均衡调用（替代写死 IP）。
- **配置中心**：配置集中管理、动态推送，改配置不用重启应用。

## 2. 服务注册与发现

`jdk8-nacos-provider` / `jdk8-nacos-consumer` 演示注册发现：

- Provider 启动后自动注册（`spring.cloud.nacos.discovery.server-addr` + `service` 名）。
- Consumer 用 `@LoadBalanced` 的 `RestTemplate` 或 `OpenFeign` 按服务名调用，Nacos 客户端负责拉取/订阅实例列表。
- `UserClient`（common 模块）定义服务间契约，体现「服务名寻址」。

**要点**：注册发现解决的是「实例地址动态变化」问题——扩缩容、重启后消费者自动感知，无需改配置。

## 3. 配置中心（动态刷新）

`jdk8-nacos-config` 演示配置监听：

### 3.1 自动刷新 `@RefreshScope`

```java
@Value("${demo.dynamic-key:default}")
private String dynamicKey;
```

配合 `@RefreshScope` 的 Bean，Nacos 推送配置变更后字段自动刷新，无需重启。

### 3.2 编程式监听（自定义热更新逻辑）

`ConfigListener` 通过 `NacosConfigManager` 拿到底层 `ConfigService`，手动 `addListener`：

```java
@EventListener(ApplicationReadyEvent.class)   // 确保 Nacos Bean 已就绪
public void registerListener() throws NacosException {
    nacosConfigManager.getConfigService().addListener(dataId, group, new Listener() {
        @Override public Executor getExecutor() { return null; } // 用 Nacos 内置线程池
        @Override
        public void receiveConfigInfo(String configInfo) {
            log.info("[ConfigListener] 配置变更: {}", configInfo);
            // TODO 执行自定义热更新：重建线程池 / 重新加载限流规则 / 刷新本地缓存
        }
    });
}
```

**适用场景**：配置变更时要执行自定义逻辑——热更新线程池核心/最大线程数、重新加载限流或路由规则、刷新本地缓存、重建客户端连接。

## 4. data-id 与 group

- **data-id**：配置项标识，默认取 `${spring.application.name}.${file-extension}`。
- **group**：分组，默认 `DEFAULT_GROUP`，用于环境/租户隔离。
- `ConfigListener` 的 `dataId`/`group` 从配置注入，可覆盖，确保监听的是正确的配置集。

## 5. 学习踩坑点

- **监听要在应用就绪后注册**：用 `ApplicationReadyEvent` 触发 `addListener`，避免 Nacos Bean 未初始化导致空指针。
- **@RefreshScope 作用域**：只刷新标注了该注解的 Bean，且是「下次注入时生效」，不是热替换已有实例字段。
- **配置优先级**：Nacos 配置 > 本地 `bootstrap.yml` > `application.yml`，排查「配置没生效」时先确认来源。
- **注册与配置用同一 server-addr**：`discovery` 与 `config` 的 `server-addr` 要一致，否则服务注册和配置拉取指向不同集群。

## 6. 参考来源

- 工程：[jdk8-nacos-demo](https://github.com/xieqiansong/chaos-java/tree/main/jdk8-platform/jdk8-nacos-demo)（provider/consumer/config 三模块）
- 同类配置中心对比：Apollo（更强调权限/灰度）、Spring Cloud Config（Git 后端，无动态推送）
