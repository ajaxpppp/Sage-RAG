# 快速开始指南

本指南帮助你从零开始，在本地把 **Ragent**（企业级 Agentic RAG 智能体平台）完整跑起来，并验证核心功能。文档同时保留了平台「多通道检索 + 后置处理器」检索架构的说明，便于在跑通后理解检索链路。

> 说明：本文所有端口、地址、账号均以仓库 `mading/main` 为准。若你的本地工作区存在差异（例如前端端口、API 指向、模型供应商等），请以本文为准重新对齐。

---

## 0. 技术栈与组件总览

| 维度 | 内容 |
| --- | --- |
| 语言 / 框架 | Java 17，Spring Boot 3.5.7 |
| 构建 | Maven 多模块：`framework` / `infra-ai` / `bootstrap` / `mcp-server` |
| 业务库 | **PostgreSQL（启用 pgvector 扩展）**，HikariCP 连接池 |
| 向量存储 | 可切换 `rag.vector.type = pg`（默认）`| milvus`，dimension=1536，metric=COSINE |
| 缓存 / 分布式锁 | Redis + Redisson |
| 消息队列 | RocketMQ 5.2.0 |
| 对象存储 | RustFS（S3 兼容） |
| 鉴权 | Sa-Token（header `Authorization`） |
| 前端 | React 18 + TypeScript + Vite 5 + Zustand + Radix UI + TailwindCSS + Axios |

### 端口与地址一览

| 服务 | 地址 / 端口 | 说明 |
| --- | --- | --- |
| 后端主服务 bootstrap | `http://localhost:29090` | context-path：`/api/ragent` |
| mcp-server | `http://localhost:29099` | MCP 工具服务，bootstrap 作为客户端调用它 |
| 前端 dev（Vite） | `http://localhost:25173` | 代理 `/api` → `http://localhost:29090` |
| PostgreSQL | `127.0.0.1:25432` | 库 `ragent`，账号 `postgres / postgres` |
| Redis | `127.0.0.1:26379` | password `123456` |
| RocketMQ NameServer | `127.0.0.1:29876` | broker 另暴露 20911/20909/20912 等 |
| Milvus（可选） | `http://localhost:29530` | 仅当 `rag.vector.type=milvus` 时需要 |
| RustFS（S3 兼容） | `http://localhost:29000` | `rustfsadmin / rustfsadmin` |
| 初始登录账号 | `admin / admin` | admin 角色 |

---

## 1. 前置环境

请先安装：

- **JDK 17**（必须；项目 `<java.version>17</java.version>`）
- **Node.js 18+**（前端 Vite 5 需要）
- **Docker / Docker Compose**（用于启动 PostgreSQL、RocketMQ、可选 Milvus 等中间件）

验证版本：

```bash
java -version     # 期望 17.x
node -v           # 期望 v18 及以上
docker -v
docker compose version
```

项目根目录已自带 Maven Wrapper（`./mvnw`），无需单独安装 Maven。

---

## 2. 用 Docker 启动中间件

所有编排文件位于仓库的 `resources/docker/` 目录下。下面命令均假设你在仓库根目录执行。

### 2.1 启动 PostgreSQL（pgvector）

```bash
docker compose -f resources/docker/postgres-pgvector-stack.compose.yaml up -d
```

该编排会启动：

- `ragent-postgres`（镜像 `pgvector/pgvector:pg17`，端口 `25432`，库 `ragent`，账号 `postgres/postgres`）
- `ragent-pgadmin`（pgAdmin4，端口 `25050`，账号 `admin@ragent.local / admin123456`，可选可视化工具）

容器启动时会自动执行挂载目录 `resources/docker/postgres/init/01-init-pgvector.sql`，其内容为：

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

即首次启动会自动为 `ragent` 库创建 pgvector 扩展，**无需手动安装扩展**。

等待容器健康：

```bash
docker compose -f resources/docker/postgres-pgvector-stack.compose.yaml ps
# 直到 ragent-postgres 状态为 healthy
```

### 2.2 启动 RocketMQ

```bash
# Apple Silicon / ARM
docker compose -f resources/docker/rocketmq-stack-5.2.0.compose.yaml up -d

# x86_64 / amd 机器请用 amd 版编排
# docker compose -f resources/docker/rocketmq-stack-amd-5.2.0.compose.yaml up -d
```

NameServer 监听 `29876`，broker 暴露 20911/20909/20912 及 dashboard 相关端口。

### 2.3 启动 Redis

仓库未提供独立的 Redis 编排文件，可用一行命令启动一个带密码的 Redis：

```bash
docker run -d --name ragent-redis -p 26379:26379 redis:7 \
  redis-server --requirepass 123456
```

> 密码必须与 `application.yaml` 中 `spring.data.redis.password` 一致，默认 `123456`。

### 2.4 启动 Milvus（可选）

只有当你把向量存储切换为 `milvus`（`rag.vector.type=milvus`）时才需要：

```bash
docker compose -f resources/docker/milvus-stack-2.6.6.compose.yaml up -d
```

`resources/docker/lightweight/` 下还提供了 2.5.8、2.6.6 的轻量版编排（参考其 `README.md`），适合资源有限的本地环境。默认使用 PostgreSQL 作为向量库时可跳过本步。

### 2.5 准备对象存储 RustFS（按需）

知识库文档上传依赖 S3 兼容的对象存储 RustFS（`http://localhost:29000`，`rustfsadmin/rustfsadmin`）。若暂不验证文档上传，可后置准备；如需启动，请参考 RustFS 官方部署方式并保证地址与凭证与 `application.yaml` 的 `rustfs` 段一致。

---

## 3. 初始化数据库

PostgreSQL 启动后，需要建表并写入初始数据。**注意：使用 PostgreSQL 脚本，不要使用任何 MySQL 脚本**（旧 MySQL 脚本已归档到 `resources/database/backups/`，仅作历史参考）。

执行顺序：

1. `resources/database/schema_pg.sql` —— 建表（共 21 张表）
2. `resources/database/init_data_pg.sql` —— 初始数据（写入 admin 账号）

> pgvector 扩展已在第 2.1 步由 `01-init-pgvector.sql` 自动创建，此处无需重复。

用 `psql` 执行（在仓库根目录）：

```bash
# 建表
PGPASSWORD=postgres psql -h 127.0.0.1 -p 25432 -U postgres -d ragent \
  -f resources/database/schema_pg.sql

# 初始数据（创建 admin/admin）
PGPASSWORD=postgres psql -h 127.0.0.1 -p 25432 -U postgres -d ragent \
  -f resources/database/init_data_pg.sql
```

若本机未装 `psql`，可借助容器内的客户端执行（先把脚本拷进容器，或用挂载方式）：

```bash
docker exec -i ragent-postgres psql -U postgres -d ragent < resources/database/schema_pg.sql
docker exec -i ragent-postgres psql -U postgres -d ragent < resources/database/init_data_pg.sql
```

执行后会创建以下 21 张表：`t_user`、`t_conversation`、`t_conversation_summary`、`t_message`、`t_message_feedback`、`t_sample_question`、`t_knowledge_base`、`t_knowledge_document`、`t_knowledge_chunk`、`t_knowledge_document_chunk_log`、`t_knowledge_document_schedule`、`t_knowledge_document_schedule_exec`、`t_intent_node`、`t_query_term_mapping`、`t_rag_trace_run`、`t_rag_trace_node`、`t_ingestion_pipeline`、`t_ingestion_pipeline_node`、`t_ingestion_task`、`t_ingestion_task_node`、`t_knowledge_vector`。

`init_data_pg.sql` 会写入初始 admin 用户：

```text
username = admin
password = admin
role     = admin
```

> 升级老库时可按需执行 `upgrade_v1.0_to_v1.1.sql`、`upgrade_v1.1_to_v1.2.sql`；全新库直接用上面的 schema + init 即可。

---

## 4. 配置 `application.yaml`

后端主配置位于：

```
bootstrap/src/main/resources/application.yaml
```

默认配置已经指向本地中间件，**通常无需修改连接信息**，核对一致即可：

```yaml
server:
  port: 29090
  servlet:
    context-path: /api/ragent

spring:
  datasource:
    driver-class-name: org.postgresql.Driver
    type: com.zaxxer.hikari.HikariDataSource
    username: postgres
    password: postgres
    url: jdbc:postgresql://127.0.0.1:25432/ragent?client_encoding=UTF8
  data:
    redis:
      host: 127.0.0.1
      port: 26379
      password: 123456

rocketmq:
  name-server: 127.0.0.1:29876

milvus:
  uri: http://localhost:29530        # 仅 rag.vector.type=milvus 时生效

rustfs:
  url: http://localhost:29000
  access-key-id: rustfsadmin
  secret-access-key: rustfsadmin

rag:
  vector:
    type: pg                         # 默认 pg；可改为 milvus
```

### 4.1 配置 AI 供应商密钥（重要）

AI 供应商的 API Key 通过**环境变量**注入，配置文件里用占位符引用，请勿把明文密钥写进 yaml：

```yaml
ai:
  providers:
    ollama:
      url: http://localhost:11434      # 本地 Ollama，无需 key
    bailian:                            # 阿里百炼 dashscope
      url: https://dashscope.aliyuncs.com
      api-key: ${BAILIAN_API_KEY:}
    aihubmix:
      url: https://aihubmix.com
      api-key: ${AIHUBMIX_API_KEY:}
    siliconflow:
      url: https://api.siliconflow.cn
      api-key: ${SILICONFLOW_API_KEY:}
```

在启动后端前，导出对应环境变量（按你实际使用的供应商填写，未使用的可留空）：

```bash
export BAILIAN_API_KEY=你的百炼key
export SILICONFLOW_API_KEY=你的硅基流动key
export AIHUBMIX_API_KEY=你的aihubmix-key
```

> 内置供应商为 `ollama`、`bailian`、`aihubmix`、`siliconflow` 四家，并无其它「自定义/第三方」内置供应商。

各类模型的默认与候选（按 priority）：

- **对话模型** `ai.chat`：default-model 与 deep-thinking-model 均为 `qwen3-max`；候选含 `qwen-plus`(bailian,p1)、`qwen3-local`(ollama,p2)、`qwen3-max`(bailian,supports-thinking,p3)、`glm-4.7`(siliconflow,p4)、`gpt-5.4`(aihubmix,p5)
- **Embedding** `ai.embedding`：default `qwen-emb-8b`(siliconflow,Qwen/Qwen3-Embedding-8B,p1)、`qwen-emb-local`(ollama,p2)、`text-embedding-3-large`(aihubmix,p3)，dimension=1536
- **Rerank** `ai.rerank`：default `qwen3-rerank`(bailian,p1)、`rerank-noop`(noop,p100)

平台内置三态熔断器（CLOSED → OPEN → HALF_OPEN，`ai.selection.failure-threshold=2`、`open-duration-ms=30000`）做模型路由与降级，配置不全的供应商会被自动跳过。

---

## 5. 启动后端服务

在仓库根目录、且已导出 AI 密钥环境变量后执行。

### 5.1 启动 bootstrap 主服务（29090）

```bash
./mvnw -pl bootstrap -am spring-boot:run
```

启动成功后服务监听 `http://localhost:29090`，全部接口前缀为 `/api/ragent`。

健康自检（应有响应而非连接拒绝）：

```bash
curl -i http://localhost:29090/api/ragent/auth/login -X POST \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin"}'
```

### 5.2 启动 mcp-server（29099）

新开一个终端，在仓库根目录：

```bash
./mvnw -pl mcp-server -am spring-boot:run
```

mcp-server 监听 `http://localhost:29099`，提供销售 / 工单 / 天气等 MCP 工具执行器，由 bootstrap 通过 `rag.mcp.servers`（默认 `http://localhost:29099`）作为 MCP 客户端调用。

> 首次构建会拉取依赖，耗时取决于网络。也可先 `./mvnw -pl bootstrap,mcp-server -am package -DskipTests` 再 `java -jar` 运行各自产物。

---

## 6. 启动前端

```bash
cd frontend
npm install
npm run dev
```

Vite dev server 默认监听 **25173**，并将 `/api` 代理到 `http://localhost:29090`。

前端 `frontend/.env` 应配置为走代理：

```env
VITE_API_BASE_URL=/api/ragent
VITE_APP_NAME=Ragent 智能问答
```

> 说明：因为 `VITE_API_BASE_URL=/api/ragent` 是相对路径，前端请求会先打到 Vite 的 `/api` 代理，再被转发到后端 `29090` 的 `/api/ragent` 前缀。请确保 `.env` 不要写成指向其它端口的绝对地址。

启动后浏览器访问：

```
http://localhost:25173
```

---

## 7. 登录与功能验证

### 7.1 登录

打开 `http://localhost:25173`，使用初始账号登录：

```
账号：admin
密码：admin
```

登录走 `POST /auth/login`，登出走 `POST /auth/logout`，鉴权使用 Sa-Token，token 通过 `Authorization` 头携带。

### 7.2 验证清单

按以下顺序逐项验证，确保整链路打通：

1. **登录成功** → 进入主界面（admin 角色可见后台菜单）。
2. **知识库**：在 `知识库管理` 新建知识库，上传一篇文档（依赖 RustFS 与 RocketMQ，文档会经摄取流水线解析、切分、嵌入、入库）。
3. **意图树**：在 `意图树管理` 查看 / 编辑意图节点（`t_intent_node`）。
4. **对话问答**：进入聊天页提问。前端通过 SSE 连接 `GET /rag/v3/chat`（参数 `question`、`conversationId`、`deepThinking`），后端流式返回「元数据 / 文本增量（含思考）/ 完成（含 messageId 与自动标题）」。
5. **消息反馈**：对回答进行点赞 / 点踩，数据写入 `t_message_feedback`（`/conversations/messages/{id}/feedback`）。
6. **Trace 追踪**：在 `Trace 管理` 查看本次问答的链路节点（`rag.trace.enabled=true`）。
7. **Dashboard**：查看 `/admin/dashboard` 的 overview / performance / trends 指标。

### 7.3 常见排查

- 登录返回连接错误：确认后端 29090 已启动、PostgreSQL 已建表并写入了 admin。
- 对话报模型相关错误：确认已导出对应 `*_API_KEY` 环境变量，且所选供应商可用；未配置 key 的候选会被熔断器跳过。
- 文档上传失败：确认 RustFS（29000）与 RocketMQ（29876）已就绪。
- 前端 401 自动跳登录：token 过期或未携带，重新登录即可。

---

## 8. 检索架构：多通道检索 + 后置处理器

> 以下为平台检索链路的架构说明，跑通后阅读有助于理解 `RetrievalEngine` 内部行为。检索通道与阈值由 `application.yaml` 的 `rag.search.channels` 控制，已为合理默认值，无需为跑通而修改。

平台实现了**多通道检索 + 后置处理器**的可扩展架构，解决意图识别覆盖率不足的问题。核心组件位于：

```
bootstrap/.../rag/core/retrieve/
├── RetrievalEngine.java                 # 检索引擎入口
├── MultiChannelRetrievalEngine.java     # 多通道检索引擎
├── channel/
│   ├── SearchChannel.java               # 检索通道接口
│   └── impl/
│       ├── VectorGlobalSearchChannel.java     # 向量全局检索
│       └── IntentDirectedSearchChannel.java   # 意图定向检索
└── postprocessor/
    ├── SearchResultPostProcessor.java   # 后置处理器接口
    └── impl/
        ├── DeduplicationPostProcessor.java    # 去重处理器
        └── RerankPostProcessor.java           # Rerank 处理器
```

底层检索由 `RetrieverService` 实现，按 `rag.vector.type` 选择 `PgRetrieverService`（pgvector）或 `MilvusRetrieverService`。

### 8.1 检索流程

```
用户问题
    ↓
【意图识别】（IntentResolver 将子问题映射到意图树节点）
    ↓
【多通道并行检索】
    ├─→ 意图定向检索（始终执行，min-intent-score 过滤低分意图）
    └─→ 向量全局检索（条件触发：意图置信度 < confidence-threshold）
    ↓
【后置处理器链】
    ├─→ 去重（合并多通道结果）
    └─→ Rerank（重排序）
    ↓
【返回 Top-K 结果】
```

### 8.2 条件触发逻辑

**向量全局检索**在以下情况下启用：

- 没有识别出任何意图
- 意图置信度都很低（< `confidence-threshold`，默认 0.6）

这样既保证了覆盖率，又避免了不必要的性能开销。

### 8.3 完整问答链路（`RAGChatServiceImpl.streamChat` / `StreamChatPipeline`）

1. 前端 SSE 连 `GET /rag/v3/chat`（question / conversationId / deepThinking）
2. 创建 `SseEmitter`，调用 `streamChat`，生成或复用 conversationId + taskId
3. `ConversationMemoryService` 加载最近 N 轮对话 + 摘要，当前问题写入 `t_message`
4. `QueryRewriteService.rewriteWithSplit`：规范化补全 + 拆分子问题
5. `IntentResolver` 把子问题映射到意图树节点（并发分类、过滤低置信、限总数）
6. `IntentGuidanceService.detectAmbiguity`：需澄清则直接 SSE 返回澄清文案并结束
7. 若全是系统意图 → 纯系统回答（不检索不调工具）
8. 否则 `RetrievalEngine.retrieve` → `MultiChannelRetrievalEngine`：意图定向检索 + 全局向量兜底 → 去重 + Rerank → Top-K；MCP 意图用 LLM 按 schema 抽参后 HTTP 调 MCP Server
9. KB + MCP 上下文全空 → 退回纯系统回答
10. 合并意图构造 IntentGroup，`RAGPromptService.buildStructuredMessages` 组装 Prompt（主问题 + KB 上下文 + MCP 上下文 + 子问题 + 意图树 + 历史）
11. `LLMService.streamChat` 流式输出 → SSE 事件拆为「元数据 / 文本增量（含思考）/ 完成」
12. 前端递增渲染，结束后可对消息反馈（`t_message_feedback`）

---

## 9. 检索配置调整（可选）

如需调整检索策略，修改 `application.yaml` 的 `rag.search.channels`：

```yaml
rag:
  search:
    channels:
      vector-global:
        confidence-threshold: 0.6  # 意图置信度阈值，低于此值时触发全局检索
        top-k-multiplier: 3        # 全局检索召回倍数
      intent-directed:
        min-intent-score: 0.4      # 最低意图分数，低于此分数的意图节点被过滤
        top-k-multiplier: 2        # 意图定向召回倍数
```

**配置说明**：

- `confidence-threshold`：意图置信度阈值，低于此值时启用全局检索（提高它会减少全局检索触发）
- `top-k-multiplier`：召回候选倍数
- `min-intent-score`：最低意图分数，低于此分数的意图节点会被过滤

**注意**：后置处理器（去重、Rerank）是代码层面的逻辑，由组件链编排，不通过该配置开关控制；如需新增/修改后置处理器，直接实现 `SearchResultPostProcessor` 接口并注册为 Spring Bean 即可。

### 启动日志参考

启动后可在日志中观察检索通道执行情况：

```
INFO  启用的检索通道：[IntentDirectedSearch, VectorGlobalSearch]
INFO  通道 IntentDirectedSearch 完成，检索到 15 个 Chunk，置信度：0.85
INFO  通道 VectorGlobalSearch 完成，检索到 20 个 Chunk，置信度：0.7
INFO  执行后置处理器：Deduplication → 去重完成，输入 35，输出 28
INFO  执行后置处理器：Rerank → Rerank 完成，输出 5
```

---

## 10. 扩展示例

### 10.1 新增自定义检索通道

实现 `SearchChannel` 接口并注册为 Spring Bean（`@Component`）：

```java
@Component
public class KeywordESSearchChannel implements SearchChannel {

    @Override
    public String getName() { return "KeywordESSearch"; }

    @Override
    public int getPriority() { return 5; }   // 中等优先级

    @Override
    public boolean isEnabled(SearchContext context) {
        // 例如：当问题包含专有名词或代码片段时启用
        return context.getMainQuestion().contains("```");
    }

    @Override
    public SearchChannelResult search(SearchContext context) {
        List<RetrievedChunk> chunks = searchByKeywords(
            context.getMainQuestion(), context.getTopK() * 2);
        return SearchChannelResult.builder()
            .channelName(getName())
            .chunks(chunks)
            .confidence(0.8)
            .build();
    }
}
```

### 10.2 新增自定义后置处理器

实现 `SearchResultPostProcessor` 接口并注册为 Spring Bean：

```java
@Component
public class VersionFilterPostProcessor implements SearchResultPostProcessor {

    @Override
    public String getName() { return "VersionFilter"; }

    @Override
    public int getOrder() { return 2; }   // 去重之后、Rerank 之前

    @Override
    public boolean isEnabled(SearchContext context) { return true; }

    @Override
    public List<RetrievedChunk> process(List<RetrievedChunk> chunks,
                                        List<SearchChannelResult> results,
                                        SearchContext context) {
        // 按文档分组后只保留最新版本等自定义逻辑
        return chunks;
    }
}
```

---

## 11. 常见问题

### Q1：为什么有时候会执行两次检索？

当意图识别置信度低于阈值（默认 0.6）时，会同时执行意图定向检索和全局检索，以保证覆盖率。

### Q2：如何提高/降低全局检索的触发频率？

调整 `rag.search.channels.vector-global.confidence-threshold`：调高会减少全局检索触发，调低会更频繁触发。

### Q3：向量库用 PostgreSQL 还是 Milvus？

默认 `rag.vector.type=pg`（pgvector）。若要用 Milvus，先用 `resources/docker/milvus-stack-2.6.6.compose.yaml` 启动 Milvus，再把该配置改为 `milvus` 并确认 `milvus.uri` 正确。

### Q4：登录账号是什么？

`admin / admin`（由 `init_data_pg.sql` 写入），admin 角色。

### Q5：如何新增检索通道 / 后置处理器？

分别实现 `SearchChannel` / `SearchResultPostProcessor` 接口并用 `@Component` 注册为 Spring Bean 即可，引擎会自动发现并编排。

---

## 12. 一键回顾（最小命令序列）

```bash
# 1) 启中间件
docker compose -f resources/docker/postgres-pgvector-stack.compose.yaml up -d
docker compose -f resources/docker/rocketmq-stack-5.2.0.compose.yaml up -d
docker run -d --name ragent-redis -p 26379:26379 redis:7 redis-server --requirepass 123456

# 2) 初始化库
PGPASSWORD=postgres psql -h 127.0.0.1 -p 25432 -U postgres -d ragent -f resources/database/schema_pg.sql
PGPASSWORD=postgres psql -h 127.0.0.1 -p 25432 -U postgres -d ragent -f resources/database/init_data_pg.sql

# 3) 配置 AI key 并启后端
export BAILIAN_API_KEY=... SILICONFLOW_API_KEY=... AIHUBMIX_API_KEY=...
./mvnw -pl bootstrap -am spring-boot:run        # 29090
./mvnw -pl mcp-server -am spring-boot:run        # 29099（新终端）

# 4) 启前端
cd frontend && npm install && npm run dev        # 25173

# 5) 浏览器访问 http://localhost:25173，登录 admin / admin
```

---

## 参考文档

- [架构说明文档](./multi-channel-retrieval.md)
- [重构总结](./refactoring-summary.md)
