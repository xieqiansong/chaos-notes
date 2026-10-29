# WebFlux 响应式编程学习记录

工程依托：[jdk8-webflux-demo](https://github.com/xieqiansong/chaos-java/tree/main/engineering/webflux-demo)（`chaos-java` 仓库，Spring Boot 2.7.18 + spring-boot-starter-webflux（Netty 非阻塞）+ reactor-test，**纯内存零外部依赖**，StepVerifier / WebTestClient 双轨验证）。

> `chaos-notes` 此前完全没有响应式内容，本篇补齐「Reactor 类型与操作符 + 背压 + 两种端点写法 + WebClient」这一块。

## 1. 一句话定位

WebFlux = Spring MVC 的**非阻塞**替代：底层 Netty，全程不占业务线程，适合 IO 密集 / 高并发 / 流式推送。JDK 8 下 WebFlux 需 Spring Boot 2.x（本 demo 即用 2.7.18）。

```bash
mvn -pl jdk8-webflux-demo -am test                 # 核心验证，纯内存零依赖
mvn -pl jdk8-webflux-demo -am spring-boot:run      # 看 DemoApp 控制台「输入→输出」
```

## 2. Reactor 响应式类型与操作符（★★★）

引擎层。`Mono` = 0|1 个元素，`Flux` = 0..N 个元素，都是「声明式数据流」——**直到订阅才执行**，天然非阻塞、内建背压。

```java
Flux.fromIterable(products)
    .map(p -> p.withPrice(p.price() * 1.1))        // 同步转换
    .filter(p -> p.stock() > 0)                     // 过滤
    .flatMap(p -> remoteEnrich(p))                  // 异步展开（delayElement 模拟远程）
    .collectList();
```

- 创建：`Flux.fromIterable` / `Mono.justOrEmpty`
- 汇聚/终结：`collectList()` / `block()`（`block` 仅测试/演示，生产禁止）

## 3. 背压（★★★）

响应式区别于「回调 / 线程池」的核心：生产者可能远快于消费者，没背压会撑爆内存。Reactor 用 `request(n)` 让消费者**按需拉取**。

```java
// 拉不过来时的策略
onBackpressureBuffer(max, DROP_OLDEST)   // 缓存等待 / 丢最旧
onBackpressureDrop()                     // 直接丢弃
onBackpressureLatest()                   // 只留最新
```

验证：`BackpressureTest` 用 `BaseSubscriber` 钩子，`thenRequest(5)` 后只收 1..5，`thenRequest(15)` 后才收 6..20，证明背压生效。

## 4. RouterFunction 函数式路由（★★☆）

对 `@Controller` 注解的替代：路由与处理都是「函数」，类型安全、可组合、易测试，不走反射/注解扫描。

```java
@Bean
public RouterFunction<ServerResponse> routes(ProductHandler h) {
    return RouterFunctions.route()
        .GET("/api/products", h::list)
        .POST("/api/products", h::create)
        .build();
}
// Handler：每个方法收 ServerRequest，返回 Mono<ServerResponse>（非阻塞）
```

适合「端点少、要极致可控 / 可拼装」的场景。

## 5. 注解式响应式 Controller（对照）（★★☆）

`@RestController` + Mono/Flux 返回值，写法与 MVC 几乎一致，只是方法返回 Mono/Flux 即非阻塞——从 MVC 平滑迁移的形态。与 RouterFunction 二选一，两者底层同一套 WebFlux 引擎，路径不冲突即可共存。

```java
@GetMapping("/api/annotated/products")
public Flux<Product> list() { return repository.findAll(); }
```

## 6. WebClient 异步非阻塞调用（★★☆）

WebFlux 的声明式 HTTP 客户端。相比 RestTemplate（每请求占一个线程），WebClient 全程非阻塞（Netty），高并发省大量线程，返回 Mono/Flux 可链式 `flatMap`/`zip` 编排。

```java
webClient.get().uri("/api/products/{id}", id).retrieve()
          .bodyToMono(Product.class);
// 非阻塞 ≠ 无限等待：统一 responseTimeout（本 demo 3s），超时 Mono 以错误终结，调用方 onErrorResume 兜底
```

## 踩坑

- **`block()` 生产禁止**：只在测试/演示用，它会卡住事件循环线程。
- **非阻塞也要超时**：下游长时间不返回会一直占连接，必须配 `responseTimeout` + `onErrorResume`。
- **背压不是默认开启的魔法**：要在订阅端显式 `request(n)` 或用 `onBackpressure*` 策略才有意义。
- **与 MVC 选型**：IO 密集、高并发、要流式/背压 → WebFlux；简单 CRUD、要同步事务、团队熟 MVC → MVC（或虚拟线程），不必强行响应式。

## 进阶方向

- SSE / WebSocket 流式推送（`MediaType.TEXT_EVENT_STREAM` + `Flux`）。
- 响应式持久层：R2DBC / Reactive MongoDB / Reactive Redis，整条链路不阻塞。
- 背压落地策略：`onBackpressureBuffer(max, DROP_OLDEST)` 按业务取舍。
- 错误处理与超时：`onErrorResume` / `timeout` / `retryWhen`（指数退避）。
- WebFlux vs MVC 选型：见上方踩坑最后一条。

## 参考来源

- 工程：[jdk8-webflux-demo](https://github.com/xieqiansong/chaos-java/tree/main/engineering/webflux-demo)
- [Spring WebFlux 文档](https://docs.spring.io/spring-framework/reference/web/webflux.html)
- [Project Reactor 文档](https://projectreactor.io/docs)
