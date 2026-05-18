## Context

现有 `project-architecture.md` 是项目初始化时的概要（约 100 行），覆盖了基本架构但深度不足。我们已通过 5 个并行 Agent 完成了对 Config、Device、调度/GUI/Server、Tasks 层、OCR/Atom/基础工具五大子系统的深度探索，积累了丰富的素材。本次将基于这些素材全面重写架构文档。

约束：架构是既成事实，文档是记录而非设计，不需要讨论技术方案优劣。

## Goals / Non-Goals

**Goals:**
- 写出一份可指导后续开发的完整架构参考文档
- 覆盖全部核心子系统：Config、Device、调度执行、Tasks/Component、页面导航、Atom/OCR、GUI、Server、通知、Team Flow
- 包含架构图、数据流图、类继承图（ASCII 格式）
- 记录关键设计决策和已有模式

**Non-Goals:**
- 不分析和评论架构设计的好坏（既成事实）
- 不提出架构改进建议
- 不修改任何应用代码
- 不在文档中包含 API 签名级别的细节（保持模块级）

## Decisions

### 1. 文档组织：单文件 vs 拆分

**决策**：采用"一篇主文档 + 按需子文档"的结构。

- 主文档 `project-architecture.md`：整体架构图、核心链路、子系统概览（各子系统 3-5 段 + 关键图）
- 子文档：仅当子系统复杂度超过 200 行时才拆分（如 Device 子系统、页面导航系统）

**理由**：单文件过大不利于 Agent 按需读取，但过度拆分增加索引维护负担。主文档提供全局视图，Agent 通过 INDEX.md 按需深入子系统详情。

### 2. 写作风格

**决策**：参考手册风格（非教程叙事）。

- 每个子系统独立成章，Agent 可按需跳读
- 关键概念用粗体标记
- 代码模式用简短代码块说明
- 避免从"一条任务执行链路"出发的叙事式——那适合导读但不适合查证

### 3. 架构图策略

**决策**：全部使用 ASCII 图，嵌入 markdown 代码块中。

- 整体架构分层图
- Device 多继承 MRO 图
- Config 数据流图
- Task 生命周期状态机
- 页面导航有向图

ASCII 图可被任何编辑器渲染，不依赖外部工具。

### 4. 覆盖范围

文档覆盖以下子系统（按逻辑分组）：

| 组 | 子系统 | 深度 |
|----|--------|------|
| 核心链路 | Config → Device → Task | 模块级 + 关键类 |
| 调度系统 | Script 主循环、RuntimeController、Scheduler | 流程级 |
| 任务体系 | BaseTask API、Component 清单、Task 模式 | API 级 |
| 导航系统 | Page、Transition、Dijkstra、Session | 类级 |
| 检测系统 | Atom 规则类型、OCR 引擎、RPC 架构 | 模块级 |
| 交互层 | GUI（PySide6+FluentUI）、Server（FastAPI） | 架构级 |
| 辅助系统 | 通知（onepush）、Team Flow（MQTT）、日志 | 模块级 |

## Risks / Trade-offs

- **维护负担**：架构文档需要随代码变更同步更新 → 在文档元信息中标明"状态"字段，定期审查
- **准确性风险**：文档基于当前代码快照，未来可能过时 → 每次重大重构通过 openspec change 触发文档更新
- **深度权衡**：太深变成代码注释，太浅无参考价值 → 以"模块级接口 + 关键模式"为边界，类级细节仅在必要时记录
