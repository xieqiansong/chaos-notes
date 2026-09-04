# MCP 服务学习记录

工程依托：[jdk17-mcp-server-demo](https://github.com/xieqiansong/chaos-java/tree/main/jdk17-platform/jdk17-mcp-server-demo)（`chaos-java` 仓库，Spring AI MCP 的服务端模块，与 `jdk17-springai-demo` 的 `McpChatService` 客户端侧配合）。

> 本模块没有独立 README，内容依据 `jdk17-springai-demo` 的架构说明整理。MCP 是跨进程协议，服务端与客户端必须分开部署——这正是它能被多个 AI 应用共享的前提。

## 1. 定位

MCP（Model Context Protocol）把「工具（tool）」从 AI 应用里抽出来，放到一个**独立进程的服务端**暴露，AI 应用作为客户端通过协议调用。好处：工具实现可独立部署、升级、被多个 AI 应用共享，不侵入应用代码。

- 传输：SSE（Server-Sent Events），默认端口 **30052**。
- 关系：本服务端是被 `jdk17-springai-demo` 的 `McpChatService`（启用 `mcp` profile 时）远程调用的工具提供方。

## 2. 安装 / 运行

```bash
cd jdk17-mcp-server-demo
mvn -q package -DskipTests
java -jar target/jdk17-mcp-server-demo-1.0-SNAPSHOT.jar    # 端口 30052
```

客户端侧（在 `jdk17-springai-demo`）需先启动本服务端，并叠加 `mcp` profile 才能跑 MCP 场景：

```bash
mvn -pl jdk17-springai-demo test -Dspring.profiles.active=deepseek,mcp -Dtest=McpTest
```

## 3. 关键认知（踩坑）

- **服务端必须独立进程**：MCP 是跨进程协议，不是应用内库；客户端启动时若连不上服务端，Spring 上下文会启动失败（见 `jdk17-springai-demo` 踩坑：默认 `spring.ai.mcp.client.enabled=false`，用 `mcp` profile 启用）。
- **探活用 TCP 端口探测**：`/sse` 是长连接，用 HTTP 请求判断存活会一直挂起；应改 TCP 端口探测（见 `ModelEndpoint#isMcpServerUp`）。
- **SSE 传输**：服务端以 SSE 持续推送，客户端建立长连接接收工具调用结果。

## 进阶方向

- 工具粒度设计（无状态工具、参数 schema 定义）、多客户端共享同一服务端。
- 传输层换 stdio / 流式 HTTP；鉴权与多租户隔离。

## 参考来源

- [Model Context Protocol](https://modelcontextprotocol.io/)
- 配套客户端：`jdk17-springai-demo` 的 `McpChatService`
