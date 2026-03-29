# Ragent 业务流程详解文档工作计划

## TL;DR

> **Quick Summary**: 产出一份面向新手的超详细业务流程文档 `docs/Ragent业务详解.md`，覆盖 Ragent 从在线问答到离线入库再到管理闭环的完整业务链路，并补充 Mermaid 图与“面试话术速记”。
>
> **Deliverables**:
> - `docs/Ragent业务详解.md`（主文档）
> - 文档内至少 6 张 Mermaid 图（架构/时序/分支/流水线/闭环/熔断状态）
> - 文档内“面试话术速记”章节（30 秒版 + 2 分钟版 + 追问应对）
>
> **Estimated Effort**: Medium
> **Parallel Execution**: YES - 2 waves
> **Critical Path**: 证据梳理 → 大纲与边界固化 → 主文撰写 → Mermaid 与验收修订

---

## Context

### Original Request
- 参考：`D:\programFile\Java\JavaEE\SpringCloud\shortlink\shortlink-documents\日志分析与业务流程详解.md`
- 目标：写一份 Ragent 项目版本的业务流程详解
- 要求：非常详细，新手可读
- 交付：`docs/Ragent业务详解.md`
- 新增：加入“面试话术速记”小节

### Interview Summary
**已确认偏好**：
- 深度：超详细新手版
- 形式：文字 + Mermaid 流程图
- 输出方式：先给摘要，后进入正式计划

**已确认内容范围**：
- 角色与系统全景
- 在线问答主链路
- 异常分支（排队超时、歧义引导、低召回、模型切换、用户取消）
- 文档摄取流水线
- 管理闭环
- 核心机制拆解与术语表
- 面试话术速记

### Metis Review（已吸收）
**关键补强点**：
- 明确“非常详细”的边界（章节与图数量、术语数量、话术条数）
- 严禁臆造参数/类名，阈值需标注“示例或默认，实际以配置为准”
- 强制区分“主链路”与“异常链路”
- 防止范围膨胀到部署/调优/源码逐行讲解

---

## Work Objectives

### Core Objective
在不修改源码功能的前提下，输出一份结构化、可读性强、对新手友好的 Ragent 业务流程详解文档，帮助读者快速建立“角色-链路-机制-闭环”的完整认知。

### Concrete Deliverables
- `docs/Ragent业务详解.md`（唯一交付文件）
- 文档内必含以下模块：
  1) 项目定位与角色
  2) 在线问答主流程
  3) 异常分支详解
  4) 文档摄取流水线
  5) 管理闭环
  6) 核心机制拆解
  7) 新手术语表
  8) 面试话术速记

### Definition of Done
- [ ] `docs/Ragent业务详解.md` 文件存在且可读
- [ ] 至少 6 张 Mermaid 图，语法可渲染
- [ ] 覆盖 5 个异常分支并逐项解释“触发条件/系统动作/用户感知”
- [ ] 术语表不少于 25 条
- [ ] 面试话术速记不少于 10 条，含“30秒版/2分钟版”

### Must Have
- 新手可读（首次术语出现即解释）
- 证据导向（关键机制关联到现有文档/类名）
- 主链路与异常链路分离讲解
- 明确“示例配置”与“真实配置”的边界说明

### Must NOT Have (Guardrails)
- 不做源码实现、接口改造、部署步骤、性能调优手册
- 不臆造类名、阈值、数据库字段、未落地能力
- 不把 README 大段复制粘贴当“详解”
- 不把“面试话术速记”扩展成完整面经

---

## Verification Strategy (Documentation-Focused)

### Test Decision
- **Infrastructure exists**: YES（项目有测试体系，但本任务为文档交付）
- **User wants tests**: NO（文档任务）
- **Framework**: none（使用文档质量验收命令）

### Automated Verification Only (No User Intervention)

1) 文件存在性校验
```bash
test -f "docs/Ragent业务详解.md"
```

2) 结构完整性校验（章节）
```bash
grep -n "^## " "docs/Ragent业务详解.md"
```

3) Mermaid 图数量校验
```bash
grep -n "^```mermaid" "docs/Ragent业务详解.md"
```
> 期望：不少于 6 处。

4) 异常分支覆盖校验
```bash
grep -n "排队超时\|歧义\|低召回\|模型切换\|用户取消" "docs/Ragent业务详解.md"
```

5) 面试话术章节校验
```bash
grep -n "面试话术速记\|30 秒\|2 分钟" "docs/Ragent业务详解.md"
```

6) 关键组件名一致性校验
```bash
grep -n "RAGChatServiceImpl\|MultiChannelRetrievalEngine\|ModelRoutingExecutor\|ModelHealthStore\|IngestionEngine\|ChatQueueLimiter" "docs/Ragent业务详解.md"
```

---

## Execution Strategy

### Parallel Execution Waves

```
Wave 1 (Start Immediately):
├── Task 1: 证据矩阵与素材抽取（主链路/机制/术语）
└── Task 2: 文档骨架与边界模板（章节、图位、口径声明）

Wave 2 (After Wave 1):
├── Task 3: 主文撰写（主链路+异常分支+摄取+闭环）
├── Task 4: Mermaid 图补全（6图）
└── Task 5: 面试话术速记 + 术语表 + 全文验收修订

Critical Path: Task 1 → Task 3 → Task 4 → Task 5
Parallel Speedup: ~25%
```

### Dependency Matrix

| Task | Depends On | Blocks | Can Parallelize With |
|------|------------|--------|----------------------|
| 1 证据矩阵 | None | 3,5 | 2 |
| 2 文档骨架 | None | 3,4,5 | 1 |
| 3 主文撰写 | 1,2 | 4,5 | None |
| 4 Mermaid 图 | 2,3 | 5 | None |
| 5 总修订验收 | 1,2,3,4 | None | None |

### Agent Dispatch Summary

| Wave | Tasks | Recommended Agents |
|------|-------|--------------------|
| 1 | 1,2 | `delegate_task(category="writing", load_skills=["notebooklm"], run_in_background=true)`（若无需外部知识库，可不加载 notebooklm） |
| 2 | 3,4,5 | `delegate_task(category="writing", run_in_background=false)` 顺序执行并最终校验 |

---

## TODOs

- [ ] 1. 建立“证据矩阵”（来源→结论→写作位置）

  **What to do**:
  - 从 `README.md`、`docs/ragent-project-guide.md`、`docs/multi-channel-retrieval.md` 抽取关键事实
  - 形成“机制名-关键类-解释要点-限制说明”映射

  **Must NOT do**:
  - 不从未验证来源引入参数
  - 不使用推测语气写成既定事实

  **Recommended Agent Profile**:
  - **Category**: `writing`
    - Reason: 该任务是结构化整理证据，不涉及实现。
  - **Skills**: `notebooklm`（可选）
    - `notebooklm`: 若需对比多个文档来源并做引用一致性检查。
  - **Skills Evaluated but Omitted**:
    - `ui-ux-pro-max`: 与纯文档证据整理域重叠低。

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1（与 Task 2）
  - **Blocks**: 3,5
  - **Blocked By**: None

  **References**:
  - `README.md` - 全局能力、核心链路、设计模式与工程特性主叙述
  - `docs/ragent-project-guide.md` - 模块定位、主链路、核心类名、面试表达素材
  - `docs/multi-channel-retrieval.md` - 检索通道与后处理细节补充

  **Acceptance Criteria**:
  - [ ] 形成证据矩阵（至少覆盖 8 个核心机制/模块）
  - [ ] 每个机制都能指向至少 1 个内部来源文件

  **Commit**: NO

- [ ] 2. 固化文档骨架与章节边界

  **What to do**:
  - 固定 13 章结构（见本计划“Suggested Outline”）
  - 标注每章目标、读者收益、输入证据来源
  - 预留 6 个 Mermaid 图位（图标题 + 图类型 + 目的）

  **Must NOT do**:
  - 不提前进入大段正文写作

  **Recommended Agent Profile**:
  - **Category**: `writing`
    - Reason: 结构设计任务。
  - **Skills**: `notebooklm`（可选）

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1（与 Task 1）
  - **Blocks**: 3,4,5
  - **Blocked By**: None

  **References**:
  - `D:\programFile\Java\JavaEE\SpringCloud\shortlink\shortlink-documents\日志分析与业务流程详解.md` - 风格参照（结构化拆解与流程叙述）

  **Acceptance Criteria**:
  - [ ] 章节结构定稿
  - [ ] 图位与章节一一对应

  **Commit**: NO

- [ ] 3. 撰写主文：角色、主链路、异常分支、摄取、闭环

  **What to do**:
  - 完成新手导读：先讲“角色与职责”
  - 完成在线问答主链路逐步拆解（输入、处理、输出）
  - 完成 5 大异常分支逐条说明（触发条件/系统动作/用户感知）
  - 完成摄取流水线与管理闭环

  **Must NOT do**:
  - 不把部署命令、调优参数当正文重点

  **Recommended Agent Profile**:
  - **Category**: `writing`
    - Reason: 高密度技术解释与教学化表达。
  - **Skills**: `notebooklm`（可选）

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Sequential
  - **Blocks**: 4,5
  - **Blocked By**: 1,2

  **References**:
  - `README.md` - 主链路与工程设计的业务语义
  - `docs/ragent-project-guide.md` - 细化时序、核心类名与模块关系
  - `docs/interview-strategy.md` - “面试话术速记”的语境素材

  **Acceptance Criteria**:
  - [ ] 主链路章节完整
  - [ ] 5 个异常分支均有独立子节
  - [ ] 章节中关键类名表述一致且无冲突

  **Commit**: NO

- [ ] 4. 生成并插入 Mermaid 图（至少 6 张）

  **What to do**:
  - 绘制：总体架构图、问答时序图、异常分支决策图、摄取流水线图、管理闭环图、熔断状态图
  - 保持图中术语与正文一致

  **Must NOT do**:
  - 不使用复杂语法导致渲染失败

  **Recommended Agent Profile**:
  - **Category**: `writing`
    - Reason: 文档可视化表达。
  - **Skills**: `ui-ux-pro-max`（可选）
    - `ui-ux-pro-max`: 优化信息可视化排版与图文层次。

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Sequential
  - **Blocks**: 5
  - **Blocked By**: 2,3

  **References**:
  - `docs/ragent-project-guide.md` - 用于确保链路顺序与命名一致

  **Acceptance Criteria**:
  - [ ] 6 张 Mermaid 均可渲染
  - [ ] 图与对应章节含义一致

  **Commit**: NO

- [ ] 5. 完成“术语表 + 面试话术速记 + 全文一致性验收”

  **What to do**:
  - 术语表按 4 类分组：检索、生成、稳定性、工程化
  - 面试话术速记输出至少 10 条（30 秒/2 分钟/追问应对）
  - 全文做一致性校对：术语、类名、流程顺序、图文对应
  - 执行自动化质量校验命令

  **Must NOT do**:
  - 不夸大个人 ownership
  - 不输出无法从项目语境支撑的 KPI 数字

  **Recommended Agent Profile**:
  - **Category**: `writing`
    - Reason: 内容打磨与可信度校验。
  - **Skills**: `notebooklm`（可选）

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Final
  - **Blocks**: None
  - **Blocked By**: 1,2,3,4

  **References**:
  - `docs/interview-strategy.md` - 面试表达语气和问答方向
  - `README.md` - 项目定位与能力范围约束

  **Acceptance Criteria**:
  - [ ] 术语表 ≥ 25 条
  - [ ] 面试话术速记 ≥ 10 条
  - [ ] 自动化校验命令全部通过

  **Commit**: NO

---

## Suggested Final Doc Outline (for executor)

1. 写在前面：本文目标、读者、阅读方式
2. Ragent 是什么：一句话定位与业务价值
3. 角色与架构全景
4. 在线问答主流程（一步一步）
5. 异常分支详解（5 条）
6. 文档摄取流水线
7. 管理闭环（配置-观测-反馈-优化）
8. 核心机制拆解（问题/方案/代价/常见误解）
9. 端到端样例（成功路径 + 失败路径）
10. 新手术语表（≥25）
11. 面试话术速记（30 秒/2 分钟/追问）
12. 常见误区与排错入口
13. 总结与下一步阅读路径

---

## Gap Classification (Post-Plan Self-Review)

### Auto-Resolved（Minor）
- “非常详细”的边界已量化：章节、图数量、术语数量、话术数量。
- 参考来源优先级已固定：README 与 docs 内文档为主，风格参考 shortlink 文档。

### Defaults Applied（Ambiguous）
- Mermaid 图数量默认设为 **至少 6 张**。
- 面试话术默认分为 **30 秒版 / 2 分钟版 / 追问应对** 三层。
- 文档验收默认使用 shell 结构校验（文件、章节、关键词、图块）。

### Critical Decisions Needed
- 无（用户已确认加入“面试话术速记”，深度和形式也已确认）。

---

## Success Criteria

### Verification Commands
```bash
test -f "docs/Ragent业务详解.md"
grep -n "^## " "docs/Ragent业务详解.md"
grep -n "^```mermaid" "docs/Ragent业务详解.md"
grep -n "排队超时\|歧义\|低召回\|模型切换\|用户取消" "docs/Ragent业务详解.md"
grep -n "面试话术速记\|30 秒\|2 分钟" "docs/Ragent业务详解.md"
grep -n "RAGChatServiceImpl\|MultiChannelRetrievalEngine\|ModelRoutingExecutor\|ModelHealthStore\|IngestionEngine\|ChatQueueLimiter" "docs/Ragent业务详解.md"
```

### Final Checklist
- [ ] 所有 Must Have 均满足
- [ ] 所有 Must NOT Have 均未违反
- [ ] 结构、图示、术语、话术、分支覆盖全部达标
