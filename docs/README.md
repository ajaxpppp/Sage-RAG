 # Ragent 项目介绍与业务流程（超详细）

## 1. 项目整体概览

### 1.1 项目定位

Ragent（RAG + Agent）是一个面向企业内部知识问答场景的 Agentic RAG 智能体平台。后端基于 Java 17 和 Spring Boot 3.5.x，多模块 Maven 架构；前端基于 React 18；向量检索使用 Milvus 2.6；配套对象存储 RustFS、Redis、MySQL 等基础设施。

它的设计目标不是简单的演示 Demo，而是覆盖以下完整能力：

- 文档解析、切分、向量化、入库和定时刷新
- 多知识库管理、文档启停、分组和统计
- 多通道检索（意图定向检索加全局兜底检索）和 Rerank 重排
- 对话记忆、摘要、历史窗口控制
- 意图树、关键词归一化、Ambiguity 引导
- MCP 工具调用和参数抽取，将外部系统能力接入对话
- RAG Trace 链路追踪和管理仪表盘
- 完整的用户和权限体系

非常适合作为：

- 学习企业级 RAG 系统的参考实现
- 面试时展示 AI 工程化能力的项目
- 二次开发定制企业内部智能助手的基础骨架

### 1.2 技术栈总览

后端技术栈：

- Java 17
- Spring Boot 3.5.x
- MyBatis Plus，HikariCP
- MySQL 8 业务数据库
- Redis 缓存与分布式锁
- Milvus 2.6 向量数据库
- RustFS（S3 兼容对象存储）
- Apache Tika 文档解析
- 自研 infra ai 模块封装多个模型提供方

前端技术栈：

- React 18
- TypeScript
- Vite
- Radix UI 组件
- TailwindCSS
- Zustand 状态管理
- React Router
- Axios 请求封装
- Recharts 等可视化组件

基础设施：

- Docker Compose 编排 Milvus 加 RustFS 加 etcd 加 Attu
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
  主业务服务，包含 RAG 对话、知识库管理、文档摄取流水线、用户与后台管理等全部业务逻辑。

- `mcp-server`  
  独立的 MCP Tool Server，用于演示如何通过 MCP 将企业内部系统能力接入 RAG 对话。

项目根目录下还有：

- `frontend` 前端 React 工程
- `resources/database` 数据库脚本
- `resources/docker` Milvus 和 RustFS 的 compose 文件
- `docs` 文档目录（本文件即 docs/README.md）

### 2.2 bootstrap 模块包划分

`bootstrap/src/main/java/com/nageoffer/ai/ragent` 下按业务域进行划分：

- `admin`  
  管理后台域。  
  包含 `DashboardController` 和 `DashboardService`，提供 overview、performance、trends 等统计接口，用于仪表盘。

- `user`  
  用户与认证授权。  
  - `config` 中配置 Sa Token 与用户上下文注入  
  - `AuthController` 处理登录和登出  
  - `UserController` 负责用户管理、当前登录用户信息和修改密码  
  - `UserService` 和 `AuthService` 实现具体逻辑  
  - 数据落地在 `t_user` 等表中

- `rag`  
  RAG 对话核心域。主要子包：
  - `controller`  
    - `RAGChatController`：SSE 流式对话接口 `/rag/v3/chat`  
    - `ConversationController`：会话列表与消息列表  
    - `SampleQuestionController`：样例问题  
    - `RAGSettingsController`：系统 RAG 配置查询  
    - `RagTraceController`：RAG Trace 查询接口
  - `service`  
    - `RAGChatService` 及其实现 `RAGChatServiceImpl`  
    - `ConversationService` 与 `ConversationMessageService`  
    - `RagTraceQueryService`、`SampleQuestionService` 等
  - `core`  
    - `memory` 对话记忆与摘要  
    - `rewrite` 查询重写与多问题拆分  
    - `intent` 意图树和意图分类  
    - `retrieve` 多通道检索与 MCP 工具和知识库整合  
    - `guidance` 模糊问题检测与澄清  
    - `prompt` Prompt 构造与模板加载器  
    - `mcp` MCP 工具抽象与客户端  
  - `config`  
    对应 `application.yaml` 中的 rag 和 ai 配置，如 Rate Limit、Memory、Milvus、搜索渠道等  
  - `dao`  
    会话、消息、样例问题、Trace 等领域的 Mapper 和实体  
  - `aop`  
    `ChatRateLimit`、请求队列控制和 Trace 切面

- `knowledge`  
  知识库管理域。  
  - 控制器：`KnowledgeBaseController`、`KnowledgeDocumentController`、`KnowledgeChunkController`  
  - 任务调度：`KnowledgeDocumentScheduleJob` 定时刷新文档  
  - 服务：`KnowledgeBaseService`、`KnowledgeDocumentService` 等  
  - 对应表包括知识库、文档、chunk、schedule 及其执行记录。

- `ingestion`  
  文档摄取流水线域。  
  - `IngestionPipelineController` 管理 Pipeline 模板  
  - `IngestionTaskController` 创建和查询摄取任务  
  - `domain` 描述 Pipeline、节点配置、运行上下文和结果  
  - `engine` 中的 `IngestionEngine` 负责按流水线执行各个节点  
  - `node` 包含具体节点实现，例如 Fetcher、Parser、Chunker、Enricher、Indexer  
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

- `framework/convention/Result` 和 `framework/web/Results`  
  定义统一返回结构和快速构造方法，所有接口返回一个 code 加 message 加 data 的包装对象。

- `framework/exception` 与 `framework/errorcode`  
  约定业务异常层级和错误码枚举，方便上层统一处理。

- `framework/context`  
  `UserContext`、`LoginUser` 负责在业务代码中方便获取当前登录用户信息。

- `framework/idempotent`  
  基于 Redis Template 和 Lua 脚本实现的接口幂等控制，配合注解 `@IdempotentSubmit` 使用。

- `framework/trace`  
  定义 `RagTraceContext` 和 `@RagTraceNode` 注解，配合数据库表记录每一次 RAG 调用链路。

- `framework/web`  
  全局异常处理、统一 SSE 返回封装等。

- `framework/distributedid` 以及 `framework/src/main/resources/lua/snowflake_init.lua`  
  提供分布式雪花 ID 生成功能。

### 2.4 infra ai 模块

主要是对 AI 厂商接口的抽象与封装：

- `chat`  
  定义 `ChatClient` 和 `LLMService` 接口，以及具体实现：
  - 面向 SiliconFlow、百炼、Ollama、多届等的 Chat 客户端
  - `RoutingLLMService` 负责根据配置进行模型路由和降级

- `embedding`  
  封装 Embedding 生成和多模型候选选择逻辑。

- `rerank`  
  包含多种重排模型客户端以及路由服务。

- `model`  
  `ModelSelector`、`ModelRoutingExecutor` 等类，用于根据健康检测、能力标签、优先级来选择后端模型。

- `config/AIModelProperties`  
  映射 `application.yaml` 中 ai 相关配置，并暴露给 `RAGSettingsController` 供前端查看。

### 2.5 mcp server 模块

- `MCPServerApplication` 为独立 Spring Boot 启动类。
- `core` 包中定义 MCP 工具的元模型、请求和响应格式、执行器接口等。
- `endpoint` 包用来接收 JSON RPC 请求，并路由到合适的工具执行器。
- `executor/SalesMCPExecutor` 实现了一个模拟销售数据分析工具：
  - 支持 region、period、product、salesPerson、queryType 等参数；
  - 可以返回 summary、ranking、detail、trend 等不同形式的结果；
  - 内部用随机数据模拟业绩记录，便于本地演示。

bootstrap 模块通过 `rag.core.mcp` 包中的客户端和执行器，作为 MCP 客户端调用这个服务。

---

## 3. 数据模型概览

数据库结构在 `resources/database/schema_table.sql` 中定义，这里只提几个关键领域。

### 3.1 会话与消息

- `t_conversation`  
  存储每个用户的会话列表，包括会话标识、标题、最后消息时间等。

- `t_conversation_summary`  
  存储每个会话的压缩摘要，支持长会话场景下的历史摘要替代。

- `t_conversation_message`  
  存储每一条对话消息，记录角色、内容、是否为思考内容、所属会话和顺序等。

对应服务：

- `ConversationService`  
  负责会话增删改查和重命名。

- `ConversationMessageService` 与 `ConversationMemoryService`  
  负责加载最近 N 轮对话、保存新消息以及触发摘要逻辑。

### 3.2 知识库和文档

- `t_knowledge_base`  
  知识库主表，包含名称、描述、默认向量 collection 名称等。

- `t_knowledge_document`  
  知识库中的单个文档，包括存储位置（RustFS 对象 key）、文件名、类型、大小、启用状态等。

- `t_knowledge_chunk`  
  文档切分后的片段，记录 chunk 内容、所属文档、页码、在 Milvus 中的向量 ID 等信息。

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

- `t_intent_tree_node`  
  存储完整意图树，每个节点可以对应某个知识库、某个 MCP 工具或者系统意图。

- `t_query_term_mapping`  
  存储关键词映射规则，用于把口语化表达映射为标准术语，支持多种匹配方式和优先级。

### 3.5 RAG Trace 与样例问题

- `t_rag_trace_run` 和 `t_rag_trace_node`  
  记录一次 RAG 调用及其内部各步骤（重写、意图识别、检索、MCP 调用、LLM 调用等）的执行信息。

- `t_sample_question`  
  存储首页展示的示例问题，便于引导用户提问。

---

## 4. 核心业务流程

本节重点展现业务链路，方便讲解和排查问题。

### 4.1 登录流程

1. 前端在登录页提交用户名和密码。
2. 调用 `POST /auth/login`，由 `AuthController` 处理。
3. `AuthService.login` 校验账号密码，使用 Sa Token 生成登录 token，并把登录态写入 Redis。
4. 返回 `LoginVO`，包含 token、用户名、角色、头像等。
5. 前端将 token 缓存到本地存储，同时在 `api.ts` 中的 axios 拦截器自动带上 `Authorization` 头。
6. 后续请求通过 `UserContext` 获取当前登录用户信息，非管理员接口只要求登录态，管理员接口会调用 `StpUtil.checkRole('admin')` 进行校验。

初始数据中已经插入账号：

- 用户名 admin  
- 密码 admin  
- 角色 admin

### 4.2 构建知识库的流程

1. 在知识库管理页面，点击新建，前端调用 `POST /knowledge-base`。
2. `KnowledgeBaseService.create` 创建 `t_knowledge_base` 记录，并返回 kb id。
3. 在知识库详情页上传文档，前端调用 `POST /knowledge-base/{kb-id}/docs/upload`，同时携带文件和上传参数。
4. `KnowledgeDocumentService.upload` 将文件上传到 RustFS，并在 `t_knowledge_document` 中插入记录，状态为待切分。
5. 上传成功后，前端调用 `POST /knowledge-base/docs/{doc-id}/chunk` 启动切分任务。
6. 后端创建一个 `IngestionTask`，并通过 `IngestionEngine` 按流水线执行：
   - Fetcher 节点：从对象存储或远程源拉取原始文件；
   - Parser 节点：使用 Tika 把文件转换为纯文本；
   - Chunker 节点：根据策略将长文本切分为多个 chunk；
   - Enricher 节点：可选，调用 LLM 进行文本清洗或结构化抽取；
   - Indexer 节点：生成 embedding，写入 Milvus，同时把 chunk 元数据写入 `t_knowledge_chunk`。
7. 节点执行状态写入 `t_ingestion_task_node`，异常信息写入日志表。
8. 前端通过 `GET /ingestion/tasks/{id}` 和 `GET /knowledge-base/docs/{docId}/chunk-logs` 查看任务和文档的执行情况。

### 4.3 文档定时刷新流程

1. 为某个文档配置定时任务记录 `t_knowledge_document_schedule`，包括 cron 表达式、锁字段、下次执行时间等。
2. `KnowledgeDocumentScheduleJob` 被 `@Scheduled` 周期性触发：
   - 扫描需要执行并且未被其他实例锁定的 schedule 记录；
   - 尝试通过乐观锁字段获取执行权；
   - 为每条记录异步提交执行。
3. 在执行过程中：
   - 再次获取对应文档和知识库记录，校验未被删除且仍启用；
   - 若文档被删除或禁用，则关闭该 schedule；
   - 重新拉取远程文档内容，判断 ETag 或哈希是否变化；
   - 如有变化，则重新走一次摄取流水线。
4. 执行结果写入 `t_knowledge_document_schedule_exec`，包括成功与失败情况以及耗时。

### 4.4 用户提问的完整 RAG 流程

该流程的核心实现集中在 `RAGChatServiceImpl.streamChat` 中。

1. 前端通过 SSE 连接 `GET /rag/v3/chat`，携带参数：
   - question 用户问题  
   - conversationId 会话 id，可选  
   - deepThinking 是否启用思考模式
2. 控制器创建 `SseEmitter` 并调用 `ragChatService.streamChat`，返回 SSE 流对象。
3. 生成或者复用 `conversationId` 和 `taskId`，并记录日志。
4. 通过 `ConversationMemoryService` 加载最近若干轮对话和摘要，并将当前问题写入消息表。
5. 调用 `QueryRewriteService.rewriteWithSplit`：
   - 对原始问题进行语言层的规范化与补全；
   - 跟据需要拆分为多个子问题。
6. 调用 `IntentResolver.resolve` 将每个子问题映射到意图树上的多个节点：
   - 并发调用意图分类器；
   - 过滤低置信度意图；
   - 控制总意图数量不超过上限。
7. 调用 `IntentGuidanceService.detectAmbiguity` 检查是否存在模糊或冲突：
   - 若需要澄清，则直接通过 SSE 返回澄清文案并结束本次流；
   - 否则进入下一步。
8. 判断是否所有子问题都只包含系统意图：
   - 如果是，则进入纯系统回答流程，不查知识库也不调工具；
   - 流程中加载系统对话 Prompt 模板，构造 system 加 user 两条消息，调用 LLM 返回结果。
9. 若需要检索，调用 `RetrievalEngine.retrieve`：
   - 对每个子问题：
     - 提取需要知识库检索的意图；
     - 提取需要 MCP 调用的意图。
   - 调用多通道检索引擎 `MultiChannelRetrievalEngine`：
     - 意图定向检索，从意图所绑定的知识库中召回；
     - 全局向量检索，用于兜底补充召回；
     - 经过去重和重新排序，返回 Top K chunk。
   - 将 chunk 按意图分组，格式化为 KB 上下文字符串。
   - 对 MCP 意图，构造 MCP 请求：
     - 利用 LLM 根据 schema 抽取工具所需参数；
     - 通过 HTTP 调用 MCP Tool Server；
     - 将成功返回的结果格式化为 MCP 上下文。
10. 若最终 KB 和 MCP 上下文全部为空，则退回纯系统回答模式。
11. 否则，合并所有子问题的意图，构造 `IntentGroup`。
12. 使用 `RAGPromptService.buildStructuredMessages` 组合 Prompt：
    - 注入重写后的主问题；
    - 注入 KB 上下文和 MCP 上下文；
    - 注入子问题列表和意图树信息；
    - 注入对话历史。
13. 构造 `ChatRequest` 并调用 `LLMService.streamChat`：
    - 根据是否包含 MCP 上下文设置温度等参数；
    - 以流式方式逐步推送返回内容。
14. SSE 层将模型输出拆分为多个事件：
    - 流元数据事件，包含任务 id、会话 id 等；
    - 文本增量事件，其中部分标记为思考内容；
    - 完成事件，附带最终消息 id 和自动生成的会话标题。
15. 前端将这些事件递增渲染到对话窗口中，并支持在结束后发送反馈。

### 4.5 MCP 工具调用示例流程

1. 意图树中某个叶子节点配置为 MCP 类型，并绑定某个 MCP 工具 id。
2. 用户提出和销售分析相关的问题，例如询问某区域某段时间的业绩排行。
3. 该问题通过意图分类匹配到该 MCP 节点。
4. 检索引擎在构造 MCP 请求时：
   - 根据工具 schema 使用 LLM 抽取参数 region、period、product、salesPerson、queryType 等；
   - 构造 MCP 请求对象并通过客户端调用 MCP 服务。
5. MCP 端的 `SalesMCPExecutor` 从模拟数据集中筛选符合条件的记录，并根据 queryType 生成相应格式的结果（汇总、排行榜、明细或趋势）。
6. 返回结果被格式化为 MCP 上下文，在最终 Prompt 中以工具结果形式展示给 LLM。
7. 通过 Trace 页面可以详细看到这一步的耗时与参数内容。

---

## 5. 本地运行步骤

下面给出从零开始把项目跑起来的完整步骤。

### 5.1 准备运行环境

1. 安装并配置好 JDK 17。
2. 安装 Node 与 npm，推荐 Node 版本在 18 以上。
3. 安装 MySQL 并启动，准备好一个具有建库建表权限的用户。
4. 安装 Redis 并启动。
5. 安装 Docker 与 docker compose。
6. 确保端口未被占用：
   - 9090 预留给主服务 bootstrap
   - 9091 预留给 MCP Server
   - 19530、9000 等预留给 Milvus 与 RustFS
   - 5173 预留给前端开发服务器

### 5.2 初始化数据库

1. 在项目根目录执行数据库脚本：

   ```bash
   mysql -uroot -proot < resources/database/schema_table.sql
   mysql -uroot -proot ragent < resources/database/init_data.sql
   ```

   如果数据库账号密码不同，请替换为自己的参数。

2. 检查表和数据是否生成成功。

### 5.3 启动 Milvus 和 RustFS

1. 在项目根目录执行：

   ```bash
   cd resources/docker
   docker compose -f milvus-stack-2.6.6.compose.yaml up -d
   ```

2. 等待容器全部处于 healthy 状态，可以通过 `docker ps` 查看。
3. 可选择访问 Attu 控制台地址，默认为 `http://localhost:8000`，验证 Milvus 是否可用。

### 5.4 修改 application.yaml（如有必要）

1. 打开 `bootstrap/src/main/resources/application.yaml`。
2. 根据本地情况调整：
   - `spring.datasource` 中的用户名、密码和数据库地址；
   - `spring.data.redis` 中的地址和密码；
   - `milvus.uri` 是否为 `http://localhost:19530`；
   - `rustfs` 地址是否和 docker compose 保持一致；
   - `rag.mcp.servers` 列表中 MCP Server 的地址是否为 `http://localhost:9091`。
3. 若 AI 模型使用自己的账号，将配置中的 api key 改为从环境变量读取，并在系统环境中注入。

### 5.5 启动后端服务

1. 启动 bootstrap 模块：

   ```bash
   # 在项目根目录
   .\mvnw -pl bootstrap -am spring-boot:run
   ```

   或打包后运行：

   ```bash
   .\mvnw -pl bootstrap -am clean package -DskipTests
   java -jar .\bootstrap\target\bootstrap-0.0.1-SNAPSHOT.jar
   ```

2. 启动 mcp server 模块（可选但推荐）：

   ```bash
   .\mvnw -pl mcp-server -am spring-boot:run
   ```

   或打包后运行对应 jar。

3. 简单验证：

   - 使用 curl 或 Postman 调用登录接口；  
   - 检查控制台日志是否有报错。

### 5.6 启动前端

1. 安装依赖：

   ```bash
   cd frontend
   npm install
   ```

2. 开发模式运行：

   ```bash
   npm run dev
   ```

   默认端口为 5173，对应地址为 `http://localhost:5173`。

3. 前端会读取 `.env` 中的 `VITE_API_BASE_URL`，默认指向 `http://localhost:9090/api/ragent`，与后端一致时即可正常访问。

### 5.7 首次登录和功能验证

1. 打开浏览器访问 `http://localhost:5173`。
2. 使用账号 admin 密码 admin 登录。
3. 在主对话页面输入几个简单问题，确认对话可以顺利返回。
4. 打开知识库管理页面，查看预置知识库和文档。
5. 上传一份自己的 Markdown 或 PDF 文档，启动切分任务，观察任务列表与日志。
6. 打开后台仪表盘和 Trace 页面，观察对话调用链路。

---

## 6. 总结

通过以上说明，你可以从三个角度来理解和讲解 Ragent 项目：

1. 作为一个 Java 后端工程，它演示了如何编写模块化、可维护的业务代码，具备完善的基础设施和运维能力。
2. 作为一个 RAG 系统，它覆盖了文档解析、Chunking、向量检索、多通道检索、重排、记忆和摘要等完整链路。
3. 作为一个 Agentic RAG 实战项目，它通过 MCP 工具、意图树和 Trace，把外部系统和可观测性引入到对话中，真正体现了 AI 应用工程化的深度。

只要跟着本 README 的结构逐段讲清楚，你就可以在面试或分享中把 Ragent 解释得既专业又落地。
