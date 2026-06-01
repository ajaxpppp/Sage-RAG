# Ragent 项目深度解析与面试指南

> 本文档以仓库主干（mading/main）的真实代码与配置为唯一事实来源，所有版本、端口、表名、类名均与 `bootstrap/src/main/resources/application.yaml`、`resources/database/schema_pg.sql`、`pom.xml` 及源码对齐。

## 一、项目概述

### 1.1 项目名称与定位
- 项目名称：Ragent（RAG + Agent）
- 定位：企业级 Agentic RAG 智能体平台
- 一句话介绍：基于 Spring Boot 3.5.7 + React 18 + PostgreSQL(pgvector) 构建的企业级检索增强生成（RAG）智能体平台，支持多轮对话、查询改写、三级意图识别、多通道检索、Rerank 精排、MCP 工具调用、模型路由三态熔断、分布式队列限流与全链路追踪。

### 1.2 项目背景
- 来源：nageoffer（拿个 offer）社群开源项目。
- 解决的问题：企业内部知识库问答场景中，传统关键词搜索无法理解用户意图，大模型又存在幻觉问题。Ragent 通过 RAG 架构将企业私有知识与大模型能力结合，实现精准、可控、可溯源的智能问答，并通过 MCP（Model Context Protocol）让对话能直接调用企业内部系统（销售、工单、天气等），实现"查知识"与"调系统"的融合。
- 技术趋势：RAG + Agent 是当下 AI 应用落地的主流架构，该项目完整覆盖 RAG 全链路 + Agent 工具调用，是面试中展示 AI 工程化能力的理想项目。

### 1.3 技术栈总览

后端：
- Java 17 + Spring Boot 3.5.7 + Maven 多模块（framework / infra-ai / bootstrap / mcp-server）
- 业务库：**PostgreSQL（启用 pgvector 扩展）** + HikariCP，连接 `jdbc:postgresql://127.0.0.1:25432/ragent`，账号 `postgres/postgres`。**不是 MySQL**。
- 向量存储：可切换 `rag.vector.type = pg（默认）| milvus`，dimension=1536，metric=COSINE。Milvus（可选）`http://localhost:29530`。
- Redis 26379（password 123456）+ Redisson 4.0.0：缓存 / 分布式锁 / 队列限流 / Pub/Sub。
- RocketMQ 29876（rocketmq-spring-boot-starter 2.3.5）：异步消息。
- 对象存储：**RustFS（S3 兼容）** `http://localhost:29000`，`rustfsadmin/rustfsadmin`（AWS S3 SDK 2.40.2）。
- Sa-Token 1.43.0（认证鉴权）+ MyBatis-Plus 3.5.14（ORM，含 jsqlparser）。
- Apache Tika 3.2.3（文档解析）、Hutool 5.8.37、OkHttp 4.12.0、transmittable-thread-local 2.14.5。
- MCP SDK 1.1.2、Milvus SDK 2.6.6。
- 注意：项目通过自研的 `infra-ai` 抽象层直连各厂商 OpenAI 兼容接口，**未引入 Spring AI**。

前端：
- React 18 + TypeScript + Vite 5 + React Router 6 + Zustand（状态管理）
- Radix UI + TailwindCSS（**非 Ant Design**）+ Axios + Recharts + react-markdown + react-virtuoso + sonner

AI 模型（`ai.providers`，**不含 DuoJie/多杰**，那是本地改动）：
- 供应商：ollama（本地 11434）、bailian（阿里百炼 dashscope）、aihubmix、siliconflow。
- Chat 候选（按 priority）：qwen-plus(bailian,p1)、qwen3-local(ollama,p2)、qwen3-max(bailian,supports-thinking,p3)、glm-4.7(siliconflow,Pro/zai-org/GLM-4.7,supports-thinking,p4)、gpt-5.4(aihubmix,p5)。`default-model` 与 `deep-thinking-model` 均为 **qwen3-max**。
- Embedding 候选：qwen-emb-8b(siliconflow,Qwen/Qwen3-Embedding-8B,p1)、qwen-emb-local(ollama,p2)、text-embedding-3-large(aihubmix,p3)，dimension=1536。
- Rerank 候选：qwen3-rerank(bailian,p1)、rerank-noop(noop,p100)。

### 1.4 端口与认证
- bootstrap 主服务：**29090**，context-path `/api/ragent`。
- mcp-server：**29099**（不是 9091）。
- 前端 dev（Vite）：**25173**，代理 `/api` → `http://localhost:29090`，`.env` 的 `VITE_API_BASE_URL=/api/ragent`。
- 认证：Sa-Token，header `Authorization`，timeout 2592000，token-style simple-uuid，初始账号 **admin / admin（admin 角色）**。

### 1.5 项目规模
- 4 个 Maven 模块：framework（基础设施）、infra-ai（AI 抽象层）、bootstrap（业务逻辑）、mcp-server（MCP 工具服务）+ frontend（React 前端）。
- 21 张数据库表（`resources/database/schema_pg.sql`）。

---

## 二、项目架构深度解析

### 2.1 模块划分

```
ragent/
├── framework/          # 基础设施（异常体系、幂等、分布式ID、链路追踪、SSE、上下文、缓存、MQ）
├── infra-ai/           # AI 抽象层（Chat/Embedding/Rerank 客户端、模型路由、三态熔断）
├── bootstrap/          # 核心业务（RAG 对话、知识库、文档摄取、意图、用户、Dashboard）
├── mcp-server/         # MCP 工具服务（销售/工单/天气，JSON-RPC 端点，独立 29099 端口）
└── frontend/           # React 前端应用
```

#### 2.1.1 framework —— 基础设施模块（与业务无关，可复用）

锚点包：`com.nageoffer.ai.ragent.framework.*`

- **convention**：统一返回体 `Result<T>`（code / message / data / requestId / success）。
- **web**：`Results` 静态工厂、`GlobalExceptionHandler` 全局异常处理、SSE 封装 `SseEmitterSender`（统一封装流式事件发送、错误兜底、超时关闭）。
- **exception + errorcode**：三级异常体系 `ClientException`（客户端错误，4xx 语义）/ `ServiceException`（业务异常）/ `RemoteException`（下游/远程调用异常），配合 `IErrorCode` 错误码枚举，异常携带错误码，由 `GlobalExceptionHandler` 转 `Result`。
- **context**：`UserContext`（基于 TTL ThreadLocal 的当前登录用户）、`LoginUser`。
- **idempotent**：`@IdempotentSubmit` 注解 + AOP + Redis Lua 脚本实现幂等提交（防重复提交）。
- **trace**：`RagTraceContext` 链路上下文 + `@RagTraceRoot`（开启一次 trace 根）/`@RagTraceNode`（标记节点）注解 + AOP，自动记录每个节点的耗时、输入输出、错误，落到 `t_rag_trace_run` / `t_rag_trace_node`。
- **distributedid**：Snowflake 雪花算法分布式 ID，`lua/snowflake_init.lua` 在 Redis 中分配 workerId，保证全局唯一、趋势递增。
- **cache / mq**：缓存与 RocketMQ 封装。
- **线程池**：所有线程池统一用 `TtlExecutors` 包装（transmittable-thread-local 2.14.5），跨线程透传 `UserContext` 与 `RagTraceContext`，解决线程池场景下 ThreadLocal 丢失问题。

面试可讲：framework 把"与业务无关的横切能力"沉淀为独立模块——统一返回体、三级异常体系、幂等、雪花 ID、链路追踪、TTL 上下文传播，体现了对工程规范和可复用基础设施的把控。

#### 2.1.2 infra-ai —— AI 抽象层模块

锚点包：`com.nageoffer.ai.ragent.infra.*`

- **chat**：`ChatClient` 接口 + `AbstractOpenAIStyleChatClient` 基类（封装 OpenAI 兼容协议的请求构造、流式 SSE 解析），具体实现 `BaiLianChatClient` / `SiliconFlowChatClient` / `AIHubMixChatClient` / `OllamaChatClient`；上层服务 `LLMService` 与路由实现 `RoutingLLMService`；流式回调 `StreamCallback`、取消句柄 `StreamCancellationHandle`（支持中途停止生成）。
- **embedding**：`EmbeddingClient` 接口 + `RoutingEmbeddingService`（按候选优先级 + 熔断路由生成向量）。
- **rerank**：`RoutingRerankService`（qwen3-rerank 为主，rerank-noop 兜底）。
- **model**：模型路由与熔断核心，`ModelSelector`（按 priority 选候选）、`ModelRoutingExecutor`（依次尝试 + 熔断隔离）、`ModelHealthStore`（三态熔断器状态机）、`ModelTarget`（一次调用目标：provider + model + endpoint）。
- **config**：`AIModelProperties` 映射 `ai.*` 配置（providers / chat / embedding / rerank / selection / stream），供 `RAGSettingsController` 查询展示。

面试可讲：infra-ai 把"模型能力"抽象成 ChatClient/EmbeddingClient/RerankClient 三类接口，业务层只依赖 `Routing*Service`，底层的厂商切换、优先级路由、熔断对业务完全透明——典型的依赖倒置 + 策略模式。

#### 2.1.3 bootstrap —— 核心业务模块

锚点包：`com.nageoffer.ai.ragent.*`，启动类 `RagentApplication`。业务域：admin / user / rag / knowledge / ingestion / core。

- **rag.controller**：`RAGChatController`（`GET /rag/v3/chat` SSE 流式对话 + 停止任务接口）、`ConversationController`、`SampleQuestionController`、`RAGSettingsController`、`RagTraceController`、`IntentTreeController`。
- **rag.service**：`RAGChatServiceImpl.streamChat`（核心入口）、`StreamChatPipeline`（编排 memory→rewrite→intent→guidance→retrieve→prompt→LLM 的责任链/模板）、`ConversationService`、`ConversationMessageService`、`ConversationMemoryService`。
- **rag.core**（RAG 算法核心，五大扩展点都在这里）：
  - memory：`ConversationMemoryService` + `ConversationMemoryStore`（`JdbcConversationMemoryStore` 实现），滑动窗口（history-keep-turns=4）+ 自动摘要（summary-start-turns=5、summary-enabled=true、summary-max-chars=200）。
  - rewrite：`QueryRewriteService.rewriteWithSplit`，上下文补全规范化 + 子问题拆分。
  - intent：`IntentClassifier` / `DefaultIntentClassifier`、`IntentResolver`、`IntentTreeFactory`，三级意图树并发分类。
  - retrieve：`RetrievalEngine` 接口 + `MultiChannelRetrievalEngine` 实现；`SearchChannel`（`VectorGlobalSearchChannel`、`IntentDirectedSearchChannel`）；`SearchResultPostProcessor`（`DeduplicationPostProcessor`、`RerankPostProcessor`）；底层向量检索 `RetrieverService`（`PgRetrieverService`、`MilvusRetrieverService`，由 `rag.vector.type` 决定）。
  - guidance：`IntentGuidanceService.detectAmbiguity`，歧义时主动澄清。
  - prompt：`RAGPromptService.buildStructuredMessages` + `PromptTemplateLoader`（模板在 `resources/prompt`）。
  - mcp：`McpToolExecutor` 接口 + `McpToolRegistry` + `McpClientToolExecutor`，作为 MCP 客户端 HTTP 调用 mcp-server。
- **rag.aop**：`@ChatRateLimit` 注解 + 请求队列控制 + Trace 切面。
- **ingestion**：`IngestionPipelineController`、`IngestionTaskController`、`engine(IngestionEngine)`、`node(IngestionNode：FetcherNode / ParserNode / ChunkerNode / EnhancerNode / EnricherNode / IndexerNode)`、`domain`、`prompt`。
- **knowledge**：`KnowledgeBaseController`、`KnowledgeDocumentController`、`KnowledgeChunkController`、`KnowledgeDocumentScheduleJob`（`@Scheduled` 定时扫描刷新文档）。
- **core**：`parser`（Apache Tika 解析 PDF/Office/Markdown）、`chunk`（按标题/长度/段落等切分策略）。
- 预置知识库文档在 `resources/file`，prompt 模板在 `resources/prompt`。

#### 2.1.4 mcp-server —— MCP 工具服务模块

锚点包 + 启动类 `McpServerApplication`（端口 **29099**）。

- **executor**：`SalesMcpExecutor`（销售数据）、`TicketMcpExecutor`（工单）、`WeatherMcpExecutor`（天气）——三个内置工具示例。
- **endpoint**：JSON-RPC 接收端点，按工具名路由到对应 executor。
- **core**：工具元模型 / 请求响应模型 / 执行器接口。
- bootstrap 通过 `rag.core.mcp` 客户端（`rag.mcp.servers` 配置 `http://localhost:29099`）作为 MCP 客户端调用它。

面试可讲：mcp-server 是独立部署的 MCP 工具网关，bootstrap 在对话时把"系统类意图"映射成工具调用，用 LLM 按工具 schema 抽参，再 HTTP/JSON-RPC 调过来——这就是 Agent 的工具使用能力。

### 2.2 五大可扩展点（面试高频）

项目用接口 + 注册表把可变逻辑抽象为扩展点，新增能力无需改核心代码：

1. **SearchChannel**（`rag.core.retrieve.channel`）——检索通道。实现 `VectorGlobalSearchChannel`、`IntentDirectedSearchChannel`，新增检索策略（如 BM25 关键词通道）只需实现接口并注册。
2. **SearchResultPostProcessor**（`rag.core.retrieve.postprocessor`）——检索后处理。责任链：`DeduplicationPostProcessor`（去重）→ `RerankPostProcessor`（Rerank 精排），可继续追加过滤、截断等处理器。
3. **McpToolExecutor**（`rag.core.mcp`）——工具执行器。`McpClientToolExecutor` 经 `McpToolRegistry` 注册，扩展新工具即新增执行器。
4. **IngestionNode**（`ingestion.node`）——摄取流水线节点。Fetcher/Parser/Chunker/Enhancer/Enricher/Indexer 均实现统一节点接口，按图编排可任意替换/插入节点。
5. **ChatClient**（`infra.chat`）——大模型客户端。`AbstractOpenAIStyleChatClient` + 四个厂商实现，接入新厂商只需继承基类。

### 2.3 模型路由 + 三态熔断器（infra-ai/model）

核心类：`ModelSelector`、`ModelRoutingExecutor`、`ModelHealthStore`、`ModelTarget`。配置 `ai.selection`：**failure-threshold=2、open-duration-ms=30000**。

三态状态机（`ModelHealthStore` 内 `State` 枚举，已核对源码）：
- **CLOSED**（正常）：放行；连续失败累计到 failure-threshold(=2) → 转 OPEN，设 `openUntil = now + open-duration-ms`。
- **OPEN**（熔断）：`openUntil` 未到则直接拒绝（跳过该模型，路由到下一候选）；到期后转 HALF_OPEN 并放行一次探测（`halfOpenInFlight`）。
- **HALF_OPEN**（半开）：仅放行一个探测请求；探测成功 → markSuccess 转 CLOSED 并清零失败计数；探测失败 → markFailure 退回 OPEN 并重置 `openUntil`。

`ModelRoutingExecutor` 按 `ModelSelector` 给出的优先级候选列表依次尝试，跳过 OPEN 状态的模型，遇异常 `markFailure`、成功 `markSuccess`，实现故障自动隔离与恢复。每个模型 id 维护独立健康状态，互不影响。

面试话术：
> "模型路由用 ModelSelector 按 priority 排候选，ModelRoutingExecutor 依次尝试。每个模型由 ModelHealthStore 维护一个三态熔断器：CLOSED 下连续失败 2 次转 OPEN 拒绝请求，30 秒后转 HALF_OPEN 放行一次探测，成功回 CLOSED、失败退回 OPEN。这样某个厂商抖动时会被自动隔离并路由到下一候选，对用户透明。"

### 2.4 分布式队列式限流（rag.rate-limit.global）

配置：enabled=true、max-concurrent=10、max-wait-seconds=15、lease-seconds=30、poll-interval-ms=200。由 `@ChatRateLimit` + rag.aop 队列控制器实现。

机制（Redis 信号量 + ZSET 排队 + Pub/Sub 通知 + Lua 原子判断）：
- **Redis 信号量**控制全局最大并发（max-concurrent），lease-seconds 为持有租约（防止持有者宕机导致名额泄漏）。
- **ZSET 排队**：score 为时间戳，请求入队天然有序，公平排队；max-wait-seconds 超时则放弃。
- **Lua 脚本**：把"判断名额 + 占用 / 出队"等多步操作做成原子判断，避免竞态。
- **Pub/Sub 通知**：名额释放后通知排队节点尝试获取，配合 poll-interval-ms 轮询兜底。
- 配合 `@IdempotentSubmit` 防重复提交。

面试话术：
> "大模型调用慢且贵，所以做了分布式队列限流：Redis 信号量控制全局并发上限，ZSET 按时间戳排队保证公平，Lua 脚本保证占名额/出队的原子性，名额释放用 Pub/Sub 通知 + 轮询兜底，租约机制防止持有者宕机泄漏名额。既控住了成本，又保证排队体验。"

### 2.5 多路检索 + 后处理流水线（rag.core.retrieve）

编排器 `MultiChannelRetrievalEngine` 实现 `RetrievalEngine`：
- **IntentDirectedSearchChannel**（意图定向）：基于意图识别命中的节点 collection 定向检索；配置 `rag.search.channels.intent-directed`：min-intent-score=0.4、top-k-multiplier=2。
- **VectorGlobalSearchChannel**（全局向量兜底）：当意图置信度不足（confidence-threshold=0.6）时触发全局向量相似度搜索；top-k-multiplier=3。
- 通道并行执行（CompletableFuture），结果合并后进入后处理责任链：`DeduplicationPostProcessor`（去重）→ `RerankPostProcessor`（qwen3-rerank 重排，noop 兜底）→ 截断 Top K。
- 底层向量检索由 `RetrieverService` 抽象：`rag.vector.type=pg` 走 `PgRetrieverService`（pgvector，COSINE，dim=1536），`=milvus` 走 `MilvusRetrieverService`。

### 2.6 文档摄取流水线（IngestionEngine）

基于节点图的链式流水线，每个节点实现 `IngestionNode` 接口，可独立替换/插入：

```
FetcherNode（获取：S3/RustFS/本地/HTTP）
  → ParserNode（解析：Tika 支持 PDF/DOC/DOCX/Markdown）
    → ChunkerNode（分块：按标题/长度/段落等策略）
      → EnhancerNode（增强：清洗/元数据补充）
        → EnricherNode（富化：生成 Embedding 向量）
          → IndexerNode（索引：写入 pgvector 或 Milvus + 业务表）
```

流水线定义 / 节点配置 / 任务 / 任务节点执行记录分别落 `t_ingestion_pipeline` / `t_ingestion_pipeline_node` / `t_ingestion_task` / `t_ingestion_task_node`。`KnowledgeDocumentScheduleJob` 支持 `@Scheduled` 定时增量刷新（ETag / Content Hash 判变更 + 分布式锁防重）。

### 2.7 八大设计模式应用

1. **策略模式**：`ChatClient` 多厂商实现、`SearchChannel` 多检索策略、`chunk` 多分块策略——运行时按配置/上下文选择。
2. **责任链模式**：`SearchResultPostProcessor` 后处理链（去重→Rerank→截断）、`StreamChatPipeline` 编排各阶段。
3. **模板方法模式**：`AbstractOpenAIStyleChatClient` 固定 OpenAI 兼容协议骨架，子类只填厂商差异。
4. **状态机模式**：`ModelHealthStore` 三态熔断器 CLOSED/OPEN/HALF_OPEN。
5. **工厂模式**：`IntentTreeFactory` 构建意图树、`Results` 返回体工厂。
6. **注册表模式**：`McpToolRegistry` 工具注册、通道/后处理器按 Spring Bean 注册收集。
7. **装饰器/代理模式**：`TtlExecutors` 包装线程池透传上下文、AOP 切面（幂等/限流/trace）。
8. **门面模式**：`RoutingLLMService` / `RoutingEmbeddingService` / `RoutingRerankService` 对业务屏蔽路由+熔断细节。

### 2.8 完整 RAG 链路（15 步，RAGChatServiceImpl.streamChat / StreamChatPipeline）

1. 前端 SSE 连 `GET /rag/v3/chat`（参数 question / conversationId / deepThinking）。
2. 创建 `SseEmitter`（`rag.default.sse-timeout-ms=300000`），调用 `streamChat`。
3. 生成或复用 conversationId + taskId（taskId 用于停止生成）。
4. `ConversationMemoryService` 加载最近 N 轮对话（history-keep-turns=4）+ 摘要，当前问题写入 **t_message**。
5. `QueryRewriteService.rewriteWithSplit`：规范化补全（结合 t_query_term_mapping 关键词归一化）+ 拆分子问题。
6. `IntentResolver` 把子问题映射到意图树节点（并发分类、过滤低置信 min-intent-score=0.4、限总数）。
7. `IntentGuidanceService.detectAmbiguity`：需澄清则直接 SSE 返回澄清文案并结束。
8. 若全是系统意图（SYSTEM）→ 走纯系统回答（不检索、不调工具）。
9. 否则 `RetrievalEngine.retrieve` → `MultiChannelRetrievalEngine`：意图定向检索 + 全局向量兜底（置信度 <0.6 触发）→ 去重 + Rerank → Top K；MCP 意图用 LLM 按 schema 抽参 → HTTP 调 mcp-server(29099)。
10. KB + MCP 上下文全空 → 退回纯系统回答。
11-12. 合并意图构造 IntentGroup，`RAGPromptService.buildStructuredMessages` 组装结构化 Prompt（主问题 + KB 上下文 + MCP 上下文 + 子问题 + 意图树 + 历史摘要）。
13-14. `LLMService.streamChat`（经 RoutingLLMService 路由 + 熔断）流式输出 → SSE 事件拆为「元数据 stream-meta / 文本增量 message-delta（content 正文 + thinking 思考）/ 完成 completion（messageId + 自动标题）」。
15. 前端递增渲染，结束后可对消息反馈（写 **t_message_feedback**）；全程被 `@RagTraceNode` 记录到 trace 表。

---

## 三、四大核心亮点（面试重点）

### 亮点一：多通道并行检索 + Rerank 精排

**问题**：单一检索策略无法兼顾精确匹配与语义召回，检索质量不稳定。

**方案**：
- 用 `SearchChannel` 接口抽象检索通道，可插拔。
- `IntentDirectedSearchChannel`：基于意图识别结果定向检索（精确度高，min-intent-score=0.4）。
- `VectorGlobalSearchChannel`：全局向量兜底，意图置信度 < 0.6（confidence-threshold）自动触发。
- 通道经 CompletableFuture 并行执行，不叠加延迟。
- 结果经 `DeduplicationPostProcessor` 去重 + `RerankPostProcessor`（qwen3-rerank）精排，最后截断 Top K。

**技术细节**：`MultiChannelRetrievalEngine` 作编排器，管理通道注册、并行执行、结果合并；通道决策基于意图置信度，实现动态检索策略；后处理为责任链，可灵活扩展。底层 `PgRetrieverService`（pgvector，默认）/`MilvusRetrieverService` 由 `rag.vector.type` 切换。

**面试话术**：
> "我把检索策略抽象成 SearchChannel 接口，实现了意图定向和全局向量两个通道，用 CompletableFuture 并行执行，再经去重 + qwen3-rerank 精排责任链返回 Top K。意图置信度低于 0.6 时自动触发全局向量兜底，保证鲁棒性。底层向量库通过配置在 pgvector 和 Milvus 之间切换。"

### 亮点二：模型路由 + 三态熔断器

**问题**：依赖单一厂商有可用性风险。

**方案**：`ModelRoutingExecutor` 按 priority 路由多个候选；`ModelHealthStore` 经典三态熔断器 CLOSED→OPEN→HALF_OPEN；每个模型独立健康状态。

**技术细节（已核对配置/源码）**：失败阈值 **failure-threshold=2**，OPEN 持续 **open-duration-ms=30000**（30 秒）；HALF_OPEN 仅放行一个探测请求（halfOpenInFlight），成功转 CLOSED 并清零、失败退回 OPEN；候选与优先级在 `ai.chat.candidates` 配置，支持百炼/SiliconFlow/AIHubMix/Ollama 四厂商。

**面试话术**：
> "ModelRoutingExecutor 按优先级依次尝试模型，每个模型挂一个 ModelHealthStore 三态熔断器。连续失败 2 次熔断 30 秒，到期半开放行一次探测，成功恢复、失败继续熔断，故障厂商被自动隔离并切到下一候选。"

### 亮点三：分布式队列限流（Redis 信号量 + ZSET + Lua + Pub/Sub）

**问题**：大模型调用成本高、并发需可控，又要保证排队体验。

**方案**：`@ChatRateLimit` + 队列控制器。Redis 信号量控全局并发（max-concurrent=10、lease-seconds=30 租约防泄漏）；ZSET 按时间戳排队（公平、max-wait-seconds=15 超时放弃）；Lua 脚本保证占名额/出队原子；Pub/Sub 通知 + poll-interval-ms=200 轮询兜底；配合 `@IdempotentSubmit` 防重复提交。

**面试话术**：
> "用 Redis 信号量控全局并发，ZSET 按时间戳排队保证公平，Lua 保证原子，名额释放用 Pub/Sub 通知 + 轮询兜底，租约防止持有者宕机泄漏名额。"

### 亮点四：会话记忆管理（滑动窗口 + 自动摘要压缩）

**问题**：多轮对话历史累积超上下文窗口、增 Token 成本。

**方案**：`ConversationMemoryService` + `ConversationMemoryStore`（`JdbcConversationMemoryStore`）。滑动窗口保留最近 **4 轮**（history-keep-turns=4）；超过 **5 轮**（summary-start-turns=5）触发 LLM 自动摘要（summary-enabled=true，summary-max-chars=200），摘要作为系统消息注入上下文。摘要缓存到 `t_conversation_summary` 避免重复生成；会话标题自动生成（title-max-length=30）。

**面试话术**：
> "记忆采用滑动窗口 + 自动摘要：保留最近 4 轮原文，超过 5 轮就用 LLM 把更早的对话压成 200 字以内的摘要注入上下文，摘要落 t_conversation_summary 复用。既保留关键历史，又控住 Token。"

---

## 四、技术栈详解

### 4.1 后端技术栈（版本以 pom.xml 为准）

| 技术 | 版本 | 用途 |
|------|------|------|
| Java | 17 | 开发语言 |
| Spring Boot | 3.5.7 | 应用框架 |
| MyBatis-Plus | 3.5.14 | ORM（含 jsqlparser） |
| Sa-Token | 1.43.0 | 认证鉴权 |
| PostgreSQL + pgvector | - | 业务库 + 默认向量存储（dim 1536, COSINE） |
| HikariCP | - | 连接池 |
| Redisson | 4.0.0 | 缓存/分布式锁/限流/Pub/Sub |
| RocketMQ Spring Boot Starter | 2.3.5 | 异步消息 |
| Milvus SDK | 2.6.6 | 可选向量数据库 |
| MCP SDK | 1.1.2 | Model Context Protocol |
| Apache Tika | 3.2.3 | 文档解析（PDF/DOC/DOCX/MD） |
| AWS S3 SDK | 2.40.2 | RustFS（S3 兼容）对象存储 |
| Hutool | 5.8.37 | 工具库 |
| OkHttp | 4.12.0 | HTTP 客户端（调模型/MCP） |
| transmittable-thread-local | 2.14.5 | 线程池上下文透传 |

> 注意：**业务库是 PostgreSQL 不是 MySQL**；向量存储默认 pgvector，Milvus 为可选；对象存储是 RustFS；**未使用 Spring AI**，AI 能力由自研 infra-ai 抽象层提供；供应商**不含 DuoJie/多杰**。

### 4.2 前端技术栈

| 技术 | 版本 | 用途 |
|------|------|------|
| React | 18 | UI 框架 |
| TypeScript | - | 类型安全 |
| Vite | 5 | 构建工具（dev 端口 25173，代理 /api → :29090） |
| React Router | 6 | 路由管理 |
| Radix UI | - | 无样式可访问组件（非 Ant Design） |
| TailwindCSS | - | 原子化 CSS |
| Zustand | - | 状态管理 |
| Axios | - | HTTP 客户端 |
| Recharts | - | 图表（Dashboard） |
| react-markdown / react-virtuoso / sonner | - | Markdown 渲染 / 虚拟列表 / Toast |

### 4.3 基础设施亮点

1. **TTL 上下文传播**：`TtlExecutors` 包装所有线程池，跨线程透传 `UserContext` 与 `RagTraceContext`。
2. **全链路追踪**：`@RagTraceRoot`/`@RagTraceNode` + AOP 自动记录节点耗时/输入输出/错误（rag.trace.enabled=true，max-error-length=1000），落 `t_rag_trace_run`/`t_rag_trace_node`。
3. **幂等提交**：`@IdempotentSubmit` + AOP + Redis Lua。
4. **雪花算法分布式 ID**：Snowflake + `lua/snowflake_init.lua` 分配 workerId。
5. **统一异常体系**：`ClientException` / `ServiceException` / `RemoteException` 三层分类 + 错误码枚举。
6. **统一返回体**：`Result<T>` + `Results` 工厂 + `GlobalExceptionHandler`。

---

## 五、前端页面与功能

### 5.1 页面清单（src/pages）

**用户端：**
- `LoginPage`：登录页。
- `ChatPage`：主对话页，SSE 流式输出、深度思考、Markdown 渲染、代码高亮、消息反馈。

**管理端（admin/*）：**
- dashboard：数据看板（对话量/用户/知识库统计，Recharts）。
- knowledge：知识库列表 / 文档 / chunk 三级管理。
- intent-tree：意图树（树/列表/编辑）。
- ingestion：摄取流水线与任务。
- traces：链路追踪列表 / 详情。
- settings：系统设置（展示 ai.* 配置）。
- users：用户管理。
- sample-questions：示例问题管理。
- query-term-mapping：关键词归一化映射管理。

### 5.2 核心组件、服务与状态（src/components / services / stores / hooks）

- **components**：ui（Radix 封装）、chat（`ChatInput` / `MessageList` / `MessageItem` / `MarkdownRenderer` / `FeedbackButtons` / `WelcomeScreen` / `ThinkingIndicator`）、admin、layout。
- **services**：`api.ts`（Axios 实例 + 拦截器，按 code 字段判定成功/失败，401 跳登录）、authService、chatService、sessionService、knowledgeService、dashboardService、ingestionService、ragTraceService、intentTreeService、userService、settingsService、queryTermMappingService、sampleQuestionService。
- **stores**：authStore、chatStore、themeStore（Zustand）。
- **hooks**：useAuth、useChat、`useStreamResponse`（SSE 流式处理）。
- **router.tsx**：`RequireAuth` / `RequireAdmin` / `RedirectIfAuth` 路由守卫。

---

## 六、数据库表设计（21 张，resources/database/schema_pg.sql）

> 数据库为 PostgreSQL（pgvector），初始化用 `schema_pg.sql` + `init_data_pg.sql`，pgvector 扩展由 `resources/docker/postgres/init/01-init-pgvector.sql` 创建。旧 MySQL 脚本已归档到 `resources/database/backups/`，不再用于主流程。

**会话域（5 张）：**

| 表名 | 说明 | 核心字段 |
|------|------|----------|
| t_conversation | 会话列表 | conversation_id, user_id, title, last_time |
| t_conversation_summary | 会话摘要 | conversation_id, last_message_id, content |
| **t_message** | 消息记录（非 t_conversation_message） | conversation_id, user_id, role(system/user/assistant), content |
| **t_message_feedback** | 消息反馈（点赞/点踩） | message_id, user_id, vote(1点赞/-1点踩), reason |
| t_sample_question | 示例问题 | title, description, question |

**知识库域（6 张）：**

| 表名 | 说明 | 核心字段 |
|------|------|----------|
| t_knowledge_base | 知识库 | name, embedding_model, collection_name |
| t_knowledge_document | 文档 | kb_id, doc_name, file_url, file_type, status, chunk_strategy, pipeline_id |
| t_knowledge_chunk | 文档分块 | kb_id, doc_id, chunk_index, content, content_hash, char_count |
| **t_knowledge_vector** | 分块向量存储（pgvector） | chunk_id, kb_id, embedding(vector 1536), collection_name |
| t_knowledge_document_chunk_log | 分块日志 | doc_id, status, extract/chunk/embedding_duration, chunk_count |
| t_knowledge_document_schedule | 定时刷新任务 | doc_id, cron_expr, next_run_time, last_etag, last_content_hash |

> （定时刷新执行记录见下方摄取/调度域。）

**调度执行（1 张）：**

| 表名 | 说明 | 核心字段 |
|------|------|----------|
| t_knowledge_document_schedule_exec | 定时刷新执行记录 | schedule_id, doc_id, status, content_hash, etag |

**摄取流水线域（4 张）：**

| 表名 | 说明 | 核心字段 |
|------|------|----------|
| t_ingestion_pipeline | 流水线定义 | name, description |
| t_ingestion_pipeline_node | 流水线节点配置 | pipeline_id, node_id, node_type, next_node_id, settings_json |
| t_ingestion_task | 摄取任务记录 | pipeline_id, source_type, status, chunk_count, logs_json |
| t_ingestion_task_node | 任务节点执行记录 | task_id, node_id, node_type, status, duration_ms, output_json |

**意图与检索域（2 张）：**

| 表名 | 说明 | 核心字段 |
|------|------|----------|
| **t_intent_node** | 意图树节点（非 t_intent_tree_node） | intent_code, name, level(0/1/2), parent_code, collection_name, mcp_tool_id, kind(RAG/SYSTEM) |
| t_query_term_mapping | 关键词归一化映射 | source_term, target_term, match_type(精确/前缀/正则/整词), priority |

**链路追踪域（2 张）：**

| 表名 | 说明 | 核心字段 |
|------|------|----------|
| t_rag_trace_run | Trace 运行记录 | trace_id, conversation_id, user_id, status, duration_ms |
| t_rag_trace_node | Trace 节点记录 | trace_id, node_id, parent_node_id, depth, node_type, duration_ms |

**用户域（1 张）：**

| 表名 | 说明 | 核心字段 |
|------|------|----------|
| t_user | 系统用户 | username, password, role(admin/user), avatar |

---

## 七、核心 API 接口清单

> 所有接口前缀为 context-path `/api/ragent`。

### 对话相关
| 接口 | 方法 | 说明 |
|------|------|------|
| /rag/v3/chat | GET(SSE) | 发起 RAG 对话，流式返回（question / conversationId / deepThinking） |
| 停止任务接口 | - | 按 taskId 停止生成（StreamCancellationHandle） |
| /conversations | GET | 会话列表 |
| /conversations/{id} | DELETE/PUT | 删除/重命名会话 |
| /conversations/{id}/messages | GET | 会话消息列表 |
| /conversations/messages/{id}/feedback | POST | 消息反馈（点赞/点踩） |
| /sample-questions | GET | 示例问题列表 |

### 知识库管理
| 接口 | 方法 | 说明 |
|------|------|------|
| /knowledge-base | GET/POST | 知识库列表/创建 |
| /knowledge-base/{kbId}/docs/upload | POST | 上传文档 |
| /knowledge-base/docs/{docId}/chunk | GET | 文档分块列表 |
| /knowledge-base/docs/{docId}/chunk-logs | GET | 分块日志 |

### 摄取流水线
| 接口 | 方法 | 说明 |
|------|------|------|
| /ingestion/pipelines | GET/POST | 流水线列表/创建 |
| /ingestion/tasks | GET/POST | 任务列表/创建 |
| /ingestion/tasks/{id} | GET | 任务详情（含节点执行记录） |

### 意图树管理
| 接口 | 方法 | 说明 |
|------|------|------|
| /intent-tree | GET | 完整意图树 |
| /intent-tree/nodes | POST | 创建意图节点 |
| /intent-tree/nodes/{id} | PUT/DELETE | 更新/删除意图节点 |

### 系统管理
| 接口 | 方法 | 说明 |
|------|------|------|
| /auth/login | POST | 登录（admin/admin） |
| /auth/logout | POST | 登出 |
| /users | GET/POST | 用户列表/创建 |
| /rag/settings | GET | 获取系统/模型配置（映射 ai.*） |
| /admin/dashboard/overview | GET | 看板概览 |
| /admin/dashboard/performance | GET | 性能指标 |
| /admin/dashboard/trends | GET | 趋势数据 |
| /traces | GET | 链路追踪列表 |
| /traces/{id} | GET | 链路追踪详情 |

---

## 八、本地运行（规范步骤）

1. 准备 JDK 17、Node 18+、Docker。
2. 用 `resources/docker/` 的 compose 启动中间件：
   - `postgres-pgvector-stack.compose.yaml`（PostgreSQL + pgvector，含 `init/01-init-pgvector.sql`）
   - Redis、`rocketmq-stack-5.2.0.compose.yaml`（amd 版另有文件）
   - 可选 `milvus-stack-2.6.6.compose.yaml`（lightweight/ 下有 2.5.8、2.6.6 轻量版）
3. 初始化库：执行 `resources/database/schema_pg.sql` + `init_data_pg.sql`（pgvector 扩展由 `01-init-pgvector.sql` 创建；升级用 `upgrade_v1.0_to_v1.1.sql`、`upgrade_v1.1_to_v1.2.sql`）。
4. 配置 `bootstrap/src/main/resources/application.yaml`（datasource=postgres、redis、rocketmq、milvus、rustfs；AI key 用环境变量 `BAILIAN_API_KEY`/`SILICONFLOW_API_KEY`/`AIHUBMIX_API_KEY`）。
5. 启动后端：`./mvnw -pl bootstrap -am spring-boot:run`（29090）；`./mvnw -pl mcp-server -am spring-boot:run`（29099）。
6. 前端：`cd frontend && npm install && npm run dev`（25173）→ 访问 `http://localhost:25173`。
7. 登录 `admin / admin`。

---

## 九、面试竞争力分析

### 9.1 项目亮点总结
1. **技术前沿性**：RAG + Agent（MCP 工具调用）是当前 AI 落地主流架构。
2. **工程完整性**：从摄取流水线、向量化、多通道检索、对话到管理后台 + 链路追踪，覆盖完整产品链路。
3. **架构设计深度**：五大扩展点、三态熔断、队列限流、责任链后处理、TTL 上下文传播。
4. **技术栈丰富度**：Java + Spring Boot 3 + React 18 + PostgreSQL/pgvector + Redis + RocketMQ + 多厂商大模型。

### 9.2 面试常见问题预演

**Q1：介绍一下你的项目？**
> "Ragent 是企业级 Agentic RAG 平台，基于 Spring Boot 3.5.7 + React 18 + PostgreSQL(pgvector)。把企业私有知识库与大模型结合做精准问答，并通过 MCP 调用内部系统。我主要负责多通道检索引擎、模型路由熔断、分布式队列限流和会话记忆四个核心模块。"

**Q2：RAG 完整流程？**
> "加载会话记忆 → 查询改写与拆分 → 三级意图识别 → 歧义引导 →（意图定向 + 全局向量兜底）多通道并行检索 → 去重 + Rerank → 组装结构化 Prompt → 经路由熔断的 LLM 流式输出 → SSE 三类事件返回，结束可反馈。"

**Q3：为什么默认用 pgvector 而不是单独的向量库？**
> "pgvector 让向量与业务数据同库，事务一致、运维简单、适合中小规模知识库；数据量大或需更强 ANN 时通过 `rag.vector.type=milvus` 一键切到 Milvus，底层都由 RetrieverService 抽象，业务无感知。"

**Q4：熔断器怎么实现的？**
> "ModelHealthStore 三态机：CLOSED 连续失败 2 次转 OPEN 拒绝 30 秒，到期 HALF_OPEN 放行一次探测，成功回 CLOSED、失败退 OPEN。每个模型独立状态，ModelRoutingExecutor 跳过 OPEN 候选路由到下一个。"

**Q5：分布式限流怎么做的？**
> "Redis 信号量控全局并发，ZSET 按时间戳公平排队，Lua 保证原子，名额释放 Pub/Sub 通知 + 轮询兜底，租约防泄漏。"

---

## 十、项目不足与改进建议

### 10.1 现有不足
1. 单元/集成测试覆盖偏少（已引入 Mockito 5.20.0，但核心链路测试不足）。
2. 未集成 Swagger/SpringDoc，缺 API 文档。
3. 前端缺国际化（仅中文）。
4. 缺 Prometheus + Grafana 监控告警。
5. MCP 工具仅销售/工单/天气三个示例。

### 10.2 改进建议
1. 补 `RAGChatServiceImpl`、`MultiChannelRetrievalEngine`、`ModelRoutingExecutor`、队列限流器的单测。
2. 集成 SpringDoc OpenAPI。
3. 完善 docker compose 一键编排（PG/Redis/RocketMQ/Milvus/RustFS/后端/前端）。
4. 扩展更多 MCP 工具，展示 Agent 扩展性。
5. 增加混合检索（向量 + BM25 关键词通道，直接复用 SearchChannel 扩展点）。

---

## 十一、核心源码文件索引

| 文件/类 | 模块 | 说明 |
|------|------|------|
| RagentApplication | bootstrap | 启动类（29090） |
| RAGChatServiceImpl / StreamChatPipeline | bootstrap | 核心 RAG 对话编排 |
| MultiChannelRetrievalEngine / RetrievalEngine | bootstrap | 多通道检索引擎 |
| VectorGlobalSearchChannel / IntentDirectedSearchChannel | bootstrap | 检索通道（SearchChannel） |
| DeduplicationPostProcessor / RerankPostProcessor | bootstrap | 后处理责任链 |
| PgRetrieverService / MilvusRetrieverService | bootstrap | 向量检索（pg 默认/milvus 可选） |
| IntentResolver / DefaultIntentClassifier / IntentTreeFactory | bootstrap | 三级意图识别 |
| QueryRewriteService | bootstrap | 查询改写与拆分 |
| ConversationMemoryService / JdbcConversationMemoryStore | bootstrap | 会话记忆 |
| IntentGuidanceService | bootstrap | 歧义引导 |
| RAGPromptService / PromptTemplateLoader | bootstrap | Prompt 组装 |
| McpToolExecutor / McpToolRegistry / McpClientToolExecutor | bootstrap | MCP 客户端 |
| IngestionEngine / IngestionNode(6 节点) | bootstrap | 文档摄取流水线 |
| ChatClient / AbstractOpenAIStyleChatClient / 四厂商实现 | infra-ai | 大模型客户端 |
| RoutingLLMService / RoutingEmbeddingService / RoutingRerankService | infra-ai | 路由门面 |
| ModelSelector / ModelRoutingExecutor / ModelHealthStore / ModelTarget | infra-ai | 模型路由 + 三态熔断 |
| AIModelProperties | infra-ai | ai.* 配置映射 |
| McpServerApplication / Sales·Ticket·WeatherMcpExecutor | mcp-server | MCP 工具服务（29099） |
| application.yaml | bootstrap | 核心配置 |
| schema_pg.sql / init_data_pg.sql / 01-init-pgvector.sql | resources | 数据库初始化 |

---

## 十二、容易被忽略的隐藏功能

### 12.1 关键词归一化映射（t_query_term_mapping）
用户口语化表达与系统标准术语常不一致（如"报销"vs"费用报销申请"）。该表支持精确/前缀/正则/整词四种匹配，按 priority 排序（值越小越优先），在查询改写阶段自动替换，提升精确命中率。

### 12.2 文档定时刷新（Schedule）
`t_knowledge_document_schedule` 支持 Cron 定时拉取；增量更新靠 ETag / Content Hash 判变更；`KnowledgeDocumentScheduleJob` 用分布式锁（lock_owner + lock_until）防多节点重复；执行记录落 `t_knowledge_document_schedule_exec`。配置见 `rag.knowledge.schedule`。

### 12.3 意图树节点双模式（kind 字段）
`t_intent_node.kind`：RAG 类关联 collection_name 走向量检索；SYSTEM 类关联 mcp_tool_id 走 MCP 工具调用。同一棵意图树同时处理"查知识"与"调系统"，是 RAG + Agent 融合的关键。

### 12.4 SSE 流式输出三种事件
- `stream-meta`：流开始，携带 conversationId / taskId。
- `message-delta`：内容增量，type 区分 content（正文）与 thinking（思考）。
- `completion`：流结束，携带 messageId 与自动生成的会话标题。

---

以上即 Ragent 项目的完整深度解析，所有事实均与主干代码、application.yaml、schema_pg.sql、pom.xml 对齐，可直接用于面试准备。
