# Nacos 学习记录

工程依托：[jdk8-nacos-demo](https://github.com/xieqiansong/chaos-java/tree/main/jdk8-platform/jdk8-nacos-demo)（`chaos-java` 仓库，注册中心 + 配置中心，多模块）。

两件事：服务注册发现（地址动态寻址）、配置中心（动态推送，改配置不重启）。

## 1. 安装

standalone 单节点，`v2.5.0`，映射 `8848`（HTTP）/ `9848`（gRPC 客户端）/ `9849`（gRPC 集群间）。数据源用的是 PostgreSQL（走插件），不是默认的 MySQL。

`docker-compose.yml`：

```yaml
services:
  nacos:
    image: nacos/nacos-server:v2.5.0
    container_name: nacos-server
    environment:
      MODE: standalone
    ports:
      - "8848:8848"
      - "9848:9848"
      - "9849:9849"
    volumes:
      - ./conf/application.properties:/home/nacos/conf/application.properties
      # - ./nacos-logs:/home/nacos/logs
      - ./plugins:/home/nacos/plugins
```

`conf/application.properties`（PostgreSQL 数据源）：

```properties
# PostgreSQL 数据源配置
spring.datasource.platform=postgresql
db.num=1
db.url.0=jdbc:postgresql://<postgresql-host>:30101/demo_nacos?currentSchema=public
db.user.0=postgres
db.password.0=postgres123
db.pool.config.driverClassName=org.postgresql.Driver
db.pool.config.connectionTimeout=30000
db.pool.config.validationTimeout=10000
db.pool.config.maximumPoolSize=50
```

`plugins/postgresql/` 下放了两个 jar：`nacos-datasource-plugin-postgresql-0.0.5.jar` + `postgresql-42.7.13.jar`（数据源插件来自 [nacos-datasource-plugin-pg](https://github.com/pig-mesh/nacos-datasource-plugin-pg.git) + PostgreSQL 官方驱动）。

`docker-compose up -d` 起来，控制台 `http://<ubuntu-ip>:8848/nacos`，HTTP API 走 `8848`。下面命令行均在 ubuntu 宿主机执行，连 `localhost:8848`。

## 2. 服务注册与发现

`jdk8-nacos-provider` 启动自动注册，`jdk8-nacos-consumer` 按服务名调用（用了 `@LoadBalanced` 的 `RestTemplate` 或 OpenFeign）。

```java
// Provider：配置里指定 server-addr + 服务名，启动即注册
//   spring.cloud.nacos.discovery.server-addr=localhost:8848
//   spring.application.name=jdk8-nacos-provider

// Consumer：按服务名调用，Nacos 客户端负责拉取/订阅实例列表
@Bean
@LoadBalanced
public RestTemplate restTemplate() {
    return new RestTemplate();
}

// 调用时写服务名而非 IP
restTemplate.getForObject("http://jdk8-nacos-provider/user/1", String.class);
```

```bash
# 对应 Nacos 指令（OpenAPI，在 ubuntu 宿主机执行）
# 查看已注册服务列表
curl -X GET 'http://localhost:8848/nacos/v1/ns/service/list?pageNo=1&pageSize=10'

# 查看某服务的健康实例
curl -X GET 'http://localhost:8848/nacos/v1/ns/instance/list?serviceName=jdk8-nacos-provider'
```

## 3. 配置中心（动态刷新）

`jdk8-nacos-config` 演示配置监听。`@RefreshScope` 注解的 Bean 字段随配置推送自动刷新，无需重启。

```java
// 配置类字段，Nacos 推送变更后自动刷新
@RefreshScope
@Component
public class DemoConfig {
    @Value("${demo.dynamic-key:default}")
    private String dynamicKey;
    // getter...
}

// 接口里直接读，能拿到最新值
@GetMapping("/key")
public String key() {
    return demoConfig.getDynamicKey();
}
```

```bash
# 对应 Nacos 指令（OpenAPI 发布/获取配置）
# 发布配置（dataId=demo-config, group=DEFAULT_GROUP）
curl -X POST 'http://localhost:8848/nacos/v1/cs/configs' \
  -d 'dataId=demo-config&group=DEFAULT_GROUP&content=demo.dynamic-key=hello'

# 获取当前配置
curl -X GET 'http://localhost:8848/nacos/v1/cs/configs?dataId=demo-config&group=DEFAULT_GROUP'
```

## 4. 编程式监听（自定义热更新逻辑）

`@RefreshScope` 只能刷新字段；要"配置变了顺带干点别的"（重建线程池、重载限流规则、刷本地缓存），用 `NacosConfigManager` 拿 `ConfigService` 手动 `addListener`。

```java
// 等应用就绪再注册，避免 Nacos Bean 未初始化导致空指针
@EventListener(ApplicationReadyEvent.class)
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

```bash
# 对应 Nacos 指令（改配置即触发上面的 receiveConfigInfo 回调）
curl -X POST 'http://localhost:8848/nacos/v1/cs/configs' \
  -d 'dataId=demo-config&group=DEFAULT_GROUP&content=demo.dynamic-key=changed'
# 监听线程会打印：[ConfigListener] 配置变更: demo.dynamic-key=changed
```

## 5. 几个注意点

- **data-id / group**：data-id 默认取 `应用名.后缀`；group 默认 `DEFAULT_GROUP`，可做环境/租户隔离。编程式监听时 dataId/group 从配置注入，确保监听的是对的配置集。
- **注册与配置用同一 server-addr**：`discovery` 和 `config` 的 `server-addr` 必须一致，否则注册和拉配置指向不同集群。
- **配置优先级**：Nacos 配置 > 本地 `bootstrap.yml` > `application.yml`，排查"配置没生效"先确认来源。
- **监听要在就绪后注册**：用 `ApplicationReadyEvent` 触发 `addListener`，否则 Nacos Bean 还没初始化会空指针。

## 参考来源

- 工程：[jdk8-nacos-demo](https://github.com/xieqiansong/chaos-java/tree/main/jdk8-platform/jdk8-nacos-demo)（provider/consumer/config 三模块）
- PostgreSQL 数据源插件：[nacos-datasource-plugin-pg](https://github.com/pig-mesh/nacos-datasource-plugin-pg.git)（安装段 `plugins/postgresql/` 的 jar 来源）
- 同类配置中心对比：Apollo（更强调权限/灰度）、Spring Cloud Config（Git 后端，无动态推送）
