# Ragent 面试项目介绍策略与话术指南

## 一、核心原则：怎么让面试官对你的项目感兴趣

### 1.1 面试官在想什么

面试官听你介绍项目时，脑子里在评估三件事：
1. 这个项目有没有技术深度（不是 CRUD 套壳）
2. 你在里面做了什么（不是背别人的代码）
3. 你能不能讲清楚为什么这么做（不是只会 what，还要会 why）

记住一个原则：**话术里抛出的每个技术点，背后都要有能站得住脚的细节**（类名、配置数值、设计取舍）。下文每个亮点都配了「被追问时能答出的弹药」，面试前务必把这些数值记牢。

### 1.2 三个吸引面试官的关键策略

**策略一：开场制造差异化**

不要说"我做了一个问答系统"，要说"我做了一个企业级 Agentic RAG 智能体平台"。RAG、Agent、MCP 这些词本身就是面试官的兴趣触发器。几乎所有技术团队都在往 AI 方向靠，你一开口就聊 RAG + Agent + MCP，面试官的注意力立刻就来了。

**策略二：用问题驱动而不是功能罗列**

不要说"我做了检索、做了对话、做了管理后台"。要说"企业知识库问答有三个核心难题：检索不准、模型不稳、成本不可控，我分别用多通道检索 + Rerank、三态熔断器 + 模型路由、队列式分布式限流来解决"。

面试官听到的是：你理解问题 -> 你设计了方案 -> 你落地了实现。这比罗列功能有说服力得多。

**策略三：主动埋钩子，引导面试官追问你擅长的方向**

介绍时故意留一些"半句话"，让面试官忍不住追问。比如：
- "检索引擎用了多通道并行架构，**其中有一个基于置信度的动态兜底机制比较有意思**" -> 面试官大概率会问"什么动态兜底机制"
- "模型路由里有一个**首包探测**的设计" -> 面试官会问"首包探测是什么"
- "限流方案不是简单的令牌桶，是**基于 Redis ZSET 排队 + Pub/Sub 通知的队列式限流**" -> 面试官会问"为什么不用令牌桶"
- "我们还自建了一个 **MCP Server**，让大模型能调销售、工单、天气这类外部业务工具" -> 面试官会问"MCP 怎么落地的"

---

## 二、30 秒版本（电梯演讲）

适用场景：面试官说"简单介绍一下你的项目"

> "我做的是一个企业级 Agentic RAG 智能体平台，叫 Ragent，RAG + Agent。核心是把企业私有知识库、大模型和外部业务工具结合起来，做精准的智能问答。
>
> 技术栈是 Java 17 + Spring Boot 3.5 + React 18，Maven 多模块（framework / infra-ai / bootstrap / mcp-server），向量存储默认用 PostgreSQL 的 pgvector 扩展，也能一键切到 Milvus。我主要负责了几个核心模块：多通道并行检索引擎、模型路由与三态熔断器、基于 Redis 的队列式分布式限流、意图树识别 + 歧义引导，以及全链路 Trace。
>
> 整个 RAG 链路从查询改写、意图识别、多路检索、Rerank 精排到流式生成，都是我参与设计和实现的。"

要点：
- 30 秒内说完
- 包含项目名称、定位、技术栈、你的核心职责
- 每个职责都用了技术关键词，给面试官留追问空间
- **不要说错技术栈**：业务库是 PostgreSQL（pgvector），不是 MySQL；向量存储默认就是 pg，Milvus 是可选项

---

## 三、2 分钟版本（详细介绍）

适用场景：面试官说"详细介绍一下"

> "Ragent 是一个企业级 Agentic RAG 智能体平台，解决的核心问题是：企业内部知识库问答场景中，传统关键词搜索理解不了用户意图，直接用大模型又有幻觉问题。我们通过 RAG 架构把私有知识和大模型能力结合起来，再用 MCP 协议接外部业务工具，让它能"查"也能"做"。
>
> 技术栈方面，后端是 Java 17 + Spring Boot 3.5，Maven 多模块拆成 framework（通用框架）、infra-ai（模型/检索基础设施）、bootstrap（业务）、mcp-server（工具服务）。前端 React 18 + TypeScript + Vite。业务库用 PostgreSQL，向量存储默认走 pgvector 扩展、维度 1536、COSINE 距离，配置一改就能切到 Milvus。Redis + Redisson 做缓存和分布式锁，RocketMQ 做异步，RustFS（S3 兼容）存文件。
>
> 整个系统的核心链路在 RAGChatServiceImpl.streamChat 里，由 StreamChatPipeline 编排：用户提问后，先加载会话记忆（最近 4 轮 + 摘要）做上下文补全；然后 QueryRewriteService 做查询改写和子问题拆分，解决指代消解；接着 IntentResolver 把子问题映射到意图树节点；如果意图有歧义，IntentGuidanceService 会直接返回澄清话术让用户二选一。识别完之后走多通道并行检索——一个通道按意图定向检索，一个通道做全局向量兜底——结果经过去重和 Rerank 精排取 Top K。如果命中 MCP 类意图，就让 LLM 按 schema 抽参，HTTP 调我们自建的 MCP Server。最后 RAGPromptService 组装 Prompt 发给大模型，通过 SSE 流式返回，前端边收边渲染。
>
> 我重点负责了几个模块：
>
> 第一个是多通道检索引擎。核心设计是把检索策略抽象成 SearchChannel 接口，MultiChannelRetrievalEngine 用 CompletableFuture 并行执行各通道，后面接 SearchResultPostProcessor 责任链做去重和 Rerank。全局向量通道的置信度阈值是 0.6、top-k 放大 3 倍，意图定向通道最小意图分 0.4、放大 2 倍，都在配置里。
>
> 第二个是模型路由和熔断。生产环境不能只依赖一个模型厂商，我用 ModelSelector 按 priority 选模型、ModelRoutingExecutor 负责路由和故障转移、ModelHealthStore 维护每个模型的健康状态，做了一个三态熔断器：CLOSED -> OPEN -> HALF_OPEN，失败 2 次熔断、熔断 30 秒后进半开试探。还有一个首包探测机制，切换模型时先缓冲第一个响应包确认正常再输出，用户完全无感知。
>
> 第三个是队列式分布式限流。大模型调用成本高、耗时长，需要控制的是"同时在处理的并发数"而不是 QPS。我用 Redis 信号量控并发、ZSET 做等待队列、Lua 脚本保证原子性、Pub/Sub 做跨节点唤醒，默认单节点最大并发 10、最长等待 15 秒、租约 30 秒。用户排队时通过 SSE 实时推送位置。
>
> 第四个是会话记忆 + 全链路 Trace。记忆用滑动窗口保留最近 4 轮，从第 5 轮起自动调 LLM 生成摘要缓存到库里，既控 Token 又不丢上下文。Trace 这块我用自定义注解 @RagTraceRoot/@RagTraceNode 把整条链路的每个节点耗时、入参出参落库，问题可回溯。"

---

## 四、面试官追问应对策略（高频问题）

### 4.1 项目整体类

**Q: 为什么选择自己实现 RAG 而不是用 LangChain4j？**

> "LangChain4j 更适合快速原型，但企业场景需要更细粒度的控制。比如我们的多通道检索需要根据意图置信度动态决定走哪些通道，后处理链需要自定义去重和 Rerank 逻辑，意图树识别 + 歧义引导也是业务强相关的，这些用 LangChain4j 的抽象反而会受限。而且 LangChain4j 版本迭代很快，低版本功能缺失，高版本升级约等于重写，自己实现核心链路可控性更强。我们只在底层用了 OkHttp 直连各家 OpenAI 风格的接口。"

**Q: 为什么用 PostgreSQL + pgvector 而不是专门的向量数据库？**

> "默认用 pgvector 是因为对中小规模企业知识库来说，业务数据和向量放在同一个 PostgreSQL 里，能少维护一套中间件、事务和一致性也更简单。但我们没有把自己锁死——向量存储抽象成了 RetrieverService 接口，配置项 rag.vector.type 可以在 pg 和 milvus 之间一键切换，对应 PgRetrieverService 和 MilvusRetrieverService 两个实现。数据量上来了切 Milvus，上层检索逻辑完全不用动。维度统一 1536、距离用 COSINE。"

**Q: 项目的并发量大概是多少？怎么做的压测？**

> "这是一个面向企业内部的知识库问答系统，并发量不会特别大，但大模型调用本身响应慢、成本高，所以瓶颈不在 QPS 而在并发控制。我们通过队列式限流控制同时调用大模型的请求数，默认配置是单节点最大 10 个并发、最长排队 15 秒。压测方面主要关注检索延迟和端到端响应时间，多通道并行检索因为是 CompletableFuture 同时跑，整体延迟取决于最慢的那个通道而不是相加。"

**Q: 你们团队几个人？你负责哪些部分？**

> "核心开发 3 个人，我主要负责后端的 RAG 核心链路，包括检索引擎、模型路由熔断、限流、会话记忆和全链路 Trace。前端是另一个同学负责的，我会看前端代码但不是主要开发者。文档摄取流水线（Ingestion，Fetcher -> Parser -> Chunker -> Enhancer -> Enricher -> Indexer 这套节点编排）是一起设计的，我负责了其中的分块策略和向量化部分。"

### 4.2 检索相关

**Q: 为什么不用 Elasticsearch 做混合检索？**

> "我们的向量检索目前是 pgvector / Milvus 两套可切换的实现。ES 的优势在全文检索和 BM25，如果后续要加关键词检索通道，架构上是完全支持的——只要再实现一个 SearchChannel（比如 EsKeywordSearchChannel）注册成 Bean，MultiChannelRetrievalEngine 就会自动把它并进多路检索里，不用改任何已有代码。所以混合检索是个扩展点，不是没考虑。"

**Q: Rerank 的原理是什么？为什么需要 Rerank？**

> "向量检索是基于 Embedding 的近似最近邻搜索，召回的是语义相似的文档，但相似不等于相关。Rerank 模型是交叉编码器，把 query 和每个候选文档拼在一起做精细的相关性打分，比双塔 Embedding 相似度更准。我们 Rerank 走的是百炼的 qwen3-rerank，在 RerankPostProcessor 里作为后处理链的一环，对去重后的候选重新排序取 Top K。Rerank 是可开关的，配了一个 rerank-noop 兜底，关掉时直接跳过精排。代价是 Rerank 慢，所以放在向量召回之后只对少量候选精排。"

**Q: 意图识别置信度低于 0.6 时触发全局检索，这个 0.6 怎么定的？**

> "全局向量通道的 confidence-threshold 配的就是 0.6，意图定向通道的 min-intent-score 是 0.4，这些是通过实验调出来的平衡点——阈值太高会导致大量请求走全局检索增加延迟，太低会导致意图识别错误时没有兜底。值都在 application.yaml 的 rag.search.channels 里，可以动态调整。"

### 4.3 模型路由与熔断

**Q: 熔断器为什么不用 Resilience4j 或 Sentinel？**

> "Resilience4j 和 Sentinel 是通用熔断限流框架，但我们的场景有特殊需求：模型调用是流式的，需要首包探测——切换到新模型时要先缓冲第一个响应包确认正常再输出，这和通用熔断器的 fallback 机制不一样。而且我们的熔断粒度是模型级别的，每个模型在 ModelHealthStore 里独立维护状态，三态切换很简单：失败累计到 failure-threshold=2 就 OPEN，open-duration 30 秒后进 HALF_OPEN 试探，探测成功回 CLOSED。核心就是一个 ConcurrentHashMap + 计数器，自己实现更贴合流式场景。"

**Q: 首包探测具体怎么实现的？**

> "核心是 ProbeStreamBridge 配合 ForwardingStreamCallback 这套装饰器，包在原始 StreamCallback 外面。RoutingLLMService 切到一个候选模型时，会先拦住第一个 token 事件缓冲起来不发给用户；如果后续正常收到更多 token，说明模型可用，就把缓冲的内容一起放出去；如果首包就超时或报错，就在 ModelHealthStore 里把这个模型标记为不健康，按 priority 切到下一个候选重试。整个过程对用户端是无感知的，他只会看到正常的流式输出。"

**Q: 你们都接了哪些模型？默认用哪个？**

> "供应商有四家：ollama（本地）、bailian（阿里百炼）、aihubmix、siliconflow。Chat 模型按 priority 排了一串候选：qwen-plus、本地 qwen3-local、qwen3-max、glm-4.7、gpt-5.4。默认模型和深度思考模型都是 qwen3-max，它支持思考模式。Embedding 默认是 siliconflow 的 Qwen3-Embedding-8B，维度 1536。这些全在 application.yaml 的 ai.* 配置里，RAGSettingsController 还能在后台查到当前生效的配置。"

### 4.4 限流相关

**Q: 为什么不用令牌桶或滑动窗口限流？**

> "令牌桶和滑动窗口适合控制 QPS，但我们要控制的不是每秒请求数，而是同时在处理的请求数。大模型一次调用可能 10-30 秒，用令牌桶的话令牌很快发完，后面的请求直接被拒。我们需要排队机制——请求进来排队等，前面处理完后面自动顶上，还要告诉用户排在第几位。所以用 Redis 信号量控并发（max-concurrent=10）、ZSET 按时间戳排队、Lua 脚本原子判断能否进入、Pub/Sub 跨节点通知。最长等 15 秒（max-wait-seconds），拿到名额后租约 30 秒（lease-seconds）防止死锁，轮询间隔 200ms。"

**Q: Pub/Sub 通知会不会有惊群效应？**

> "会有，所以我们做了本地合并通知。收到 Pub/Sub 消息后不是每个等待线程都去抢，而是本地统一处理，用 Lua 原子判断 ZSET 队首是不是自己、信号量有没有空位，是才唤醒对应线程。这样避免了所有节点的所有等待线程同时去 Redis 查询。"

### 4.5 记忆管理

**Q: 会话记忆是怎么压缩的？会不会丢失关键信息？**

> "策略在 rag.memory 里配置：滑动窗口保留最近 4 轮完整对话（history-keep-turns=4），从第 5 轮起（summary-start-turns=5）触发摘要，调 LLM 把更早的历史压成不超过 200 字的摘要（summary-max-chars=200）缓存到 t_conversation_summary 表。摘要 Prompt 明确要求保留关键实体、数字、结论，尽量只压冗余的对话过程。会有信息损失，这是 trade-off，但用户关注的信息通常在最近几轮里，实测对回答质量影响不大。会话标题也是自动生成的，限长 30 字。"

**Q: 如果用户问的是很早之前聊过的内容怎么办？**

> "这确实是边界场景。目前靠摘要保留关键信息覆盖大部分情况。要做得更好，可以对历史消息也做向量化，提问时不仅检索知识库也检索历史对话——其实我们的检索通道是接口化的，加一个"历史对话检索通道"在架构上是顺手的事。这作为后续优化方向已经在规划中。"

### 4.6 MCP / Agent 能力

**Q: 你说的 MCP 是什么？怎么落地的？**

> "MCP 是模型上下文协议，让大模型能调外部工具。我们单独拆了一个 mcp-server 模块，独立跑在 29099 端口，里面有 SalesMcpExecutor、TicketMcpExecutor、WeatherMcpExecutor 几个工具执行器，通过 JSON-RPC 端点接收并路由。bootstrap 这边在 rag.core.mcp 下有 McpToolRegistry、McpToolExecutor、McpClientToolExecutor 作为 MCP 客户端。整个链路是：意图识别命中 MCP 类意图 -> LLM 按工具 schema 抽取参数 -> HTTP 调 mcp-server 执行 -> 把工具结果作为上下文拼进 Prompt。这就是项目里"Agent"的体现，模型不只是查知识库，还能触发业务动作。"

### 4.7 工程能力类

**Q: 异步线程多了上下文怎么传？TTL 是怎么做的？**

> "用的阿里开源的 TransmittableThreadLocal（TTL），版本 2.14.5。普通 ThreadLocal 在线程池场景下会丢，因为线程是复用的。TTL 通过包装线程池的 execute/submit，在任务提交时捕获当前线程的 ThreadLocal 值、执行时恢复。我们在 framework 里统一用 TtlExecutors 包装所有线程池，确保用户上下文（UserContext 里的 userId、role）和链路追踪上下文（RagTraceContext 的 traceId）在多通道检索、并发意图分类这些异步场景里不丢失。这是把全链路 Trace 做完整的前提。"

**Q: 全链路 Trace 是怎么实现的？**

> "framework 里定义了 RagTraceContext 和两个注解 @RagTraceRoot / @RagTraceNode，配合 AOP 切面。入口方法标 @RagTraceRoot 开一个 trace run，链路上每个关键节点（改写、意图、各检索通道、Rerank、LLM 调用）标 @RagTraceNode，切面自动记录每个节点的耗时、入参出参、异常（错误信息最长截断 1000 字）。数据落到 t_rag_trace_run 和 t_rag_trace_node 两张表，后台 RagTraceController 能查列表和详情，一次问答出了问题可以逐节点回溯。这个之所以能跨异步线程串起来，就是靠前面说的 TtlExecutors 透传 traceId。"

**Q: 框架层做了哪些通用能力？**

> "framework 模块沉淀了不少：统一返回 Result / 全局异常处理、三级异常体系 + 错误码枚举、SSE 封装（SseEmitterSender）、基于 Redis + Lua 的幂等注解 @IdempotentSubmit、Snowflake 分布式 ID（用 Lua 初始化 workerId）、用户上下文、缓存、MQ 封装，以及刚说的 Trace 和 TtlExecutors 线程池。业务模块都依赖这一层，保证横切关注点统一。"

---

## 五、面试节奏控制技巧

### 5.1 开场 30 秒定基调

前 30 秒决定面试官对你项目的第一印象。要做到：
- 说出项目名称和一句话定位（Ragent = RAG + Agent，企业级 Agentic RAG 平台）
- 点出 2-3 个技术关键词（多通道检索、三态熔断、MCP）
- 明确你的核心职责
- **别把技术栈说错**：PostgreSQL + pgvector（默认），不是 MySQL

### 5.2 中间用"问题-方案"结构

每讲一个模块，先说遇到了什么问题，再说你的方案。这比直接说"我用了 XXX 技术"有说服力得多。

示例：
- 不好的："我用了 Redis ZSET 做限流"
- 好的："大模型调用成本高、响应慢，传统令牌桶控制的是 QPS 不适合这个场景，所以我设计了基于 Redis 信号量 + ZSET 排队的队列式限流，单节点并发上限 10，支持排队等待和 SSE 实时位置推送"

### 5.3 STAR 式表达示例（讲熔断这个亮点）

- **S（情境）**："生产环境我们接了百炼、siliconflow、aihubmix 等多家模型，单家厂商偶发超时或限流是常态。"
- **T（任务）**："要保证某家模型抖动时对话不中断，还不能让用户看到切换痕迹。"
- **A（行动）**："我设计了模型级三态熔断器，ModelHealthStore 维护每个模型的状态，失败 2 次熔断、30 秒后半开试探；并做了 ProbeStreamBridge 首包探测，切换新模型时先缓冲首包确认可用再输出。"
- **R（结果）**："单家模型故障时能在首包阶段就自动切换到下一个候选，用户侧无感知，对话成功率明显提升。"

### 5.4 主动引导追问方向

如果你对某个模块特别熟，就在介绍时多留"钩子"：
- "这里有一个比较巧妙的设计是..."（然后不展开）
- "这个方案还解决了一个额外的问题..."（然后一笔带过）

面试官大概率会追问，而你已经准备好了答案。

### 5.5 不会的问题怎么办

如果被问到不熟悉的细节，诚实说"这部分我了解大致原理但没深入实现"，然后把话题引到你熟悉的方向：

> "pgvector 的 HNSW 索引参数调优我没深入研究，但在检索层面我更关注多通道的编排策略和后处理链设计，这部分我可以详细聊聊。"

---

## 六、面试官可能的"陷阱"问题

### 6.1 "这个项目是你自己写的还是开源项目？"

> "这是基于一个开源项目的架构做的二次开发和深度定制。核心的 RAG 链路、检索引擎、模型路由熔断、限流方案、全链路 Trace 都是我自己设计和实现的。我选这个项目是因为它的多模块架构设计合理，但很多核心模块要根据实际业务做深度定制，这个过程让我对 Agentic RAG 系统有了很深的理解。"

### 6.2 "你说的这些设计模式是不是过度设计？"

> "每个抽象都对应一个具体的扩展需求。比如 SearchChannel 接口不是为了用策略模式而用，是因为我们确实需要动态增减检索通道——后续加 ES 关键词通道时，只要实现接口注册成 Bean，MultiChannelRetrievalEngine 就自动并进来，不用改已有代码。RetrieverService 抽象也是同理，pg 和 Milvus 可切换。如果只有一种检索方式，我不会抽象接口。"

### 6.3 "向量数据库挂了怎么办？"

> "默认情况下向量数据就在 PostgreSQL 里，跟着业务库一起做高可用，少了一套独立中间件的故障面。如果切到 Milvus 且它完全不可用，检索引擎会捕获异常返回空结果，链路会退回纯系统回答并提示用户"当前知识库暂时不可用"，不会让整个服务挂掉。而且这一步的异常会被 Trace 记下来，方便排查。"

### 6.4 "数据库为什么不用 MySQL？"

> "业务库选的是 PostgreSQL，最主要的原因是 pgvector 扩展——可以直接在关系库里存向量、做近似最近邻检索，业务数据和向量同库，省一套中间件、事务一致性也更好。项目早期确实有过 MySQL 的脚本，后来统一迁到了 PostgreSQL，旧脚本归档了。建表脚本是 schema_pg.sql，pgvector 扩展由 docker 的 01-init-pgvector.sql 初始化。"

---

## 七、一页纸速查表（面试前 5 分钟看）

| 维度 | 关键词 / 数值 |
|------|--------|
| 项目名 | Ragent = RAG + Agent，企业级 Agentic RAG 智能体平台 |
| 构建 | Maven 多模块：framework / infra-ai / bootstrap / mcp-server |
| 技术栈 | Java 17 + Spring Boot 3.5 + React 18 + PostgreSQL(pgvector，默认) + Redis/Redisson + RocketMQ + RustFS |
| 端口 | bootstrap 29090（/api/ragent）、mcp-server 29099、前端 25173 |
| 向量存储 | rag.vector.type=pg（默认）/ milvus 可切，dim=1536，COSINE |
| 供应商 | ollama / bailian / aihubmix / siliconflow（默认模型 qwen3-max） |
| 职责1 | 多通道检索：SearchChannel 接口 + CompletableFuture 并行 + 去重 + Rerank(qwen3-rerank)，阈值 0.6/0.4 |
| 职责2 | 模型路由熔断：ModelSelector/ModelRoutingExecutor/ModelHealthStore，三态熔断 failure-threshold=2 / open-duration 30s + 首包探测(ProbeStreamBridge) |
| 职责3 | 队列式限流：Redis 信号量 + ZSET 排队 + Lua + Pub/Sub，max-concurrent=10 / wait 15s / lease 30s + SSE 位置推送 |
| 职责4 | 会话记忆：保留最近 4 轮 + 第 5 轮起 LLM 摘要(≤200字)缓存入库 |
| 职责5 | 全链路 Trace：@RagTraceRoot/@RagTraceNode + AOP，落 t_rag_trace_run/node，靠 TtlExecutors 跨线程透传 traceId |
| Agent | 自建 mcp-server(29099)：Sales/Ticket/Weather 工具，JSON-RPC，LLM 按 schema 抽参调用 |
| RAG 链路 | 记忆加载 -> 查询改写/拆分 -> 意图树识别 -> 歧义引导 -> 多通道检索(意图定向+全局向量兜底) -> 去重/Rerank -> Prompt 组装 -> SSE 流式生成 -> 可反馈(t_message_feedback) |
| 核心库表 | t_message（消息）、t_intent_node（意图树）、t_conversation_summary、t_rag_trace_run/node 等共 21 张 |
| 为什么 pgvector | 业务数据与向量同库、少维护中间件、事务一致；接口化可切 Milvus |
| 为什么不用令牌桶 | 控制的是并发数不是 QPS，需要排队 + 位置推送 |
| 为什么不用通用熔断框架 | 流式 + 模型级粒度 + 首包探测，自实现更贴场景 |
| 上下文透传 | TransmittableThreadLocal + TtlExecutors 包装所有线程池 |
| 设计模式 | 策略(SearchChannel)、责任链(PostProcessor)、装饰器(首包探测)、模板方法(AbstractOpenAIStyleChatClient)、注册表(McpToolRegistry)、观察者(Pub/Sub)——每个都解决实际问题 |
