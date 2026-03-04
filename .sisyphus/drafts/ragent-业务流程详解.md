# Draft: Ragent 业务流程详解

## Requirements (confirmed)
- 参考文件：`D:\programFile\Java\JavaEE\SpringCloud\shortlink\shortlink-documents\日志分析与业务流程详解.md`
- 目标产物：Ragent 项目版本的“业务流程详解”文档
- 详细程度：非常详细，面向新手可读
- 目标路径：`docs/Ragent业务详解.md`
- 用户偏好：先看摘要后再进入计划生成
- 深度偏好：超详细新手版
- 呈现偏好：文字 + Mermaid 流程图
- 新增要求：加入“面试话术速记”小节

## Technical Decisions
- 文档结构沿用“架构总览 → 端到端流程 → 关键模块 → 核心机制 → 常见问题/术语”的讲解方式
- 内容以仓库现有资料与高置信实现证据为主：README、`docs/ragent-project-guide.md`、`docs/multi-channel-retrieval.md` 等
- 采用“先概念后时序再落地”的新手友好叙述方式

## Research Findings
- 角色与模块：用户端、管理端、RAG 编排、检索引擎、模型路由与熔断、MCP 工具、摄取流水线
- 主链路：会话记忆 → 查询改写/拆分 → 意图识别 → 多通道检索 + 后处理 → Prompt 组装 → 流式生成
- 检索机制：意图定向通道 + 全局向量兜底；后处理含去重与 Rerank
- 稳定性机制：模型优先级路由、三态熔断、首包探测；分布式排队限流（Redis ZSET + Lua + Pub/Sub + Semaphore）
- 关键实现线索（高置信）：`RAGChatServiceImpl`、`MultiChannelRetrievalEngine`、`ModelRoutingExecutor`、`ModelHealthStore`、`IngestionEngine`

## Scope Boundaries
- INCLUDE：Ragent 业务全流程、模块职责、关键机制、对新手友好的术语与示例时序
- EXCLUDE：源码改动、接口实现、配置调优实操、部署运维细节

## Open Questions
- 无
