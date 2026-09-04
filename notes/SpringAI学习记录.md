# Spring AI 学习记录

工程依托：[jdk17-springai-demo](https://github.com/xieqiansong/chaos-java/tree/main/jdk17-platform/jdk17-springai-demo)（`chaos-java` 仓库，JDK17 + Spring Boot 3.5.14 + Spring AI 1.1.0，按「能力组/场景/单测」组织，每个场景一个 Service + 一个测试）。

## 1. 安装

模型端点支持两种模式，业务代码与测试完全不用改（DeepSeek 兼容 OpenAI 协议，只换连接信息）：

```bash
# 模式一：本地 llama.cpp（默认，零成本、离线）
llama-server.exe -m <model.gguf> --port 30040 -c 8192 -n 2048 -ngl 99 -t 8 -np 1 --reasoning off
mvn -pl jdk17-springai-demo test

# RAG 还需 embedding 服务（无论对话走云端还是本地，embedding 都用本地模型）
llama-server.exe -m <bge-m3.gguf> --port 30041 --embeddings --ubatch-size 2048

# 模式二：云端 DeepSeek
$env:DEEPSEEK_API_KEY="sk-xxxx"   # 只走环境变量，禁止写入 yml
mvn -pl jdk17-springai-demo test -Dspring.profiles.active=deepseek
```

> 实测对比（9B 本地 vs DeepSeek v4-flash）：多轮记忆 277s 失败 vs 1.9s；流式 35.9s 空流 vs 1.7s；冒烟对话 17.9s 不稳定 vs 2.3s。

## 2. 能力组

| 能力组 | 场景 | Service | 说明 |
|---|---|---|---|
| chat | 同步对话 | `BasicChatService` | 基础对话 |
| chat | 流式输出 | `StreamChatService` | SSE 流式 |
| chat | 多轮记忆 | `MemoryChatService` | 窗口式记忆 |
| chat | 提示词模板 | `PromptTemplateService` | ST4 模板渲染 |
| output | 结构化输出 | `StructuredOutputService` | Bean/List 绑定 |
| tools | 工具调用 | `ToolCallingService` | `@Tool` 定义与编排 |
| rag | 检索增强生成 | `RagService` | 读取/切分/入库/检索/生成 |
| rag | 生产向量库 | `RagService` + `VectorStoreConfig` | PgVector 持久化（pgvector profile） |
| mcp | MCP 协议 | `McpChatService` + 独立服务端 | 远程工具调用（mcp profile + 启动 `jdk17-mcp-server-demo`）|

## 3. 向量库切换（内存 ↔ PgVector）

RAG 存储层统一走 `VectorStore` 接口，换存储**不改一行业务代码**：

- 默认 `SimpleVectorStore`（内存版，零依赖、重启即丢）。
- 启用 `pgvector` profile → `PgVectorStore` 持久化到 Postgres（密码只走环境变量）。

```bash
$env:PG_PASSWORD="<你的密码>"
mvn -pl jdk17-springai-demo test -Dspring.profiles.active=deepseek,pgvector -Dtest=RagTest
```

## 踩坑（Spring AI 1.1.0）

| 问题 | 现象 | 解法 |
|---|---|---|
| 推理模型长思考 | 本地模型思考占满 token，`content` 空、流式空块 | llama.cpp `-n` 是**总输出**（含思考）；服务端加 `--reasoning off`。DeepSeek `max_tokens` 只限最终输出，不受影响 |
| `extraBody` 不生效 | 带与不带 `chat_template_kwargs` 耗时一致 | 1.1.0 中 `OpenAiChatModel` 未消费 `extraBody`，无法下发自定义字段 |
| 模板语法报错 | `{#if}` / `{var:默认值}` 失败 | 模板引擎是 ST4：变量 `{name}`、条件 `{if(cond)}...{endif}`，**无**默认值语法 |
| `getContent()` 不存在 | 编译失败 | 1.1.0 统一为 `getText()` |
| 记忆 Advisor 构造失败 | `new MessageChatMemoryAdvisor(chatMemory)` 参数不匹配 | 改用 `MessageChatMemoryAdvisor.builder(chatMemory).build()` |
| 请求 404 | `/v1/v1/chat/completions` | `base-url` 只填根地址，Spring AI 的 `completions-path` 自带 `/v1` |
| 工具调用"没生效" | 最终响应 `getToolCalls()` 空 | 工具调用发生在**中间轮次**，最终 `AssistantMessage` 只剩文本；用「回复里出现真实数据」验证 |
| 工具参数变 `arg0` | Function Calling 参数绑定失败 | `@Tool` 依赖编译期参数名，需 `-parameters`（本模块 pom 已配） |
| RAG 检索重复片段 | 同一上下文反复 ingest 累积重复 | 改 `@TestInstance(PER_CLASS)` + `@BeforeAll` 只入库一次 |
| MCP 客户端拖垮应用 | 未启动服务端时所有测试全红 | 默认 `spring.ai.mcp.client.enabled=false`，用 `mcp` profile 启用 |
| MCP 探活总失败 | HTTP 请求 `/sse` 超时 | `/sse` 长连接，改用 TCP 端口探测 |
| PgVector 默认激活 | 加依赖后所有测试全红 | 默认 `spring.ai.vectorstore.type: simple` 关闭；开关是 `pgvector` 且 `matchIfMissing=true` |
| PgVector 拖出 Hikari 报错 | 无数据源时上下文启动失败 | 主类排除 `DataSourceAutoConfiguration`，pgvector profile 下显式提供 `DataSource` |
| jdbcUrl 缺失 | "jdbcUrl is required" | 用 `DataSourceProperties#initializeDataSourceBuilder()`，裸 `DataSourceBuilder` 不翻译 `spring.datasource.url` |
| pgvector 命中不到 | 跨运行累积重复文档 | `@BeforeAll` 里 `TRUNCATE TABLE vector_store` 再入库 |

## 进阶方向

- 多模态、Agent 编排、对话历史持久化、评估（eval）与测试集。
- 生产化：模型路由、限流、降级（本地模型兜底）、敏感信息过滤。

## 参考来源

- [Spring AI 官方文档](https://docs.spring.io/spring-ai/reference/index.html)
- [Model Context Protocol](https://modelcontextprotocol.io/)
