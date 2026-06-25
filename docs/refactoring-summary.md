# 重构总结

> 本文档描述 Ragent（RAG + Agent）平台中「多通道检索可扩展架构」的重构。
> 相关代码位于 `bootstrap/.../rag/core/retrieve/`，配置项前缀为 `rag.search`。

## 重构目标

将原本单一、强耦合的检索逻辑，重构为一套**可扩展的多通道检索架构**，在保持高精准度的同时显著提升召回率，并让后续新增检索策略 / 后处理逻辑无需改动核心代码。

具体要解决以下问题：

1. **意图识别覆盖率不足**：意图树（`t_intent_node`）依赖示例问题驱动分类，当知识库文档增多、用户问法多样时，意图识别难以覆盖所有场景，导致命中不到正确意图节点。
2. **缺乏兜底机制**：一旦意图识别失败或置信度偏低，旧逻辑只能返回空召回，相关文档检索不到，回答质量下降。
3. **扩展性差**：检索策略（向量、关键词等）与重排序、去重等后处理逻辑混在一起，新增一种检索方式或一道后处理工序，都要改动核心检索流程。

## 重构内容

### 包结构

重构后 `rag/core/retrieve/` 划分为三层：

- `retrieve/`（引擎层）：`RetrievalEngine`、`MultiChannelRetrievalEngine`、`RetrieverService`（`PgRetrieverService` / `MilvusRetrieverService`）。
- `retrieve/channel/`（检索通道层）：`SearchChannel` 接口、`SearchChannelType` 枚举、`SearchContext`、`SearchChannelResult`，以及实现 `VectorGlobalSearchChannel`、`IntentDirectedSearchChannel`；并行检索模板基类 `AbstractParallelRetriever`。
- `retrieve/channel/strategy/`（并行检索策略）：`CollectionParallelRetriever`（按 collection 并行）、`IntentParallelRetriever`（按意图节点并行）。
- `retrieve/postprocessor/`（后处理层）：`SearchResultPostProcessor` 接口，以及实现 `DeduplicationPostProcessor`、`RerankPostProcessor`。

### 核心抽象

#### 检索通道 `SearchChannel`

每个通道封装一种检索策略，可并行执行。接口方法：

- `String getName()`：通道名称（日志 / 监控）。
- `int getPriority()`：通道优先级，**数字越小优先级越高**，用于去重时决定保留哪个通道的结果。
- `boolean isEnabled(SearchContext context)`：根据上下文动态决定是否启用本通道（条件触发的关键）。
- `SearchChannelResult search(SearchContext context)`：执行检索。
- `SearchChannelType getType()`：通道类型枚举（`VECTOR_GLOBAL` / `INTENT_DIRECTED` / `KEYWORD_ES` / `HYBRID`）。

已实现的通道：

- **`IntentDirectedSearchChannel`**（`getName()="IntentDirectedSearch"`，`priority=1`，最高优先级）：基于意图识别结果，对命中的意图节点所属知识库做定向检索，是最精确的检索方式。仅当存在分数不低于 `min-intent-score` 的 KB 意图时启用，内部使用 `IntentParallelRetriever` 按意图节点并行检索，`topK = context.topK * topKMultiplier`。
- **`VectorGlobalSearchChannel`**（`getName()="VectorGlobalSearch"`，`priority=10`，较低优先级）：作为**兜底**，在全部启用的知识库 collection（从 `t_knowledge_base` 读取）中做向量检索。**条件触发**——满足以下任一条件即启用：意图定向通道被关闭（必须兜底）、未识别出任何意图、意图最高分低于 `confidence-threshold`、或仅有单一意图且分数低于 `single-intent-supplement-threshold`（补充安全网）。内部使用 `CollectionParallelRetriever` 跨 collection 并行检索，召回数量为 `context.topK * topKMultiplier`。

#### 检索上下文 `SearchContext`

在各通道之间传递的统一载体，携带 `originalQuestion`、`rewrittenQuestion`、`subQuestions`、`intents`（`SubQuestionIntent` 列表）、`topK`、`metadata`，并提供 `getMainQuestion()`（优先返回重写后的问题）。

#### 后置处理器 `SearchResultPostProcessor`

对多通道合并后的结果做统一后处理，按 `getOrder()` 升序串成处理链。接口方法：`getName()`、`int getOrder()`（数字越小越先执行）、`boolean isEnabled(SearchContext)`、`List<RetrievedChunk> process(chunks, results, context)`。

已实现的处理器：

- **`DeduplicationPostProcessor`**（`getName()="Deduplication"`，`order=1`，最先执行，始终启用）：合并各通道结果并按 Chunk 主键去重；同一 Chunk 出现在多个通道时，**按通道优先级保留**（`INTENT_DIRECTED` > `KEYWORD_ES` > `VECTOR_GLOBAL`），并在重复命中时取较高分数，使用 `LinkedHashMap` 保持顺序。
- **`RerankPostProcessor`**（`getName()="Rerank"`，`order=10`，最后执行）：调用 `RerankService` 对去重后的候选做重排序并截取最终 Top-K。是否启用由 `RAGConfigProperties#getRerankEnabled()`（即 `rag.rerank-enabled` 开关）决定。底层 Rerank 模型为 **`qwen3-rerank`**（阿里百炼 bailian，优先级 p1；另有 `rerank-noop` 兜底 p100）。

### 引擎层

#### `MultiChannelRetrievalEngine`

新增的协调引擎（`@Service`），通过 Spring 注入 `List<SearchChannel>` 与 `List<SearchResultPostProcessor>`，并使用线程池 `ragRetrievalExecutor`。核心方法 `retrieveKnowledgeChannels(subIntents, topK)`（标注 `@RagTraceNode(type="RETRIEVE_CHANNEL")`）流程：

1. `buildSearchContext` 构建 `SearchContext`。
2. **阶段 1 多通道并行检索**：过滤 `isEnabled` 为真的通道、按 `priority` 排序，用 `CompletableFuture` 提交到 `ragRetrievalExecutor` 并行执行；单个通道异常被捕获并降级为空结果，不影响其他通道；汇总成功 / 无结果 / Chunk 总数统计日志。
3. **阶段 2 后置处理器链**：过滤启用的处理器、按 `order` 排序，把各通道结果合并后依次 `process`；某个处理器抛异常时跳过该处理器、不中断整条链。

#### `RetrievalEngine`

业务入口（`@RagTraceNode(type="RETRIEVE")`），被改造为**集成 `MultiChannelRetrievalEngine`**。它按子问题（`SubQuestionIntent`）并行构建上下文：KB 部分调用 `multiChannelRetrievalEngine.retrieveKnowledgeChannels` 获取 Chunk，MCP 部分对 MCP 意图用 LLM 抽参后调用 MCP 工具；最终把 KB 上下文、MCP 上下文与按意图分组的 Chunk 组装成 `RetrievalContext`，交给 `RAGPromptService` 拼装 Prompt。

#### 底层检索 `RetrieverService`

向量存储**可切换**，由 `rag.vector.type` 决定：

- `PgRetrieverService`：`@ConditionalOnProperty(name="rag.vector.type", havingValue="pg")`，基于 **PostgreSQL + pgvector**，为默认配置（`application.yaml` 中 `rag.vector.type: pg`）。
- `MilvusRetrieverService`：`@ConditionalOnProperty(name="rag.vector.type", havingValue="milvus")`，基于 Milvus（可选）。

向量维度 `dimension=1536`，相似度 `metric=COSINE`。

## 架构设计

### 核心思想

采用**策略模式 + 责任链模式**的组合：

- **策略模式**：每个 `SearchChannel` 实现一种检索策略，互相独立、可并行。
- **责任链模式**：`SearchResultPostProcessor` 按 `order` 串成处理链，依次加工检索结果。

### 工作流程

```
子问题 + 意图识别结果（SubQuestionIntent）
        ↓
   构建 SearchContext
        ↓
【阶段 1：多通道并行检索（ragRetrievalExecutor）】
   ├─→ IntentDirectedSearchChannel（存在 KB 意图时启用，priority=1）
   └─→ VectorGlobalSearchChannel（条件触发兜底，priority=10）
        ↓ 合并所有通道结果
【阶段 2：后置处理器链（按 order 升序）】
   ├─→ DeduplicationPostProcessor（order=1，按通道优先级去重）
   └─→ RerankPostProcessor（order=10，qwen3-rerank 重排 + 截取 Top-K）
        ↓
   返回最终 RetrievedChunk 列表
```

### 关键特性

1. **条件触发的双路召回**：意图识别成功且置信度足够时，只走意图定向检索（精准）；意图缺失 / 置信度低 / 单一中等置信度意图时，自动叠加全局向量检索兜底（高召回）。
2. **真正的并行执行**：启用的通道通过 `CompletableFuture` 提交到 `ragRetrievalExecutor` 并行执行；通道内部还有 `AbstractParallelRetriever` 模板（`CollectionParallelRetriever` / `IntentParallelRetriever`）对多 collection / 多意图节点二级并行。
3. **容错隔离**：单通道或单处理器异常不会拖垮整条链，分别降级为空结果 / 跳过该处理器。
4. **统一后处理**：先按通道优先级去重合并，再用 `qwen3-rerank` 重排截断，保证多路结果分数尺度被统一裁决。
5. **零改动扩展**：新增检索策略只需实现 `SearchChannel` 并交给 Spring 管理；新增后处理只需实现 `SearchResultPostProcessor`。引擎自动发现、排序并编排，无需改动 `MultiChannelRetrievalEngine` 或 `RetrievalEngine`。

## 解决方案对比

| 方案 | 召回率 | 精准度 | 性能 | 扩展性 |
|------|--------|--------|------|--------|
| **原方案**（仅意图定向检索） | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ |
| **新方案**（多通道检索 + 后处理流水线） | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |

### 新方案优势

1. **召回率提升**：全局向量检索兜底，意图识别失败 / 低置信度时仍能召回相关文档，直接缓解「意图识别覆盖率不足」。
2. **精准度保持**：意图定向检索优先级最高，去重时优先保留其结果，再经 `qwen3-rerank` 重排，保证排在前面的仍是最相关片段。
3. **性能可控**：兜底通道条件触发，只在必要时启用；多通道、多目标均并行，额外延迟有限。
4. **扩展性强**：新增通道或处理器无需修改核心代码（Spring 自动注入 + 排序编排）。

## 使用示例

### 1. 基本使用

```java
@Service
@RequiredArgsConstructor
public class SomeRagService {

    private final MultiChannelRetrievalEngine multiChannelRetrievalEngine;

    public List<RetrievedChunk> search(List<SubQuestionIntent> subIntents) {
        // 引擎内部完成「多通道并行检索 → 去重 → Rerank」
        return multiChannelRetrievalEngine.retrieveKnowledgeChannels(subIntents, 5);
    }
}
```

### 2. 新增检索通道（零改动扩展）

```java
@Component
public class KeywordESSearchChannel implements SearchChannel {

    @Override
    public String getName() {
        return "KeywordESSearch";
    }

    @Override
    public int getPriority() {
        return 5; // 介于意图定向(1)与全局向量(10)之间
    }

    @Override
    public boolean isEnabled(SearchContext context) {
        return containsKeywords(context.getMainQuestion());
    }

    @Override
    public SearchChannelResult search(SearchContext context) {
        // 实现 ES 检索逻辑
        return SearchChannelResult.builder()
                .channelType(SearchChannelType.KEYWORD_ES)
                .channelName(getName())
                .chunks(/* ... */)
                .build();
    }

    @Override
    public SearchChannelType getType() {
        return SearchChannelType.KEYWORD_ES;
    }
}
```

### 3. 新增后置处理器（零改动扩展）

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
        // 实现版本过滤逻辑
        return chunks;
    }
}
```

## 配置说明

检索行为由前缀 `rag.search` 的配置驱动（映射到 `SearchChannelProperties`）。`application.yaml` 实际配置如下：

```yaml
rag:
  search:
    channels:
      vector-global:
        confidence-threshold: 0.6   # 意图最高分低于此值时启用全局检索
        top-k-multiplier: 3
      intent-directed:
        min-intent-score: 0.4       # 低于此分的意图节点被过滤
        top-k-multiplier: 2
```

代码侧默认值（`SearchChannelProperties`）：

- `rag.search.default-top-k = 10`
- `vector-global.enabled = true`、`confidence-threshold = 0.6`、`single-intent-supplement-threshold = 0.8`、`top-k-multiplier = 3`
- `intent-directed.enabled = true`、`min-intent-score = 0.4`、`top-k-multiplier = 2`

其他相关开关：

- `rag.vector.type: pg`（默认；可切换 `milvus`），决定 `RetrieverService` 实现。
- `rag.rerank-enabled`：控制 `RerankPostProcessor` 是否启用；Rerank 模型为 `qwen3-rerank`（bailian）。

## 未来扩展

1. **ES 关键词检索通道**：实现 `SearchChannel`，对应已预留的 `SearchChannelType.KEYWORD_ES`，做全文 / 分词检索。
2. **混合检索通道**：对应 `SearchChannelType.HYBRID`，融合向量 + 关键词。
3. **版本过滤处理器**：只保留最新版本文档。
4. **分数归一化处理器**：统一不同通道的分数尺度，改善去重与重排公平性。
5. **检索结果缓存**与**通道命中率 / 耗时监控统计**（`MultiChannelRetrievalEngine` 已记录每个通道的 `latencyMs` 与 Chunk 数，可进一步上报）。

## 测试建议

1. **单元测试**：分别测试各 `SearchChannel` 的 `isEnabled` 触发条件与 `search` 逻辑；测试各 `SearchResultPostProcessor` 的处理结果（尤其去重的优先级保留规则）。
2. **集成测试**：验证多通道并行执行、后置处理器按 `order` 串行执行的顺序与容错（单通道 / 单处理器抛异常时整链仍可返回）。
3. **性能测试**：对比单通道与多通道的延迟；验证不同 `top-k-multiplier`、是否启用 Rerank 下的耗时。
4. **召回 / 精准度测试**：对比原方案与新方案的召回率，重点验证意图识别失败时全局检索的兜底效果，以及 `qwen3-rerank` 对最终精准度的提升。

## 注意事项

1. **通道优先级**：`SearchChannel#getPriority()` 数字越小优先级越高（意图定向 `1` < 全局向量 `10`），去重时优先保留高优先级通道的 Chunk。
2. **处理器顺序**：`getOrder()` 决定执行顺序，去重（`order=1`）应最先、Rerank（`order=10`）应最后。
3. **性能考虑**：启用过多通道会增加聚合延迟，建议结合 `isEnabled` 条件触发按需启用。
4. **配置调优**：`confidence-threshold`、`single-intent-supplement-threshold`、`min-intent-score`、`top-k-multiplier` 等需结合实际效果调整。
5. **向量后端**：切换 `rag.vector.type` 会切换底层 `RetrieverService` 实现（pg / milvus），需保证对应中间件可用、维度一致（1536，COSINE）。

## 总结

本次重构把检索逻辑抽象为「**多通道并行召回 + 后处理流水线**」：以策略模式承载多种检索通道、以责任链模式承载去重与重排，由 `MultiChannelRetrievalEngine` 统一编排，`RetrievalEngine` 作为业务入口集成。通过**条件触发的双路召回**（意图定向为主、全局向量兜底）在召回率与精准度之间取得平衡，并借助 Spring 自动注入实现了新增通道 / 处理器的零改动扩展。

新架构的核心优势：

- **高召回率**：全局向量检索按需兜底，缓解意图识别覆盖率不足。
- **高精准度**：意图定向检索优先 + `qwen3-rerank` 重排。
- **易扩展**：新增 `SearchChannel` / `SearchResultPostProcessor` 即可，无需改动核心引擎。
- **灵活配置**：`rag.search` 系列配置 + `rag.vector.type`（pg/milvus 可切换）+ `rag.rerank-enabled`。
- **性能可控、容错隔离**：通道与处理器并行 / 串行编排，单点异常不影响整体。
