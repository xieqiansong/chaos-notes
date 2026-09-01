# 热路径 Servlet Filter 异步化：高频接口绕过 SpringMVC

> 性能优化 · 实战案例。对应 GitHub 工程：[chaos-java/jdk8-platform/jdk8-servlet-filter-async-demo](https://github.com/xieqiansong/chaos-java/tree/master/jdk8-platform/jdk8-servlet-filter-async-demo)。

高频「收即 ack」型接口（心跳 / 状态上报 / 埋点回执）往往占整体 QPS 的大头，但真正有价值的业务处理（落库、Bitmap 置位、入批队列）其实可以延后。问题是：这类接口如果走完整的 SpringMVC 链路，每一步都是白付的 CPU 与反射开销，而且**并发上限被 Tomcat 线程池大小死死锁住**——一个慢下游就能占满线程、拖垮所有接口。

这个 demo 把最高频的 `/api/report` 前置到 `Filter` 层，在 `DispatcherServlet` 之前手动读流、校验、异步提交即返回，Tomcat 线程即时释放。同一份二进制、仅靠 `app.mode` 切换，就能和完整 MVC 链路做公平对照，把「绕过 MVC 省下的链路开销」量化出来。

## 现状与瓶颈

- **实现**：`@PostMapping("/api/report")` 走 `DispatcherServlet` → `HandlerMapping` → `@RequestBody` 反序列化/校验 → `HandlerInterceptor` 链 → `HttpMessageConverter` 写回。
- **瓶颈**：
  1. **链路 CPU 浪费**：高频「收即 ack」接口本质上只需要「读流 → 取 `id` → 提交即返回」，却要为全量参数绑定、拦截器链、消息转换付出固定开销。
  2. **并发上限受 Tomcat 线程池束缚**：每个请求占一个 Tomcat 线程直到响应写出。下游一慢，线程被占满，所有接口一起排队——即使本接口本身不需要同步等下游。

核心矛盾：**「立即 ack」和「同步等链路收尾」是两件事，但完整 MVC 把两者绑在了一条线程上。**

## 替换方案

### 设计

把热路径从 MVC 链路里「剪」出来，改在 `Filter` 层闭环：

1. **前置到 Filter 层**：`FilterRegistrationBean` 注册为 `Ordered.HIGHEST_PRECEDENCE`、`addUrlPatterns("/*")`，先于 `DispatcherServlet` 与所有 `HandlerInterceptor` 执行（`FilterConfig`）。
2. **手动读流 + 最小化反序列化**：直接 `req.getInputStream()` 用 `ObjectMapper` 只反序列化必要字段（`StatusReport` 仅 `id` + `payload`），避免 Spring 全量参数绑定（`EarlyReportFilter.handleHotPath`）。
3. **校验前置（同步拒绝）**：缺 `id` 立即 `SC_BAD_REQUEST` 返回，不浪费异步线程（`EarlyReportFilter` 第 74-77 行）。
4. **异步提交即返回**：`CompletableFuture.runAsync(() -> reportService.submitReport(...), reportExecutor)` 把实际处理甩给独立线程池，紧接着 `resp.setStatus(SC_OK)`——**此后 `return`，不再调 `chain.doFilter`**（`EarlyReportFilter` 第 81-91 行）。
5. **慢请求可观测**：非热路径（或基线模式）照常 `chain.doFilter`，收尾时 `>100ms` 打 warn（`EarlyReportFilter` 第 62-65 行）。

效果：**Tomcat 线程即时释放 → 并发上限不再由 Tomcat 线程池决定，而由异步队列 + 下游批量引擎吞吐决定。**

### 隔离线程池（关键，不是顺手 new 一个）

`ReportExecutorConfig` 用 `ThreadPoolTaskExecutor`，刻意**不依赖 `ForkJoinPool.commonPool`**：

- `corePoolSize=8` / `maxPoolSize=32` / `queueCapacity=65536`：弹性承接上报洪峰。
- `CallerRunsPolicy`：队列满时由提交线程自己执行，天然背压（降级而非丢弃）。
- `setWaitForTasksToCompleteOnShutdown(true)` + `awaitTerminationSeconds(30)`：优雅停机，避免下线丢在途任务。

### 公平对照机制

`EarlyReportFilter` 由 `app.mode` 控制是否短路（`EarlyReportFilter` 第 56 行）：

- `mode=filter-async`：命中 `/api/report` 则自己处理并 `return`，绕开 MVC。
- `mode=controller-sync`（基线）：走 `chain.doFilter`，进入 `ReportController` 完整 MVC 链路。

二者**下游处理完全一致**——都是 `ReportService.submitReport` → `ReportSink.submit`（异步提交即返回，仅 `AtomicLong` 计数）。唯一差异是「是否被整条 MVC 链路包裹」，因此吞吐 / p50-p99 / Tomcat 忙线程 / 进程 CPU 的差值，就是「绕过 MVC 省下的链路开销」。

## 压测方案

做成 **SpringBootTest 一键跑**（`BenchMarkTest`）：`mvn -pl jdk8-servlet-filter-async-demo test` 启动两次内嵌 Tomcat（先 `controller-sync` 再 `filter-async`），各自用并发 HTTP 客户端打 `/api/report`，结果写出 `target/bench-results.md`。

- **下游刻意极轻**：`ReportSink.submit` 只做 `accepted.incrementAndGet()`，聚焦「链路」CPU 成本，与入库/业务耗时无关。这是本压测的核心前提——它度量的是「链路 overhead」，不是「业务 overhead」。
- **闭环负载**：`CONCURRENCY=64` 个固定线程持续打 12s，记「提交 → 完成」端到端延迟（含排队）。
- **指标**：吞吐(req/s)、p50/p99、errors、Tomcat 忙线程峰值（轮询 `Tomcat:type=ThreadPool` MBean 的 `currentThreadsBusy`）、进程 CPU（`com.sun.management.OperatingSystemMXBean.getProcessCpuLoad()`）。
- **环境**：Win11 / JDK 21 运行 Spring Boot 2.7.18（工程基于 JDK8 平台 API，但用 JDK21 跑以贴近现代运行环境）/ 64 并发 / 12s。

## 实测结果与解读

| mode | req/s | p50(ms) | p99(ms) | errors | maxBusyThreads | cpuPct(%) |
|------|-------|---------|---------|--------|----------------|-----------|
| controller-sync | 4220.4 | 15 | 20 | 0 | 10 | 35.7 |
| filter-async | 4307.5 | 15 | 18 | 0 | 11 | 25.9 |

要点：

- **下游极轻时，Tomcat 线程没被打满**（busy 都只有 10~11），所以「解耦 Tomcat 线程」这个最大卖点**在此场景不显现**——这正是压测设计的诚实之处：它隔离了「链路开销」这一单变量。
- **但进程 CPU 从 35.7% 降到 25.9%**（demo 自身总结为「约省 40% 进程 CPU」）：这是「绕过 MVC 省下的链路开销」最直接的证据——同样的请求量，少了整条 MVC 链路的反射/绑定/转换。
- 吞吐 +3%、p99 略好（20→18ms）：链路省下的 CPU 回流到处理能力，尾延迟更稳。

### 为什么这个收益会随下游变重而放大

上述数据是在「下游极轻」前提下测的，只暴露了「链路 CPU 节省」这一层收益。真正的杠杆在 **并发上限**：

- 若下游变重（瓶颈回到 Tomcat 线程），`controller-sync` 模式下慢下游会占满 Tomcat 线程、所有接口排队；而 `filter-async` 模式下 Tomcat 线程在 `setStatus(SC_OK)` 后就释放，**并发能力不再受 Tomcat 线程池束缚**，由异步队列 + 下游批量引擎吞吐决定。
- 换句话说：本压测量化的是「小收益」（省链路 CPU）；「解耦 Tomcat 线程」的大收益在重下游 / 高抖动场景下才完全释放，但压测设计上无法在不改变下游的前提下同时呈现——这点必须说清楚，避免误导。

## 注意事项（踩坑，比收益更重要）

1. **不要用 `ForkJoinPool.commonPool`**：示例用自定义隔离线程池 + `CallerRunsPolicy` 背压。`commonPool` 并行度 = 核数-1，是网关类纯转发场景里被全局争抢的共享资源，一旦被占满会拖垮所有依赖它的地方。
2. **`getInputStream()` 只能读一次**：本模式提前 `return`、不调 `chain`，所以无冲突；但只要热路径还让后续组件读流，就会 `IllegalStateException`。务必确保短路后不再有下游读流。
3. **异步任务失败，客户端已 200**：用 `whenComplete` 兜底（日志 + 失败计数，`ReportSink.failed` / `EarlyReportFilter` 第 83-88 行），**不能回写响应**。失败语义是「已 ack 但处理丢了」，需配合死信/重试，绝不能假装成功。
4. **异步内不得用请求作用域**：`request` / `session` / `@RequestScope` Bean 在异步线程不可用。所有需要的信息（`clientIp`、`id`）必须在提交前捕获为局部变量（`EarlyReportFilter` 第 78 行先 `req.getRemoteAddr()`）。
5. **优雅停机**：`setWaitForTasksToCompleteOnShutdown(true)` + `awaitTerminationSeconds`，避免发布/缩容时丢失在途异步任务（见线程池配置）。

## 适用边界

- **适合**：高频「收即 ack」型接口（心跳、状态上报、埋点回执、异步回执），特征是「校验轻、处理可延后、不依赖同步响应体」。
- **不适合**：需要同步返回业务结果（查询、写后读、强一致回写）的接口——这类接口本就走完整 MVC，强拆到 Filter 反而丢失了 Spring 的校验/序列化/异常处理体系。
- **本质**：用「提前短路 + 独立线程池异步」把「立即 ack」与「链路收尾」解耦，代价是放弃 MVC 生态（校验、异常处理器、响应体框架），只在热路径上做，非热路径保持标准 MVC。

> 参考：工程内 `README.md` 与本笔记互链；压测原始数据由 `mvn test` 生成于 `target/bench-results.md`。
