 # Ragent 项目介绍与业务流程（超详细）

## 1. 项目整体概览

### 1.1 项目定位

Ragent（RAG + Agent）是一个面向企业内部知识问答场景的 Agentic RAG 智能体平台。后端基于 Java 17 和 Spring Boot 3.5.7，采用多模块 Maven 架构；前端基于 React 18；业务数据库使用 **PostgreSQL（启用 pgvector 扩展）**，向量检索默认走 **pgvector**，也可切换到 **Milvus 2.6**；配套对象存储 RustFS、缓存与分布式锁 Redis、消息队列 RocketMQ。

它的设计目标不是简单的演示 Demo，而是覆盖以下完整能力：

- 文档解析、切分、向量化、入库和定时刷新
- 多知识库管理、文档启停、分组和统计
- 多通道检索（意图定向检索加全局兜底检索）和 Rerank 重排
- 对话记忆、摘要、历史窗口控制
- 意图树、关键词归一化、Ambiguity 引导
- MCP 工具调用和参数抽取，将外部系统能力接入对话
- RAG Trace 链路追踪和管理仪表盘
- 模型路由、三态熔断、分布式队列限流
- 完整的用户和权限体系

非常适合作为：

- 学习企业级 RAG 系统的参考实现
- 面试时展示 AI 工程化能力的项目
- 二次开发定制企业内部智能助手的基础骨架

### 1.2 技术栈总览

后端技术栈：

- Java 17
- Spring Boot 3.5.7
- MyBatis-Plus 3.5.14（含 jsqlparser）、HikariCP
- **PostgreSQL（pgvector 扩展）业务数据库**，向量默认存储于 pgvector
- Redis（Redisson 4.0.0）缓存与分布式锁
- RocketMQ Spring Boot Starter 2.3.5 消息队列
- Milvus SDK 2.6.6（可选向量存储）
- RustFS（S3 兼容对象存储，AWS S3 SDK 2.40.2）
- Apache Tika 3.2.3 文档解析
- Sa-Token 1.43.0 认证鉴权
- Hutool 5.8.37、OkHttp 4.12.0、MCP SDK 1.1.2、transmittable-thread-local 2.14.5
- 自研 infra-ai 模块封装多个模型提供方

前端技术栈：

- React 18 + TypeScript + Vite 5
- React Router 6
- Zustand 状态管理
- Radix UI 组件 + TailwindCSS
- Axios 请求封装
- Recharts、react-markdown、react-virtuoso、sonner 等

基础设施（resources/docker/）：

- `postgres-pgvector-stack.compose.yaml`：PostgreSQL + pgvector（含 `init/01-init-pgvector.sql` 自动创建扩展）
- `milvus-stack-2.6.6.compose.yaml`：可选 Milvus（`lightweight/` 下有 2.5.8、2.6.6 轻量版）
- `rocketmq-stack-5.2.0.compose.yaml`（另有 `rocketmq-stack-amd-5.2.0.compose.yaml`）
- SQL 脚本初始化全部数据表以及演示数据

---

## 2. 模块和目录结构

### 2.1 Maven 模块

根 pom 中定义了四个子模块：

- `framework`
  提供通用基础设施，包括统一返回体、异常与错误码体系、Redis 与 Redisson 封装、分布式 ID、幂等控制、Web 层封装和 RAG Trace 能力。

- `infra-ai`
  抽象 LLM 聊天、Embedding、Rerank 能力，支持多模型、多厂商路由和熔断。

- `bootstrap`
  主业务服务（端口 29090，context-path `/api/ragent`），包含 RAG 对话、知识库管理、文档摄取流水线、用户与后台管理等全部业务逻辑。

- `mcp-server`
  独立的 MCP Tool Server（端口 **29099**），用于演示如何通过 MCP 将企业内部系统能力接入 RAG 对话。

项目根目录下还有：

- `frontend` 前端 React 工程
- `resources/database` 数据库脚本（PostgreSQL）
- `resources/docker` PostgreSQL/pgvector、Milvus、RocketMQ 的 compose 文件
- `docs` 文档目录（本文件即 docs/README.md）

### 2.2 bootstrap 模块包划分

`bootstrap/src/main/java/com/nageoffer/ai/ragent` 下按业务域进行划分（admin / user / rag / knowledge / ingestion / core）：

- `admin`
  管理后台域。`DashboardController` 和 `DashboardService` 提供 overview、performance、trends 等统计接口，用于仪表盘。

- `user`
  用户与认证授权。
  - `config` 中配置 Sa-Token 与用户上下文注入
  - `AuthController` 处理登录和登出
  - `UserController` 负责用户管理、当前登录用户信息和修改密码
  - `UserService` 和 `AuthService` 实现具体逻辑
  - 数据落地在 `t_user` 等表中

- `rag`
  RAG 对话核心域。主要子包：
  - `controller`
    - `RAGChatController`：SSE 流式对话接口 `GET /rag/v3/chat` + 停止任务
    - `ConversationController`：会话列表与消息列表
    - `SampleQuestionController`：样例问题
    - `RAGSettingsController`：系统 RAG 配置查询
    - `RagTraceController`：RAG Trace 查询接口
    - `IntentTreeController`：意图树查询与维护
  - `service`
    - `RAGChatService` 及其实现 `RAGChatServiceImpl.streamChat`（核心入口）
    - `StreamChatPipeline`：编排 memory→rewrite→intent→guidance→retrieve→prompt→LLM
    - `ConversationService`、`ConversationMessageService`、`ConversationMemoryService`
    - `RagTraceQueryService`、`SampleQuestionService` 等
  - `core`
    - `memory`：`ConversationMemoryService`、`ConversationMemoryStore`（`JdbcConversationMemoryStore`），对话记忆与摘要
    - `rewrite`：`QueryRewriteService.rewriteWithSplit`，查询重写与多问题拆分
    - `intent`：`IntentClassifier`/`DefaultIntentClassifier`、`IntentResolver`、`IntentTreeFactory`，意图树和意图分类
    - `retrieve`：`RetrievalEngine`、`MultiChannelRetrievalEngine`、`SearchChannel`（`VectorGlobalSearchChannel`、`IntentDirectedSearchChannel`）、`SearchResultPostProcessor`（`DeduplicationPostProcessor`、`RerankPostProcessor`）、`RetrieverService`（`PgRetrieverService`、`MilvusRetrieverService`）
    - `guidance`：`IntentGuidanceService.detectAmbiguity`，模糊问题检测与澄清
    - `prompt`：`RAGPromptService.buildStructuredMessages`、`PromptTemplateLoader`
    - `mcp`：`McpToolExecutor`、`McpToolRegistry`、`McpClientToolExecutor`
  - `config`
    对应 `application.yaml` 中的 rag 和 ai 配置，如 Rate Limit、Memory、向量类型、搜索渠道等
  - `dao`
    会话、消息、样例问题、Trace 等领域的 Mapper 和实体
  - `aop`
    `@ChatRateLimit`、请求队列控制和 Trace 切面

- `knowledge`
  知识库管理域。
  - 控制器：`KnowledgeBaseController`、`KnowledgeDocumentController`、`KnowledgeChunkController`
  - 任务调度：`KnowledgeDocumentScheduleJob`（`@Scheduled` 定时刷新文档）
  - 服务：`KnowledgeBaseService`、`KnowledgeDocumentService` 等
  - 对应表包括知识库、文档、chunk、schedule 及其执行记录。

- `ingestion`
  文档摄取流水线域。
  - `IngestionPipelineController` 管理 Pipeline 模板
  - `IngestionTaskController` 创建和查询摄取任务
  - `domain` 描述 Pipeline、节点配置、运行上下文和结果
  - `engine` 中的 `IngestionEngine` 负责按流水线执行各个节点
  - `node` 包含具体节点实现：`FetcherNode`、`ParserNode`、`ChunkerNode`、`EnhancerNode`、`EnricherNode`、`IndexerNode`
  - `prompt` 中放置摄取相关的 LLM Prompt，例如结构化抽取说明
  - 服务层负责把控制器请求转换为具体流水线执行

- `core`
  文档解析和切分的通用核心模块。
  - `parser` 使用 Apache Tika 解析 PDF、Office、Markdown 等为纯文本
  - `chunk` 定义多种切分策略，例如按标题、按长度、按段落等

- 其他
  - `RagentApplication` Spring Boot 启动类
  - `resources/file` 中预置了多份 HR、IT、保险等中文文档，作为默认知识库内容
  - `resources/prompt` 存放对话和摄取所用的模板脚本

### 2.3 framework 模块

`framework` 提供项目级基础设施：

- `convention/Result` 和 `web/Results`
  定义统一返回结构和快速构造方法，所有接口返回一个 code 加 message 加 data 的包装对象。

- `exception` 与 `errorcode`
  约定三级业务异常层级和错误码枚举，方便上层统一处理。

- `context`
  `UserContext`、`LoginUser` 负责在业务代码中方便获取当前登录用户信息。

- `idempotent`
  基于 RedisTemplate 和 Lua 脚本实现的接口幂等控制，配合注解 `@IdempotentSubmit` 使用。

- `trace`
  定义 `RagTraceContext`、`@RagTraceNode`、`@RagTraceRoot` 注解，配合数据库表记录每一次 RAG 调用链路。

- `web`
  全局异常处理、统一 SSE 返回封装（`SseEmitterSender`）等。

- `distributedid` 以及 `framework/src/main/resources/lua/snowflake_init.lua`
  提供分布式雪花 ID 生成功能。

- `cache`、`mq` 等公共能力。

> 所有线程池都用 `TtlExecutors` 包装（transmittable-thread-local），跨线程透传用户上下文与 Trace 上下文。

### 2.4 infra-ai 模块

主要是对 AI 厂商接口的抽象与封装：

- `chat`
  `ChatClient` 接口 + `AbstractOpenAIStyleChatClient` 基类，以及具体实现 `BaiLianChatClient`、`SiliconFlowChatClient`、`AIHubMixChatClient`、`OllamaChatClient`；`LLMService`/`RoutingLLMService` 负责根据配置进行模型路由和降级；`StreamCallback`、`StreamCancellationHandle` 支持流式与取消。

- `embedding`
  `EmbeddingClient` + `RoutingEmbeddingService`，封装 Embedding 生成和多模型候选选择逻辑。

- `rerank`
  `RoutingRerankService`，包含重排模型客户端以及路由服务。

- `model`
  `ModelSelector`、`ModelRoutingExecutor`、`ModelHealthStore`、`ModelTarget` 等类，用于根据健康检测、能力标签、优先级来选择后端模型。

- `config/AIModelProperties`
  映射 `application.yaml` 中 `ai.*` 相关配置，并暴露给 `RAGSettingsController` 供前端查看。

### 2.5 mcp-server 模块

- `McpServerApplication` 为独立 Spring Boot 启动类，监听端口 **29099**。
- `core` 包中定义 MCP 工具的元模型、请求和响应格式、执行器接口等。
- `endpoint` 包用来接收 JSON-RPC 请求，并路由到合适的工具执行器。
- `executor` 中包含若干模拟工具执行器：`SalesMcpExecutor`、`TicketMcpExecutor`、`WeatherMcpExecutor`。
  - 以 `SalesMcpExecutor` 为例，支持 region、period、product、salesPerson、queryType 等参数；
  - 可以返回 summary、ranking、detail、trend 等不同形式的结果；
  - 内部用随机数据模拟业绩记录，便于本地演示。

bootstrap 模块通过 `rag.core.mcp` 包中的客户端和执行器，作为 MCP 客户端调用这个服务。

---

## 3. 数据模型概览

数据库结构在 `resources/database/schema_pg.sql` 中定义（PostgreSQL，启用 pgvector），共 **21 张表**。下面按业务域说明。

> 旧的 MySQL 脚本（`schema_table.sql`、`init_data.sql`）已归档到 `resources/database/backups/`，不再作为主流程使用。

### 3.1 会话与消息

- `t_conversation`
  存储每个用户的会话列表，包括会话标识、标题、最后消息时间等。

- `t_conversation_summary`
  存储每个会话的压缩摘要，支持长会话场景下的历史摘要替代。

- `t_message`
  存储每一条对话消息，记录角色、内容、是否为思考内容、所属会话和顺序等。
  （注意：消息表名为 `t_message`，并非 `t_conversation_message`。）

- `t_message_feedback`
  存储消息级别的用户反馈（点赞/点踩等），与前端 `FeedbackButtons` 对应。

对应服务：

- `ConversationService` 负责会话增删改查和重命名。
- `ConversationMessageService` 与 `ConversationMemoryService` 负责加载最近 N 轮对话、保存新消息以及触发摘要逻辑。

### 3.2 知识库和文档

- `t_knowledge_base`
  知识库主表，包含名称、描述、默认向量 collection 名称等。

- `t_knowledge_document`
  知识库中的单个文档，包括存储位置（RustFS 对象 key）、文件名、类型、大小、启用状态等。

- `t_knowledge_chunk`
  文档切分后的片段，记录 chunk 内容、所属文档、页码、对应向量 ID 等信息。

- `t_knowledge_vector`
  当向量存储类型为 pgvector 时，存放 chunk 的向量数据（pgvector 列，维度 1536，余弦相似度）。

- `t_knowledge_document_chunk_log`
  用于记录每次对某个文档执行切分和索引的过程及结果。

- `t_knowledge_document_schedule` 与 `t_knowledge_document_schedule_exec`
  描述文档定时刷新的配置和执行历史，用来支持周期性重新拉取和索引外部文档。

### 3.3 摄取流水线

- `t_ingestion_pipeline`
  定义一条摄取流水线，包括名称、描述、创建人等。

- `t_ingestion_pipeline_node`
  定义流水线中的节点顺序、类型、配置设置和条件配置。

- `t_ingestion_task`
  一次具体的摄取任务，表示某个 pipeline 对某个文档或某个数据源的一次执行。

- `t_ingestion_task_node`
  记录该任务中每个节点的执行状态、开始结束时间、错误信息等。

### 3.4 意图与关键词映射

- `t_intent_node`
  存储完整意图树，每个节点可以对应某个知识库、某个 MCP 工具或者系统意图。
  （注意：意图表名为 `t_intent_node`，并非 `t_intent_tree_node`。）

- `t_query_term_mapping`
  存储关键词映射规则，用于把口语化表达映射为标准术语，支持多种匹配方式和优先级。

### 3.5 RAG Trace 与样例问题

- `t_rag_trace_run` 和 `t_rag_trace_node`
  记录一次 RAG 调用及其内部各步骤（重写、意图识别、检索、MCP 调用、LLM 调用等）的执行信息。

- `t_sample_question`
  存储首页展示的示例问题，便于引导用户提问。

### 3.6 用户

- `t_user`
  用户表，初始数据中已插入管理员账号 `admin / admin`（admin 角色）。

> 完整 21 张表清单：`t_user`、`t_conversation`、`t_conversation_summary`、`t_message`、`t_message_feedback`、`t_sample_question`、`t_knowledge_base`、`t_knowledge_document`、`t_knowledge_chunk`、`t_knowledge_document_chunk_log`、`t_knowledge_document_schedule`、`t_knowledge_document_schedule_exec`、`t_intent_node`、`t_query_term_mapping`、`t_rag_trace_run`、`t_rag_trace_node`、`t_ingestion_pipeline`、`t_ingestion_pipeline_node`、`t_ingestion_task`、`t_ingestion_task_node`、`t_knowledge_vector`。

---

## 4. 关键配置项一览

以下数值取自 `bootstrap/src/main/resources/application.yaml`。

### 4.1 AI 供应商（ai.providers）

内置四个供应商：

- `ollama`（本地 `http://localhost:11434`）
- `bailian`（阿里百炼 dashscope，`https://dashscope.aliyuncs.com`，`BAILIAN_API_KEY`）
- `aihubmix`（`https://aihubmix.com`，`AIHUBMIX_API_KEY`）
- `siliconflow`（`https://api.siliconflow.cn`，`SILICONFLOW_API_KEY`）

### 4.2 对话模型候选（ai.chat.candidates，按 priority）

| id | provider | model | 备注 |
| --- | --- | --- | --- |
| qwen-plus | bailian | qwen-plus-latest | priority 1 |
| qwen3-local | ollama | qwen3:8b-fp16 | priority 2 |
| qwen3-max | bailian | qwen3-max | supports-thinking，priority 3 |
| glm-4.7 | siliconflow | Pro/zai-org/GLM-4.7 | supports-thinking，priority 4 |
| gpt-5.4 | aihubmix | gpt-5.4 | priority 5 |

`default-model` 与 `deep-thinking-model` 均为 `qwen3-max`。

### 4.3 Embedding 候选（ai.embedding.candidates）

| id | provider | model | priority |
| --- | --- | --- | --- |
| qwen-emb-8b | siliconflow | Qwen/Qwen3-Embedding-8B | 1（默认） |
| qwen-emb-local | ollama | qwen3-embedding:8b-fp16 | 2 |
| text-embedding-3-large | aihubmix | text-embedding-3-large | 3 |

dimension 统一为 1536。

### 4.4 Rerank 候选（ai.rerank.candidates）

| id | provider | model | priority |
| --- | --- | --- | --- |
| qwen3-rerank | bailian | qwen3-rerank | 1（默认） |
| rerank-noop | noop | noop | 100（兜底，不重排） |

### 4.5 模型路由与熔断（ai.selection）

三态熔断器 `CLOSED → OPEN → HALF_OPEN`，`failure-threshold=2`，`open-duration-ms=30000`。核心类：`ModelRoutingExecutor`、`ModelHealthStore`、`ModelSelector`、`ModelTarget`。

### 4.6 限流（rag.rate-limit.global）

队列式分布式限流（Redis 信号量 + ZSET 排队 + Pub/Sub 通知 + Lua 原子判断）：`max-concurrent=10`、`max-wait-seconds=15`、`lease-seconds=30`、`poll-interval-ms=200`。

### 4.7 会话记忆（rag.memory）

`history-keep-turns=4`、`summary-start-turns=5`、`summary-enabled=true`、`summary-max-chars=200`、`title-max-length=30`。

### 4.8 检索通道（rag.search.channels）

- `vector-global`：`confidence-threshold=0.6`，`top-k-multiplier=3`
- `intent-directed`：`min-intent-score=0.4`，`top-k-multiplier=2`

### 4.9 向量存储（rag.vector）

`rag.vector.type = pg`（默认）或 `milvus`。`dimension=1536`、`metric-type=COSINE`、默认 collection `rag_default_store`；SSE 全局超时 `sse-timeout-ms=300000`。Milvus 地址 `http://localhost:29530`（可选）。

### 4.10 Trace（rag.trace）

`enabled=true`，`max-error-length=1000`。

### 4.11 认证（sa-token）

header `Authorization`，`timeout=2592000`，`token-style=simple-uuid`。初始账号 `admin / admin`（admin 角色）。

---

## 5. 核心业务流程

本节重点展现业务链路，方便讲解和排查问题。

### 5.1 登录流程

1. 前端在登录页提交用户名和密码。
2. 调用 `POST /auth/login`，由 `AuthController` 处理。
3. `AuthService.login` 校验账号密码，使用 Sa-Token 生成登录 token，并把登录态写入 Redis。
4. 返回包含 token、用户名、角色、头像等的登录信息。
5. 前端将 token 缓存到本地存储，同时在 `api.ts` 中的 axios 拦截器自动带上 `Authorization` 头。
6. 后续请求通过 `UserContext` 获取当前登录用户信息，非管理员接口只要求登录态，管理员接口会调用 `StpUtil.checkRole('admin')` 进行校验。

初始账号：用户名 `admin`、密码 `admin`、角色 `admin`。

### 5.2 构建知识库的流程

1. 在知识库管理页面点击新建，前端调用 `POST /knowledge-base`。
2. `KnowledgeBaseService.create` 创建 `t_knowledge_base` 记录，并返回 kb id。
3. 在知识库详情页上传文档，前端调用 `POST /knowledge-base/{kbId}/docs/upload`，携带文件和上传参数。
4. `KnowledgeDocumentService.upload` 将文件上传到 RustFS，并在 `t_knowledge_document` 中插入记录，状态为待切分。
5. 上传成功后，前端调用 `POST /knowledge-base/docs/{docId}/chunk` 启动切分任务。
6. 后端创建一个 `IngestionTask`，并通过 `IngestionEngine` 按流水线执行：
   - Fetcher 节点：从对象存储或远程源拉取原始文件；
   - Parser 节点：使用 Tika 把文件转换为纯文本；
   - Chunker 节点：根据策略将长文本切分为多个 chunk；
   - Enhancer / Enricher 节点：可选，调用 LLM 进行文本清洗或结构化抽取；
   - Indexer 节点：生成 embedding，写入向量存储（默认 pgvector 的 `t_knowledge_vector`，可选 Milvus），同时把 chunk 元数据写入 `t_knowledge_chunk`。
7. 节点执行状态写入 `t_ingestion_task_node`，异常信息写入日志表 `t_knowledge_document_chunk_log`。
8. 前端通过 `GET /ingestion/tasks/{id}` 和 `GET /knowledge-base/docs/{docId}/chunk-logs` 查看任务和文档的执行情况。

### 5.3 文档定时刷新流程

1. 为某个文档配置定时任务记录 `t_knowledge_document_schedule`，包括 cron 表达式、锁字段、下次执行时间等。
2. `KnowledgeDocumentScheduleJob` 被 `@Scheduled` 周期性触发（参考 `rag.knowledge.schedule`：`scan-delay-ms=10000`、`lock-seconds=900`、`batch-size=20`、`min-interval-seconds=60`）：
   - 扫描需要执行并且未被其他实例锁定的 schedule 记录；
   - 尝试通过乐观锁字段获取执行权；
   - 为每条记录异步提交执行。
3. 在执行过程中：
   - 再次获取对应文档和知识库记录，校验未被删除且仍启用；
   - 若文档被删除或禁用，则关闭该 schedule；
   - 重新拉取远程文档内容，判断 ETag 或哈希是否变化；
   - 如有变化，则重新走一次摄取流水线。
4. 执行结果写入 `t_knowledge_document_schedule_exec`，包括成功与失败情况以及耗时。

### 5.4 用户提问的完整 RAG 流程（15 步）

核心实现集中在 `RAGChatServiceImpl.streamChat` 与 `StreamChatPipeline` 中（编排顺序 memory→rewrite→intent→guidance→retrieve→prompt→LLM）。

1. 前端通过 SSE 连接 `GET /rag/v3/chat`，携带参数 question（用户问题）、conversationId（会话 id，可选）、deepThinking（是否启用思考模式）。
2. 控制器创建 `SseEmitter` 并调用 `ragChatService.streamChat`，返回 SSE 流对象。
3. 生成或复用 `conversationId` 和 `taskId`，并记录日志。
4. 通过 `ConversationMemoryService` 加载最近 N 轮对话和摘要，并将当前问题写入消息表 `t_message`。
5. 调用 `QueryRewriteService.rewriteWithSplit`：对原始问题做语言层规范化与补全，并按需拆分为多个子问题。
6. 调用 `IntentResolver` 将每个子问题映射到意图树（`t_intent_node`）上的多个节点：并发调用意图分类器、过滤低置信度意图、控制总意图数量不超过上限。
7. 调用 `IntentGuidanceService.detectAmbiguity` 检查是否存在模糊或冲突：若需要澄清则直接通过 SSE 返回澄清文案并结束本次流；否则进入下一步。
8. 判断是否所有子问题都只包含系统意图：如果是，则进入纯系统回答流程，不查知识库也不调工具，加载系统对话 Prompt 模板，构造 system + user 两条消息调用 LLM。
9. 若需要检索，调用 `RetrievalEngine.retrieve` → `MultiChannelRetrievalEngine`：
   - 意图定向检索（从意图绑定的知识库召回）+ 全局向量检索（兜底补充召回）；
   - 经过去重（`DeduplicationPostProcessor`）与 Rerank 重排（`RerankPostProcessor`，可通过开关启用），返回 Top K chunk，并按意图分组格式化为 KB 上下文；
   - 对 MCP 意图：用 LLM 按工具 schema 抽取参数 → 通过 HTTP 调用 MCP Server → 将结果格式化为 MCP 上下文。
10. 若最终 KB 和 MCP 上下文全部为空，则退回纯系统回答模式。
11. 合并所有子问题的意图，构造 `IntentGroup`。
12. 使用 `RAGPromptService.buildStructuredMessages` 组合 Prompt：注入重写后的主问题、KB 上下文、MCP 上下文、子问题列表、意图树信息、对话历史。
13. 构造 `ChatRequest` 并调用 `LLMService.streamChat`：根据是否包含 MCP 上下文设置温度等参数，以流式方式逐步推送返回内容。
14. SSE 层将模型输出拆分为多个事件：流元数据事件（任务 id、会话 id 等）、文本增量事件（部分标记为思考内容）、完成事件（最终 messageId 和自动生成的会话标题）。
15. 前端将这些事件递增渲染到对话窗口，结束后支持对消息发送反馈（写入 `t_message_feedback`）。

### 5.5 MCP 工具调用示例流程

1. 意图树中某个叶子节点配置为 MCP 类型，并绑定某个 MCP 工具 id。
2. 用户提出和销售分析相关的问题，例如询问某区域某段时间的业绩排行。
3. 该问题通过意图分类匹配到该 MCP 节点。
4. 检索引擎在构造 MCP 请求时：根据工具 schema 使用 LLM 抽取参数（region、period、product、salesPerson、queryType 等），构造 MCP 请求对象并通过客户端调用 MCP 服务（`http://localhost:29099`）。
5. MCP 端的 `SalesMcpExecutor` 从模拟数据集中筛选符合条件的记录，并根据 queryType 生成相应格式的结果（汇总、排行榜、明细或趋势）。
6. 返回结果被格式化为 MCP 上下文，在最终 Prompt 中以工具结果形式展示给 LLM。
7. 通过 Trace 页面可以详细看到这一步的耗时与参数内容。

---

## 6. 本地运行步骤（基于 PostgreSQL）

下面给出从零开始把项目跑起来的完整、可执行步骤。

### 6.1 准备运行环境

1. 安装并配置好 JDK 17。
2. 安装 Node 与 npm，推荐 Node 版本在 18 以上。
3. 安装 Docker 与 docker compose。
4. 确保端口未被占用：
   - 29090 主服务 bootstrap（context-path `/api/ragent`）
   - 29099 MCP Server
   - 25432 PostgreSQL、26379 Redis、29876 RocketMQ、29000 RustFS、29530 Milvus（可选）
   - 25173 前端开发服务器

### 6.2 启动中间件容器

在 `resources/docker` 目录下分别启动各栈：

```bash
cd resources/docker

# PostgreSQL + pgvector（init/01-init-pgvector.sql 会自动创建 vector 扩展）
docker compose -f postgres-pgvector-stack.compose.yaml up -d

# RocketMQ
docker compose -f rocketmq-stack-5.2.0.compose.yaml up -d

# 可选：Milvus（仅当 rag.vector.type=milvus 时需要）
docker compose -f milvus-stack-2.6.6.compose.yaml up -d
```

Redis 与 RustFS 可使用各自的容器/本地服务（Redis password 为 `123456`，RustFS `rustfsadmin/rustfsadmin`）。通过 `docker ps` 确认容器处于 healthy 状态。

### 6.3 初始化数据库（PostgreSQL）

pgvector 扩展由 `resources/docker/postgres/init/01-init-pgvector.sql` 在容器启动时自动创建。随后执行建表与初始数据脚本：

```bash
# 建表（21 张表）
psql "postgresql://postgres:postgres@127.0.0.1:25432/ragent" -f resources/database/schema_pg.sql

# 初始化数据（含 admin/admin、预置意图树、样例问题等）
psql "postgresql://postgres:postgres@127.0.0.1:25432/ragent" -f resources/database/init_data_pg.sql
```

如需版本升级，可执行 `resources/database/upgrade_v1.0_to_v1.1.sql`、`upgrade_v1.1_to_v1.2.sql`。

> 旧的 MySQL 脚本（`schema_table.sql`、`init_data.sql`）已归档到 `resources/database/backups/`，不要再用它们做主流程。

### 6.4 配置 application.yaml

打开 `bootstrap/src/main/resources/application.yaml`，确认/调整：

- `spring.datasource.url = jdbc:postgresql://127.0.0.1:25432/ragent?client_encoding=UTF8`，账号 `postgres/postgres`；
- `spring.data.redis`（127.0.0.1:26379，password 123456）；
- `rocketmq.name-server = 127.0.0.1:29876`；
- `rag.vector.type`（默认 `pg`，如用 Milvus 改为 `milvus` 并确认 `milvus.uri`）；
- `rustfs.url = http://localhost:29000`；
- `rag.mcp.servers` 中 MCP Server 地址为 `http://localhost:29099`；
- AI key 通过环境变量注入：`BAILIAN_API_KEY`、`SILICONFLOW_API_KEY`、`AIHUBMIX_API_KEY`（Ollama 为本地无需 key）。

```bash
export BAILIAN_API_KEY=your_key
export SILICONFLOW_API_KEY=your_key
export AIHUBMIX_API_KEY=your_key
```

### 6.5 启动后端服务

1. 启动 bootstrap 主服务（29090）：

   ```bash
   ./mvnw -pl bootstrap -am spring-boot:run
   ```

   或打包后运行：

   ```bash
   ./mvnw -pl bootstrap -am clean package -DskipTests
   java -jar bootstrap/target/bootstrap-0.0.1-SNAPSHOT.jar
   ```

2. 启动 mcp-server（29099，推荐）：

   ```bash
   ./mvnw -pl mcp-server -am spring-boot:run
   ```

3. 简单验证：使用 curl 或 Postman 调用 `POST /api/ragent/auth/login`，并检查控制台日志是否有报错。

### 6.6 启动前端

1. 安装依赖：

   ```bash
   cd frontend
   npm install
   ```

2. 开发模式运行（端口 25173）：

   ```bash
   npm run dev
   ```

3. 前端 `.env` 中 `VITE_API_BASE_URL=/api/ragent`；Vite dev server 将 `/api` 代理到 `http://localhost:29090`。访问地址 `http://localhost:25173`。

### 6.7 首次登录和功能验证

1. 打开浏览器访问 `http://localhost:25173`。
2. 使用账号 `admin` 密码 `admin` 登录。
3. 在主对话页面输入几个简单问题，确认对话可以顺利返回。
4. 打开知识库管理页面，查看预置知识库和文档。
5. 上传一份自己的 Markdown 或 PDF 文档，启动切分任务，观察任务列表与日志。
6. 打开后台仪表盘和 Trace 页面，观察对话调用链路。

---

## 7. 前端结构速览

- 技术栈：React 18 + TypeScript + Vite 5 + React Router 6 + Zustand + Radix UI + TailwindCSS + Axios + Recharts + react-markdown + react-virtuoso + sonner。
- `src/pages`：`LoginPage`、`ChatPage`、`admin/*`（dashboard、knowledge 列表/文档/chunk、intent-tree、ingestion、traces、settings、users、sample-questions、query-term-mapping）。
- `src/components`：`ui`（Radix 封装）、`chat`（ChatInput/MessageList/MessageItem/MarkdownRenderer/FeedbackButtons/WelcomeScreen/ThinkingIndicator）、`admin`、`layout`。
- `src/services`：`api.ts`（Axios 实例 + 拦截器，按 code 字段判定成功/失败，401 跳登录）、authService、chatService、sessionService、knowledgeService、dashboardService、ingestionService、ragTraceService、intentTreeService、userService、settingsService、queryTermMappingService、sampleQuestionService。
- `src/stores`：authStore、chatStore、themeStore。
- `src/hooks`：useAuth、useChat、`useStreamResponse`（SSE 流式处理）。
- `src/router.tsx`：`RequireAuth`、`RequireAdmin`、`RedirectIfAuth` 路由守卫。

核心接口：`POST /auth/login`、`POST /auth/logout`；`GET /rag/v3/chat`（SSE 流式对话 + 停止接口）；`/conversations`、`/conversations/{id}/messages`、`/conversations/messages/{id}/feedback`；`/knowledge-base`、`/knowledge-base/{kbId}/docs/upload`、`/knowledge-base/docs/{docId}/chunk`、`/knowledge-base/docs/{docId}/chunk-logs`；`/ingestion/tasks`、`/ingestion/tasks/{id}`；`/admin/dashboard/{overview,performance,trends}`。

---

## 8. 总结

通过以上说明，你可以从三个角度来理解和讲解 Ragent 项目：

1. 作为一个 Java 后端工程，它演示了如何编写模块化、可维护的业务代码，具备完善的基础设施（统一返回、异常体系、幂等、分布式 ID、TTL 上下文透传）和运维能力。
2. 作为一个 RAG 系统，它覆盖了文档解析、Chunking、向量化（pgvector / Milvus 可切换）、多通道检索、Rerank 重排、记忆和摘要等完整链路。
3. 作为一个 Agentic RAG 实战项目，它通过 MCP 工具、意图树、模型路由与三态熔断、队列式限流以及 RAG Trace，把外部系统能力与可观测性引入到对话中，真正体现了 AI 应用工程化的深度。

只要跟着本 README 的结构逐段讲清楚，你就可以在面试或分享中把 Ragent 解释得既专业又落地。
