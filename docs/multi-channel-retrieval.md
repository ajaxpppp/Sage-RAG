# 多通道检索架构说明

## 概述

Ragent 的知识库检索采用「**多通道并行检索 + 后处理流水线**」架构，核心目标是在「召回率」与「准确率」之间取得平衡：

- **多种检索策略**：当前内置两条通道——意图定向检索（`IntentDirectedSearchChannel`）与向量全局检索（`VectorGlobalSearchChannel`）；接口层预留了 ES 关键词检索、混合检索等扩展位。
- **灵活的后处理**：检索结果统一进入一条后处理链，目前内置去重（`DeduplicationPostProcessor`）与重排序（`RerankPostProcessor`），可按需扩展版本过滤、分数归一化等。
- **底层向量库可切换**：通过 `rag.vector.type` 在 `pg`（默认，pgvector）与 `milvus` 之间切换，向量维度 1536，相似度度量 COSINE。
- **易于扩展**：新增检索通道或后处理器，只需实现对应接口并注册为 Spring Bean，引擎会自动发现并纳入执行，无需修改核心代码。

涉及的核心代码位于：
`bootstrap/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieve/`

```
retrieve/
├── RetrievalEngine.java                 # 检索总入口（KB 多通道 + MCP 工具编排）
├── MultiChannelRetrievalEngine.java     # 多通道引擎（并行通道 + 后处理链）
├── RetrieverService.java                # 底层向量检索接口
├── PgRetrieverService.java              # pgvector 实现（rag.vector.type=pg，默认）
├── MilvusRetrieverService.java          # Milvus 实现（rag.vector.type=milvus）
├── RetrieveRequest.java                 # 检索请求参数
├── channel/
│   ├── SearchChannel.java               # 检索通道接口
│   ├── SearchChannelType.java           # 通道类型枚举
│   ├── SearchChannelResult.java         # 单通道检索结果
│   ├── SearchContext.java               # 检索上下文
│   ├── VectorGlobalSearchChannel.java   # 向量全局检索通道
│   ├── IntentDirectedSearchChannel.java # 意图定向检索通道
│   ├── AbstractParallelRetriever.java   # 并行检索模板基类
│   └── strategy/
│       ├── CollectionParallelRetriever.java  # 按 collection 并行检索
│       └── IntentParallelRetriever.java      # 按意图节点并行检索
└── postprocessor/
    ├── SearchResultPostProcessor.java   # 后处理器接口
    ├── DeduplicationPostProcessor.java  # 去重处理器（order=1）
    └── RerankPostProcessor.java         # 重排序处理器（order=10）
```

## 架构设计

```
用户问题
    ↓
【问题重写 / 意图识别】（上游 StreamChatPipeline 完成，产出 SubQuestionIntent 列表）
    ↓
RetrievalEngine.retrieve(subIntents, topK)
    │  按子问题并行（ragContextExecutor），每个子问题再分流：
    ├─ KB 意图 → MultiChannelRetrievalEngine.retrieveKnowledgeChannels(...)
    └─ MCP 意图 → 抽参 + 调用 MCP Server（mcpBatchExecutor）
    ↓
MultiChannelRetrievalEngine
    │
    ├─【阶段1：多通道并行检索】（ragRetrievalExecutor）
    │      ├─→ IntentDirectedSearchChannel（意图定向，priority=1）
    │      └─→ VectorGlobalSearchChannel（向量全局兜底，priority=10）
    │      每个通道内部再按 collection / 意图节点并行（innerRetrievalExecutor）
    │
    └─【阶段2：后置处理器链】（按 order 串行）
           ├─→ DeduplicationPostProcessor（去重，order=1）
           └─→ RerankPostProcessor（qwen3-rerank 重排，order=10）
    ↓
ContextFormatter 格式化为 KB 上下文
    ↓
RAGPromptService 组装 Prompt → LLM 流式生成答案
```

## 核心组件

### 1. 检索通道（SearchChannel）

检索通道负责执行一种具体的检索策略。接口位于
`rag/core/retrieve/channel/SearchChannel.java`：

```java
public interface SearchChannel {
    String getName();                              // 通道名称（日志/监控）
    int getPriority();                             // 优先级，数字越小优先级越高
    boolean isEnabled(SearchContext context);      // 是否启用该通道
    SearchChannelResult search(SearchContext context); // 执行检索
    SearchChannelType getType();                   // 通道类型
}
```

通道类型枚举 `SearchChannelType` 当前包含：`VECTOR_GLOBAL`、`INTENT_DIRECTED`、`KEYWORD_ES`（预留）、`HYBRID`（预留）。

#### 已实现的通道

**IntentDirectedSearchChannel（意图定向检索，priority=1）**

- 启用条件（`isEnabled`）：配置 `intent-directed.enabled` 为 true，且当前子问题存在「KB 类型意图」（分数不低于 `min-intent-score`）。
- 检索逻辑：从意图识别结果中过滤出分数 ≥ `min-intent-score`（默认 0.4）的 KB 意图节点，针对每个意图节点对应的知识库 collection 并行检索（委托 `IntentParallelRetriever`）。
- TopK 计算：每个意图节点的基础 TopK 优先取意图节点自身配置的 `topK`，否则用 `fallbackTopK`；再乘以 `intent-directed.top-k-multiplier`（默认 2）放大召回，便于后续 Rerank 精筛。

**VectorGlobalSearchChannel（向量全局检索，priority=10）**

- 定位：**兜底通道**，确保即使意图识别失败或置信度偏低也能召回内容。
- 启用条件（`isEnabled`，满足任一即启用）：
  1. `vector-global.enabled` 为 true，且 `intent-directed.enabled` 为 false（意图定向关闭时全局检索必须兜底，否则无通道可用）；
  2. 未识别出任何意图；
  3. 所有意图的最高分 < `confidence-threshold`（默认 0.6）；
  4. 仅识别出 1 个意图且最高分 < `single-intent-supplement-threshold`（默认 0.8，作为单意图补充安全网）。
- 检索逻辑：从 `t_knowledge_base` 查询所有未删除知识库的 `collection_name`，对每个 collection 并行检索（委托 `CollectionParallelRetriever`）。
- TopK 计算：`context.topK * vector-global.top-k-multiplier`（默认倍率 3）。

> 注意：两条通道默认**可能同时启用**（意图定向命中 KB 意图、且置信度落在兜底区间时），结果在后处理阶段去重合并。

### 2. 检索上下文与结果

`SearchContext`（在通道间传递）核心字段：`originalQuestion`、`rewrittenQuestion`、`intents`（`List<SubQuestionIntent>`）、`topK`、`metadata`；`getMainQuestion()` 优先返回重写后的问题。

`SearchChannelResult`（单通道输出）字段：`channelType`、`channelName`、`chunks`（`List<RetrievedChunk>`）、`latencyMs`（耗时）、`metadata`。

`RetrievedChunk`（`framework.convention`）字段：`id`、`text`、`score`。

### 3. 后置处理器（SearchResultPostProcessor）

后处理器对合并后的检索结果做统一处理。接口位于
`rag/core/retrieve/postprocessor/SearchResultPostProcessor.java`：

```java
public interface SearchResultPostProcessor {
    String getName();                              // 处理器名称
    int getOrder();                                // 执行顺序，数字越小越先执行
    boolean isEnabled(SearchContext context);      // 是否启用
    List<RetrievedChunk> process(                  // 处理结果
        List<RetrievedChunk> chunks,               // 上一处理器的输出
        List<SearchChannelResult> results,         // 原始多通道结果（含元信息）
        SearchContext context
    );
}
```

#### 已实现的处理器

**DeduplicationPostProcessor（去重，order=1，始终启用）**

- 用 `LinkedHashMap` 保持顺序并去重；Chunk 唯一键取 `id`，为空时退化为 `text.hashCode()`。
- 合并策略：先按**通道类型优先级**排序处理结果（`INTENT_DIRECTED`=1 优先，`KEYWORD_ES`=2，`VECTOR_GLOBAL`=3，其余=99），从高优先级通道先填充；同一 Chunk 重复出现时**保留分数更高**的副本。

**RerankPostProcessor（重排序，order=10，最后执行）**

- 启用条件：`rag.rerank.enabled`（默认 true，由 `RAGConfigProperties.rerankEnabled` 读取）。
- 逻辑：Chunk 为空时直接跳过；否则调用 `RerankService.rerank(mainQuestion, chunks, topK)`，用 Rerank 模型对候选做精排并截断为最终 TopK。
- 模型路由：底层为 `infra-ai` 的 `RoutingRerankService`，候选模型 `qwen3-rerank`（bailian，priority 1）、`rerank-noop`（noop 兜底，priority 100）。

### 4. 多通道检索引擎（MultiChannelRetrievalEngine）

引擎通过构造注入 `List<SearchChannel>`、`List<SearchResultPostProcessor>`（Spring 自动收集所有 Bean）以及 `ragRetrievalExecutor` 线程池，无需手动注册——这正是「实现接口 + 注册 Bean 即生效」的根基。

入口方法（带 `@RagTraceNode` 链路追踪）：

```java
@RagTraceNode(name = "multi-channel-retrieval", type = "RETRIEVE_CHANNEL")
public List<RetrievedChunk> retrieveKnowledgeChannels(List<SubQuestionIntent> subIntents, int topK) {
    SearchContext context = buildSearchContext(subIntents, topK);   // 1. 构建上下文
    List<SearchChannelResult> channelResults = executeSearchChannels(context); // 2. 并行检索
    if (CollUtil.isEmpty(channelResults)) {
        return List.of();
    }
    return executePostProcessors(channelResults, context);          // 3. 后处理链
}
```

## 配置说明

检索相关配置位于 `bootstrap/src/main/resources/application.yaml`：

```yaml
rag:
  vector:
    type: pg              # 向量库类型，可选 pg（默认，pgvector）/ milvus

  default:
    collection-name: rag_default_store
    dimension: 1536       # 向量维度
    metric-type: COSINE   # 相似度度量

  query-rewrite:
    enabled: true

  # rerank.enabled 默认 true（未显式配置时由 RAGConfigProperties 取默认值）

  search:
    channels:
      vector-global:
        # enabled 默认 true
        confidence-threshold: 0.6   # 意图最高分低于此值时启用全局兜底检索
        top-k-multiplier: 3         # 全局检索召回放大倍数
        # single-intent-supplement-threshold 默认 0.8（单意图补充检索阈值）
      intent-directed:
        # enabled 默认 true
        min-intent-score: 0.4       # 低于此分数的意图节点被过滤
        top-k-multiplier: 2         # 意图检索召回放大倍数
```

配置映射类：

- `rag.search.*` → `SearchChannelProperties`（含 `defaultTopK=10`、`channels.vectorGlobal`、`channels.intentDirected`）。
- `rag.rerank.enabled`、`rag.query-rewrite.enabled` → `RAGConfigProperties`。
- `rag.default.*`（collection-name、dimension、metric-type）→ `RAGDefaultProperties`（Milvus 检索时使用）。

## 向量存储说明

底层向量检索抽象为 `RetrieverService` 接口，两套实现通过 `@ConditionalOnProperty` 按 `rag.vector.type` 二选一注入：

**PgRetrieverService（`rag.vector.type=pg`，默认）**

- 基于 PostgreSQL + pgvector 扩展，查询表 `t_knowledge_vector`。
- 流程：调用 `EmbeddingService.embed(query)` 得到向量 → L2 归一化 → 拼成 pgvector 字面量。
- SQL 使用余弦距离算子 `<=>`，分数为 `1 - (embedding <=> ?::vector)`，按距离升序取 TopK；按 `metadata->>'collection_name'` 过滤所属知识库；检索前执行 `SET hnsw.ef_search = 200` 提升召回率。

**MilvusRetrieverService（`rag.vector.type=milvus`）**

- 基于 Milvus（`io.milvus.v2` 客户端），`annsField=embedding`，`outputFields=[id, content, metadata]`。
- 流程：`embed` → 归一化 → `MilvusClientV2.search`；`metric_type` 取 `rag.default.metric-type`（COSINE），搜索参数 `ef=128`；collection 为空时回退到 `rag.default.collection-name`。

> 维度统一为 1536（`rag.default.dimension`，并被 `ai.embedding.candidates` 的 dimension 引用），度量统一为 COSINE，两套实现均在检索前对查询向量做归一化以匹配余弦语义。

## 工作流程

### 1. 多通道并行执行

`executeSearchChannels` 的执行步骤：

1. 过滤出 `isEnabled(context)` 为 true 的通道，并按 `getPriority()` 升序排序。
2. 对每个启用通道用 `CompletableFuture.supplyAsync(..., ragRetrievalExecutor)` 提交到 **RAG 检索线程池** 并行执行；单通道抛异常时降级为空结果（`emptyResult`），不影响其他通道。
3. `join()` 收集全部结果，并打印每通道命中数、耗时与汇总统计。

```
IntentDirectedSearchChannel ─┐
                             ├─→ 并行（ragRetrievalExecutor）
VectorGlobalSearchChannel   ─┘
```

而每个**通道内部**还存在第二层并行（由 `innerRetrievalExecutor` 驱动）：

- `IntentDirectedSearchChannel` 通过 `IntentParallelRetriever` 对多个意图节点对应的 collection 并行检索；
- `VectorGlobalSearchChannel` 通过 `CollectionParallelRetriever` 对全部知识库 collection 并行检索。

两者都继承自模板基类 `AbstractParallelRetriever`，统一封装「提交 Future → 收集结果 → 成功/失败统计」逻辑。

> 线程池在 `ThreadPoolExecutorConfig` 中定义并由 `TtlExecutors` 包装以透传用户上下文与 Trace：`ragRetrievalExecutor`（通道级并行）、`innerRetrievalExecutor`（通道内 collection/意图级并行）、`ragContextExecutor`（子问题级并行）、`mcpBatchExecutor`（MCP 工具并行）。

### 2. 后置处理器链执行

`executePostProcessors` 的执行步骤：

1. 过滤出启用的处理器并按 `getOrder()` 升序排序；若无启用处理器，则直接 flatMap 合并所有通道结果返回。
2. 将所有通道的 chunks 合并成初始列表，**按 order 串行**依次调用 `process(...)`，前一处理器的输出作为后一处理器的输入。
3. 单个处理器抛异常时记录日志并**跳过该处理器**，处理链继续，不中断整体流程；每步打印输入/输出数量及增减量。

```
原始 Chunks（多通道合并）
    ↓
DeduplicationPostProcessor (order=1)   去重，按通道优先级保留高分
    ↓
RerankPostProcessor (order=10)         qwen3-rerank 精排 + 截断 TopK
    ↓
最终 Chunks
```

## 扩展指南

得益于「Spring 自动收集 Bean + 引擎运行时过滤排序」的设计，扩展无需改动 `MultiChannelRetrievalEngine`。

### 1. 新增一个检索通道

实现 `SearchChannel` 接口并标注 `@Component` 即可被引擎自动纳入并行执行：

```java
@Component
public class KeywordESSearchChannel implements SearchChannel {

    @Override
    public String getName() {
        return "KeywordESSearch";
    }

    @Override
    public int getPriority() {
        return 5; // 介于意图定向(1)与全局(10)之间
    }

    @Override
    public boolean isEnabled(SearchContext context) {
        // 自定义启用条件，例如问题命中特定关键词时才启用
        String q = context.getMainQuestion();
        return q.contains("最新") || q.contains("实时");
    }

    @Override
    public SearchChannelResult search(SearchContext context) {
        long start = System.currentTimeMillis();
        List<RetrievedChunk> chunks = doEsSearch(context); // 实现 ES 检索
        return SearchChannelResult.builder()
                .channelType(getType())
                .channelName(getName())
                .chunks(chunks)
                .latencyMs(System.currentTimeMillis() - start)
                .build();
    }

    @Override
    public SearchChannelType getType() {
        return SearchChannelType.KEYWORD_ES;
    }
}
```

如需新增通道类型，在 `SearchChannelType` 枚举中补充枚举值，并在 `DeduplicationPostProcessor.getChannelPriority` 中为其分配去重优先级。

### 2. 新增一个后置处理器

实现 `SearchResultPostProcessor` 接口并标注 `@Component`，引擎会按 `getOrder()` 自动插入处理链对应位置：

```java
@Component
public class VersionFilterPostProcessor implements SearchResultPostProcessor {

    @Override
    public String getName() {
        return "VersionFilter";
    }

    @Override
    public int getOrder() {
        return 2; // 在去重(1)之后、Rerank(10)之前执行
    }

    @Override
    public boolean isEnabled(SearchContext context) {
        return true;
    }

    @Override
    public List<RetrievedChunk> process(List<RetrievedChunk> chunks,
                                        List<SearchChannelResult> results,
                                        SearchContext context) {
        // 例如：同一文档多个版本只保留最新版本
        return chunks.stream()
                .filter(this::isLatestVersion)
                .toList();
    }
}
```

### 3. 切换底层向量库

将 `rag.vector.type` 改为 `milvus` 即可在不改业务代码的前提下切换实现（`@ConditionalOnProperty` 控制 Bean 装配）；保持 `dimension=1536`、`metric-type=COSINE` 与向量库 schema 一致。

## 设计优势

1. **高召回**：意图定向 + 全局兜底双通道并行，意图识别失败或低置信时仍有全局检索保底。
2. **高准确**：意图定向提供精确召回，Rerank（qwen3-rerank）对合并候选统一精排。
3. **两级并行**：通道级（`ragRetrievalExecutor`）+ 通道内 collection/意图级（`innerRetrievalExecutor`），充分压缩检索延迟。
4. **强健壮性**：单通道或单处理器失败均被隔离降级，不影响整体链路；线程池经 `TtlExecutors` 包装透传上下文与 Trace。
5. **易扩展**：新增通道/处理器仅需实现接口 + `@Component`，引擎自动发现、过滤、排序。

## 注意事项

1. **通道优先级**：`getPriority()` 数字越小越优先，排序仅影响执行/日志顺序；真正影响去重保留策略的是 `DeduplicationPostProcessor` 内部按 `SearchChannelType` 定义的优先级。
2. **处理器顺序**：`getOrder()` 决定处理链顺序，去重应最先（order=1），Rerank 应最后（order=10）。
3. **召回放大与精排**：`top-k-multiplier` 放大召回是为给 Rerank 提供更充足的候选；最终结果数由 Rerank 截断为 `context.topK`。
4. **参数调优**：`confidence-threshold`、`single-intent-supplement-threshold`、`min-intent-score`、`top-k-multiplier` 需结合实际检索效果调整。
5. **向量一致性**：切换 `pg`/`milvus` 时需保证维度（1536）与度量（COSINE）和向量库一致，否则检索结果异常。

## 未来扩展

1. **ES 关键词检索通道**：基于 Elasticsearch 的全文/分词检索（`KEYWORD_ES` 类型已预留）。
2. **混合检索通道**：向量 + 关键词融合（`HYBRID` 类型已预留）。
3. **版本过滤处理器**：同一文档多版本只保留最新版本。
4. **分数归一化处理器**：统一不同通道的分数尺度后再去重/排序。
5. **检索缓存与监控**：缓存热点 Query 结果，并记录各通道命中率、耗时等指标。
