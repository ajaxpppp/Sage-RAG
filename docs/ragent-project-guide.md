# Ragent 项目深度解析与面试指南

## 一、项目概述

### 1.1 项目名称与定位
- 项目名称：Ragent（RAG + Agent）
- 定位：企业级 RAG 智能体平台
- 一句话介绍：基于 Spring Boot 3 + React 18 + Milvus 2.6 构建的企业级检索增强生成（RAG）智能体平台，支持多轮对话、意图识别、多通道检索、MCP 工具调用、模型路由熔断等核心能力。

### 1.2 项目背景
- 来源：nageoffer（拿个offer）社群开源项目
- 解决的问题：企业内部知识库问答场景中，传统关键词搜索无法理解用户意图，大模型又存在幻觉问题。Ragent 通过 RAG 架构将企业私有知识与大模型能力结合，实现精准、可控、可溯源的智能问答。
- 技术趋势：2024-2025 年 RAG + Agent 是 AI 应用落地的主流架构，该项目完整覆盖了 RAG 全链路，是面试中展示 AI 工程化能力的理想项目。

### 1.3 技术栈总览

后端：
- Java 17 + Spring Boot 3.5.7 + Maven 多模块
- MySQL 8.0（业务数据）+ Redis（缓存/限流/分布式锁）+ Milvus 2.6（向量数据库）
- Sa-Token（认证鉴权）+ MyBatis-Plus（ORM）
- Apache Tika（文档解析）+ MinIO/S3（对象存储）
- Spring AI 1.0.0（AI 模型抽象层）

前端：
- React 18 + TypeScript + Vite
- Ant Design 5 + TailwindCSS
- Zustand（状态管理）+ React Router 7

AI 模型：
- 支持多厂商：通义千问（百炼）、SiliconFlow、Ollama、多吉（Claude）
- Embedding：SiliconFlow BAAI/bge-large-zh-v1.5
- Rerank：SiliconFlow BAAI/bge-reranker-v2-m3

### 1.4 项目规模
- 后端 Java 代码约 40,000 行，前端 TypeScript/React 代码约 18,000 行
- 20+ 数据库表
- 4 个 Maven 模块：bootstrap（业务逻辑）、framework（基础设施）、infra-ai（AI 抽象层）、mcp-server（MCP 服务）

---

## 二、项目架构

### 2.1 模块划分

```
ragent/
├── bootstrap/          # 核心业务模块（RAG、知识库、文档摄取、用户管理）
├── framework/          # 基础设施（异常体系、幂等、分布式ID、链路追踪、SSE）
├── infra-ai/           # AI 抽象层（模型路由、熔断、Chat/Embedding/Rerank 客户端）
├── mcp-server/         # MCP 工具服务（销售数据查询等外部工具）
└── frontend/           # React 前端应用
```

### 2.2 核心 RAG 对话流程（RAGChatServiceImpl）

这是整个项目最核心的链路，面试必须能完整描述：

```
用户提问
  │
  ▼
① 会话记忆加载（ConversationMemoryService.loadAndAppend）
  │  滑动窗口 + 自动摘要压缩，控制上下文长度
  ▼
② 查询改写与拆分（QueryRewriteService.rewriteWithSplit）
  │  上下文补全 + 多问题分解，解决指代消解问题
  ▼
③ 意图识别（IntentResolver.resolve）
  │  三级意图树（Domain→Category→Topic），线程池并行分类
  ▼
④ 歧义引导（IntentGuidanceService.detectAmbiguity）
  │  置信度低时主动追问用户，提升检索精度
  ▼
⑤ 多通道检索（RetrievalEngine.retrieve）
  │  知识库检索 + MCP 工具调用并行执行
  ▼
⑥ Prompt 组装（RAGPromptService.buildStructuredMessages）
  │  场景化模板选择（KB_ONLY / MCP_ONLY / MIXED）
  ▼
⑦ 流式 LLM 响应
  │  SSE 输出，支持深度思考模式
  ▼
返回用户
```

### 2.3 多通道检索架构

```
MultiChannelRetrievalEngine
  │
  ├── IntentDirectedSearchChannel（始终执行）
  │     基于意图识别结果定向检索知识库
  │
  └── VectorGlobalSearchChannel（置信度 < 0.6 时触发）
        全局向量相似度搜索，作为兜底策略
  │
  ▼
后处理链：DeduplicationPostProcessor → RerankPostProcessor
```

所有通道实现 SearchChannel 接口，完全可插拔。

### 2.4 文档摄取流水线（IngestionEngine）

```
FetcherNode（获取文档：S3/本地/HTTP/飞书）
  → ParserNode（解析：Tika 支持 PDF/DOC/DOCX/Markdown）
    → EnhancerNode（增强：元数据补充）
      → ChunkerNode（分块：固定大小/句子/段落/结构感知）
        → EnricherNode（富化：生成 Embedding 向量）
          → IndexerNode（索引：写入 Milvus + MySQL）
```

基于节点图的链式流水线，每个节点可独立替换。

---

## 三、四大核心亮点（面试重点）

### 亮点一：多通道并行检索 + Rerank 精排

**问题**：单一检索策略无法兼顾精确匹配和语义召回，导致检索质量不稳定。

**方案**：
- 设计 SearchChannel 接口抽象检索通道，实现可插拔架构
- IntentDirectedSearchChannel：基于意图识别结果定向检索，精确度高
- VectorGlobalSearchChannel：全局向量搜索兜底，当意图置信度低于 0.6 时自动触发
- 所有通道通过 CompletableFuture 并行执行，不增加延迟
- 检索结果经过去重（DeduplicationPostProcessor）+ Rerank 精排（RerankPostProcessor）
- Rerank 使用 BGE-Reranker-v2-m3 模型对候选文档重新打分排序

**技术细节**：
- MultiChannelRetrievalEngine 作为编排器，管理通道注册、并行执行、结果合并
- 通道决策基于 IntentResult 的置信度分数，实现动态检索策略
- 后处理链采用责任链模式，可灵活扩展（如添加过滤、截断等处理器）

**面试话术**：
> "我负责了多通道检索模块的设计与实现。核心思路是将检索策略抽象为 SearchChannel 接口，实现了意图定向检索和全局向量检索两个通道。通过 CompletableFuture 并行执行所有通道，再经过去重和 Rerank 精排后处理链，最终返回高质量的检索结果。当意图识别置信度低于阈值时，会自动触发全局向量检索作为兜底，保证了检索的鲁棒性。"

### 亮点二：模型路由 + 三态熔断器

**问题**：依赖单一大模型厂商存在可用性风险，模型服务不稳定时会导致整个系统不可用。

**方案**：
- ModelRoutingExecutor 实现模型优先级路由，按配置的优先级依次尝试多个模型
- ModelHealthStore 实现经典三态熔断器（CLOSED -> OPEN -> HALF_OPEN）
- 熔断状态转换：连续失败达阈值 -> OPEN（拒绝请求）-> 超时后 -> HALF_OPEN（放行探测）-> 成功 -> CLOSED
- ProbeBufferingCallback 实现首包探测，确保模型切换时用户无感知
- 支持 4 个模型厂商：多吉（Claude）、百炼（通义千问）、SiliconFlow、Ollama

**技术细节**：
- 熔断器参数：失败阈值 3 次，OPEN 持续时间 60 秒
- 首包探测机制：切换到新模型时，先缓冲第一个响应包，确认模型正常后再开始流式输出
- 模型候选列表通过 YAML 配置，支持动态调整优先级
- 每个模型维护独立的健康状态，互不影响

**面试话术**：
> "我设计了模型路由与熔断机制。核心是 ModelRoutingExecutor 按优先级依次尝试多个模型厂商，配合 ModelHealthStore 的三态熔断器（CLOSED/OPEN/HALF_OPEN）实现故障自动隔离和恢复。当某个模型连续失败 3 次后触发熔断，60 秒后进入半开状态进行探测。同时通过 ProbeBufferingCallback 实现首包探测，确保模型切换对用户完全透明。"

### 亮点三：分布式队列限流（Redis ZSET + Lua + Pub/Sub + Semaphore）

**问题**：大模型调用成本高、响应慢，需要控制并发量防止资源耗尽，同时保证用户体验。

**方案**：
- ChatQueueLimiter 实现分布式队列限流，结合 Redis 和本地信号量双重控制
- Redis ZSET 作为分布式等待队列，score 为时间戳，天然有序
- Lua 脚本保证入队/出队的原子性
- Redis Pub/Sub 实现跨节点的队列变更通知
- 本地 Semaphore 控制单节点并发上限
- 支持排队等待，向用户实时推送排队位置

**技术细节**：
- 入队：ZADD 将请求加入 ZSET，返回排队位置
- 出队：Lua 脚本原子性地检查并移除队首元素
- 通知：出队后通过 Pub/Sub 通知所有节点，等待中的请求尝试获取执行权
- 超时：支持配置最大等待时间，超时自动移除
- 幂等：配合 @IdempotentSubmit 注解防止重复提交

**面试话术**：
> "我实现了基于 Redis 的分布式队列限流方案。使用 ZSET 作为有序等待队列，Lua 脚本保证入队出队的原子性，Pub/Sub 实现跨节点通知，本地 Semaphore 控制单机并发。用户排队时会通过 SSE 实时推送排队位置，超时自动清理。这个方案既控制了大模型调用的并发成本，又保证了用户体验。"

### 亮点四：会话记忆管理（滑动窗口 + 自动摘要压缩）

**问题**：多轮对话场景下，历史消息不断累积会超出大模型的上下文窗口限制，且增加 Token 成本。

**方案**：
- DefaultConversationMemoryService 实现智能记忆管理
- 滑动窗口：保留最近 N 轮对话（默认 10 轮）
- 自动摘要压缩：当历史消息超过窗口大小时，调用 LLM 对早期对话生成摘要
- 摘要作为系统消息注入上下文，保留关键信息的同时大幅减少 Token 消耗
- 记忆加载与追加通过 loadAndAppend 方法一步完成

**技术细节**：
- 窗口大小和摘要触发阈值通过配置文件控制
- 摘要生成使用独立的 LLM 调用，不影响主对话流程
- 摘要结果缓存到数据库，避免重复生成
- 支持会话隔离，每个会话独立维护记忆状态

**面试话术**：
> "我负责了会话记忆管理模块。采用滑动窗口加自动摘要压缩的策略：保留最近 10 轮对话的完整内容，超出窗口的早期对话由 LLM 自动生成摘要。摘要作为系统消息注入上下文，既保留了关键历史信息，又将 Token 消耗控制在可接受范围内。摘要结果会缓存到数据库避免重复生成。"

---

## 四、技术栈详解

### 4.1 后端技术栈

| 技术 | 版本 | 用途 |
|------|------|------|
| Java | 17 | 开发语言 |
| Spring Boot | 3.5.7 | 应用框架 |
| Spring AI | 1.0.0 | AI 模型抽象层 |
| MyBatis-Plus | 3.5.12 | ORM 框架 |
| Sa-Token | 1.40.0 | 认证鉴权 |
| MySQL | 8.0 | 关系型数据库 |
| Redis | - | 缓存/限流/分布式锁/消息通知 |
| Milvus | 2.6 | 向量数据库 |
| Apache Tika | 3.1.0 | 文档解析（PDF/DOC/DOCX/MD） |
| MinIO/S3 | - | 对象存储 |
| Hutool | 5.8.37 | 工具库 |

### 4.2 前端技术栈

| 技术 | 版本 | 用途 |
|------|------|------|
| React | 18 | UI 框架 |
| TypeScript | - | 类型安全 |
| Vite | - | 构建工具 |
| Ant Design | 5 | UI 组件库 |
| TailwindCSS | - | 原子化 CSS |
| Zustand | - | 状态管理 |
| React Router | 7 | 路由管理 |

### 4.3 基础设施亮点

1. **8 个专用线程池**：RAG 对话、意图识别、查询改写、检索、MCP 调用、文档摄取、Embedding、通用任务，每个池独立配置核心线程数、最大线程数、队列容量
2. **TTL 上下文传播**：通过 TransmittableThreadLocal 在线程池中传递用户上下文和链路追踪上下文
3. **全链路追踪**：@RagTraceNode 注解 + AOP 自动记录每个节点的耗时、输入输出
4. **幂等提交**：@IdempotentSubmit 注解 + AOP + Redis 分布式锁，防止重复提交
5. **雪花算法分布式 ID**：保证全局唯一、趋势递增
6. **统一异常体系**：ServiceException / ClientException / RemoteException 三层异常分类

---

## 五、前端页面与功能

### 5.1 页面清单（22 个页面）

**用户端：**
- ChatPage：主对话页面，支持流式输出、深度思考、Markdown 渲染、代码高亮、反馈按钮
- LoginPage：登录页面

**管理端（AdminLayout）：**
- DashboardPage：数据看板（对话量、用户数、知识库统计）
- KnowledgeBasePage：知识库管理（创建、编辑、删除知识库）
- KnowledgeDocumentPage：文档管理（上传、查看、删除文档）
- KnowledgeChunkPage：分块管理（查看文档分块详情）
- IntentTreePage：意图树编辑（可视化编辑三级意图树）
- IngestionPipelinePage：摄取流水线管理
- IngestionTaskPage：摄取任务监控
- TraceListPage：链路追踪列表
- TraceDetailPage：链路追踪详情
- UserManagementPage：用户管理
- SystemSettingsPage：系统设置
- SampleQuestionPage：示例问题管理
- ModelConfigPage：模型配置
- ConversationHistoryPage：对话历史

### 5.2 核心组件

- MessageItem：消息气泡组件，支持用户/AI 消息、Markdown 渲染、代码块
- ChatInput：输入框组件，支持快捷键、示例问题
- MarkdownRenderer：Markdown 渲染器，支持代码高亮、表格、LaTeX
- ThinkingIndicator：深度思考动画指示器
- FeedbackButtons：消息反馈（点赞/点踩）

---

## 六、完整使用步骤

### 6.1 环境准备

1. JDK 17+
2. Node.js 18+
3. MySQL 8.0
4. Redis
5. Milvus 2.6（推荐 Docker 部署）
6. MinIO（可选，用于文档存储）

### 6.2 后端启动

```bash
# 1. 克隆项目
git clone https://github.com/nageoffer/ragent.git

# 2. 创建数据库并导入 SQL
mysql -u root -p < docs/sql/ragent.sql

# 3. 修改配置文件
# bootstrap/src/main/resources/application.yaml
# 配置 MySQL、Redis、Milvus 连接信息
# 配置 AI 模型 API Key（至少配置一个）

# 4. 编译打包
mvn clean package -DskipTests

# 5. 启动
java -jar bootstrap/target/bootstrap.jar
```

### 6.3 前端启动

```bash
cd frontend
npm install
npm run dev
# 访问 http://localhost:5173
```

### 6.4 基本使用流程

1. 登录系统（默认管理员账号见配置文件）
2. 创建知识库：管理端 -> 知识库管理 -> 新建知识库
3. 上传文档：选择知识库 -> 上传文档（支持 PDF/DOC/DOCX/MD）
4. 创建摄取任务：配置摄取流水线 -> 选择分块策略 -> 执行摄取
5. 等待摄取完成：摄取任务监控页面查看进度
6. 配置意图树：管理端 -> 意图树编辑 -> 创建意图分类并关联知识库
7. 开始对话：用户端 -> 输入问题 -> 获取基于知识库的智能回答

---

## 七、面试竞争力分析

### 7.1 项目亮点总结

作为 2026 届应届生面试项目，Ragent 具备以下竞争优势：

1. **技术前沿性**：RAG + Agent 是 2024-2025 年 AI 应用落地的主流架构，展示了对前沿技术的掌握
2. **工程完整性**：从文档摄取、向量化、检索、对话到管理后台，覆盖了完整的产品链路
3. **架构设计深度**：多通道检索、模型熔断、分布式限流等设计体现了扎实的架构能力
4. **技术栈丰富度**：Java + Spring Boot + React + Redis + Milvus + 大模型，覆盖面广

### 7.2 面试常见问题预演

**Q1: 介绍一下你的项目？**

> "Ragent 是一个企业级 RAG 智能体平台，基于 Spring Boot 3 + React 18 + Milvus 构建。核心功能是将企业私有知识库与大模型结合，实现精准的智能问答。我主要负责了多通道检索引擎、模型路由熔断、分布式限流和会话记忆管理四个核心模块的设计与实现。"

**Q2: RAG 的完整流程是什么？**

> "用户提问后，首先加载会话记忆并进行查询改写和拆分，然后通过三级意图树进行意图识别。如果意图不明确会主动追问用户。接着多通道并行检索知识库和调用 MCP 工具，检索结果经过去重和 Rerank 精排后，组装成结构化 Prompt 发送给大模型，最终通过 SSE 流式返回给用户。"

**Q3: 为什么用 Milvus 而不是 Elasticsearch？**

> "Milvus 是专门为向量检索设计的数据库，在高维向量的 ANN（近似最近邻）搜索上性能远优于 ES。我们的场景核心是语义检索，需要对 Embedding 向量做大规模相似度搜索，Milvus 的 HNSW/IVF 索引在这个场景下更合适。ES 更适合关键词检索和全文搜索场景。"

**Q4: 熔断器是怎么实现的？**

> "采用经典的三态熔断器模式。正常状态（CLOSED）下如果连续失败 3 次，切换到熔断状态（OPEN）拒绝所有请求。60 秒后进入半开状态（HALF_OPEN），放行一个探测请求，如果成功则恢复正常，失败则继续熔断。每个模型厂商维护独立的熔断状态。"

**Q5: 分布式限流怎么做的？**

> "使用 Redis ZSET 作为分布式等待队列，Lua 脚本保证入队出队的原子性。出队后通过 Pub/Sub 通知所有节点，配合本地 Semaphore 控制单机并发。用户排队时通过 SSE 实时推送排队位置。"

---

## 八、项目不足与改进建议

### 8.1 现有不足

1. **缺少单元测试和集成测试**：整个项目没有看到测试代码，对于面试来说这是一个明显的短板
2. **缺少 API 文档**：没有集成 Swagger/SpringDoc，接口文档缺失
3. **缺少 Docker Compose 一键部署**：虽然有快速启动文档，但没有提供 Docker Compose 编排文件
4. **前端缺少国际化**：目前只支持中文
5. **缺少监控告警**：没有集成 Prometheus + Grafana 等监控方案
6. **MCP 工具较少**：目前只有销售数据查询一个 MCP 工具示例

### 8.2 改进建议

1. 补充核心模块的单元测试（至少覆盖 RAGChatServiceImpl、MultiChannelRetrievalEngine、ModelRoutingExecutor、ChatQueueLimiter），面试时可以展示 TDD 能力
2. 集成 SpringDoc OpenAPI 生成 API 文档
3. 提供 Docker Compose 文件，包含 MySQL、Redis、Milvus、MinIO、后端、前端的完整编排
4. 添加更多 MCP 工具示例（如天气查询、日程管理），展示 Agent 能力的扩展性
5. 集成 Micrometer + Prometheus 暴露指标，配合 Grafana 看板展示系统运行状态
6. 考虑添加混合检索（向量 + 关键词 BM25），进一步提升检索质量

---

## 九、核心源码文件索引

| 文件 | 说明 |
|------|------|
| RAGChatServiceImpl.java | 核心 RAG 对话流程编排 |
| MultiChannelRetrievalEngine.java | 多通道检索引擎 |
| IntentResolver.java | 三级意图树识别 |
| QueryRewriteService.java | 查询改写与拆分 |
| DefaultConversationMemoryService.java | 会话记忆管理 |
| IntentGuidanceService.java | 歧义引导 |
| RAGPromptService.java | Prompt 组装 |
| MCPServiceOrchestrator.java | MCP 工具编排 |
| ModelRoutingExecutor.java | 模型路由 |
| ModelHealthStore.java | 三态熔断器 |
| ChatQueueLimiter.java | 分布式队列限流 |
| IngestionEngine.java | 文档摄取流水线 |
| ThreadPoolExecutorConfig.java | 8 个专用线程池 |
| application.yaml | 核心配置文件 |

---

以上就是 Ragent 项目的完整深度解析。掌握这份文档的内容，足以在面试中自信地介绍和回答关于该项目的各类问题。

---

## 十、补充：数据库表设计（17 张表）

项目实际包含 17 张业务表，按业务域分组如下：

**会话域（3 张）：**

| 表名 | 说明 | 核心字段 |
|------|------|----------|
| t_conversation | 会话列表 | conversation_id, user_id, title, last_time |
| t_message | 消息记录 | conversation_id, user_id, role(system/user/assistant), content |
| t_conversation_summary | 会话摘要 | conversation_id, last_message_id, content（摘要内容） |

**知识库域（6 张）：**

| 表名 | 说明 | 核心字段 |
|------|------|----------|
| t_knowledge_base | 知识库 | name, embedding_model, collection_name（Milvus Collection） |
| t_knowledge_document | 文档 | kb_id, doc_name, file_url, file_type, status, chunk_strategy, pipeline_id |
| t_knowledge_chunk | 文档分块 | kb_id, doc_id, chunk_index, content, content_hash, char_count |
| t_knowledge_document_chunk_log | 分块日志 | doc_id, status, extract/chunk/embedding_duration, chunk_count |
| t_knowledge_document_schedule | 定时刷新任务 | doc_id, cron_expr, next_run_time, last_etag, last_content_hash |
| t_knowledge_document_schedule_exec | 定时刷新执行记录 | schedule_id, doc_id, status, content_hash, etag |

**摄取流水线域（3 张）：**

| 表名 | 说明 | 核心字段 |
|------|------|----------|
| t_ingestion_pipeline | 流水线定义 | name, description |
| t_ingestion_pipeline_node | 流水线节点配置 | pipeline_id, node_id, node_type, next_node_id, settings_json |
| t_ingestion_task | 摄取任务记录 | pipeline_id, source_type, status, chunk_count, logs_json |
| t_ingestion_task_node | 任务节点执行记录 | task_id, node_id, node_type, status, duration_ms, output_json |

**意图与检索域（2 张）：**

| 表名 | 说明 | 核心字段 |
|------|------|----------|
| t_intent_node | 意图树节点 | intent_code, name, level(0/1/2), parent_code, collection_name, mcp_tool_id, kind(RAG/SYSTEM) |
| t_query_term_mapping | 关键词归一化映射 | source_term, target_term, match_type(精确/前缀/正则/整词), priority |

**链路追踪域（2 张）：**

| 表名 | 说明 | 核心字段 |
|------|------|----------|
| t_rag_trace_run | Trace 运行记录 | trace_id, conversation_id, user_id, status, duration_ms |
| t_rag_trace_node | Trace 节点记录 | trace_id, node_id, parent_node_id, depth, node_type, duration_ms |

**其他（2 张）：**

| 表名 | 说明 | 核心字段 |
|------|------|----------|
| t_user | 系统用户 | username, password, role(admin/user), avatar |
| t_sample_question | 示例问题 | title, description, question |
| t_message_feedback | 消息反馈 | message_id, user_id, vote(1点赞/-1点踩), reason |

---

## 十一、补充：完整 API 接口清单（15 个 Controller）

### 对话相关

| 接口 | 方法 | 说明 |
|------|------|------|
| /api/rag/chat | POST(SSE) | 发起 RAG 对话，流式返回 |
| /api/rag/chat/stop | POST | 停止生成 |
| /api/rag/conversations | GET | 获取会话列表 |
| /api/rag/conversations/{id} | DELETE | 删除会话 |
| /api/rag/conversations/{id}/rename | PUT | 重命名会话 |
| /api/rag/conversations/{id}/messages | GET | 获取会话消息列表 |
| /api/rag/messages/{id}/feedback | POST | 消息反馈（点赞/点踩） |
| /api/rag/sample-questions | GET | 获取示例问题列表 |

### 知识库管理

| 接口 | 方法 | 说明 |
|------|------|------|
| /api/knowledge/bases | GET/POST | 知识库列表/创建 |
| /api/knowledge/bases/{id} | GET/PUT/DELETE | 知识库详情/更新/删除 |
| /api/knowledge/documents | GET/POST | 文档列表/上传 |
| /api/knowledge/documents/{id} | GET/DELETE | 文档详情/删除 |
| /api/knowledge/documents/{id}/chunks | GET | 文档分块列表 |
| /api/knowledge/chunks/{id} | GET | 分块详情 |

### 摄取流水线

| 接口 | 方法 | 说明 |
|------|------|------|
| /api/ingestion/pipelines | GET/POST | 流水线列表/创建 |
| /api/ingestion/pipelines/{id} | GET/PUT/DELETE | 流水线详情/更新/删除 |
| /api/ingestion/tasks | GET/POST | 任务列表/创建 |
| /api/ingestion/tasks/{id} | GET | 任务详情（含节点执行记录） |

### 意图树管理

| 接口 | 方法 | 说明 |
|------|------|------|
| /api/rag/intent-tree | GET | 获取完整意图树 |
| /api/rag/intent-tree/nodes | POST | 创建意图节点 |
| /api/rag/intent-tree/nodes/{id} | PUT/DELETE | 更新/删除意图节点 |

### 系统管理

| 接口 | 方法 | 说明 |
|------|------|------|
| /api/auth/login | POST | 用户登录 |
| /api/auth/logout | POST | 用户登出 |
| /api/auth/current | GET | 获取当前用户信息 |
| /api/users | GET/POST | 用户列表/创建 |
| /api/users/{id} | PUT/DELETE | 更新/删除用户 |
| /api/rag/settings | GET/PUT | 获取/更新系统设置 |
| /admin/dashboard/overview | GET | 数据看板概览 |
| /admin/dashboard/performance | GET | 性能指标 |
| /admin/dashboard/trends | GET | 趋势数据 |
| /api/rag/traces | GET | 链路追踪列表 |
| /api/rag/traces/{id} | GET | 链路追踪详情 |

---

## 十二、补充：容易被忽略的隐藏功能

### 12.1 关键词归一化映射（t_query_term_mapping）

用户输入的口语化表达和系统内部的标准术语往往不一致。例如用户说"报销"，系统里叫"费用报销申请"。

t_query_term_mapping 表实现了关键词归一化：
- 支持 4 种匹配类型：精确匹配、前缀匹配、正则匹配、整词匹配
- 按优先级排序，数值越小优先级越高（优先匹配长词）
- 在查询改写阶段自动替换，提升检索命中率

面试加分点：这个设计体现了对 RAG 检索质量的深入思考——不是所有问题都能靠向量语义解决，关键词归一化是提升精确匹配的有效手段。

### 12.2 文档定时刷新（Schedule）

知识库文档不是一次性导入就完事的，企业场景下文档会持续更新。

- t_knowledge_document_schedule：支持 Cron 表达式配置定时拉取
- 增量更新机制：通过 ETag / Last-Modified / Content Hash 判断文档是否变更，避免无效重建
- 分布式锁：lock_owner + lock_until 防止多节点重复执行
- 执行记录：t_knowledge_document_schedule_exec 记录每次执行的状态和结果

### 12.3 意图树节点的双模式（kind 字段）

意图节点不只是关联知识库，还支持两种模式：
- kind=0（RAG 知识库类）：关联 collection_name，走向量检索
- kind=1（SYSTEM 系统交互类）：关联 mcp_tool_id，走 MCP 工具调用

这意味着同一棵意图树可以同时处理"查知识"和"调系统"两类需求，是 RAG + Agent 融合的关键设计。

### 12.4 SSE 流式输出的三种事件类型

前端通过 SSE 接收三种事件：
- `stream-meta`：流开始，携带 conversationId 和 taskId
- `message-delta`：内容增量，type 区分 content（正文）和 thinking（思考过程）
- `completion`：流结束，携带 messageId 和自动生成的会话标题
