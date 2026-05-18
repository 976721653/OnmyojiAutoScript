## Why

项目缺乏一份系统级的架构文档。现有 `.agents/docs/knowledge/project-architecture.md` 是在项目初始化时基于快速扫描编写的概要，信息深度不足。经过对 Config、Device、调度执行、Tasks 层、GUI/Server/OCR 等五个核心子系统的深度探索，现在有足够的认知来编写一份可指导后续开发的完整架构参考文档。

## What Changes

- **重写** `.agents/docs/knowledge/project-architecture.md`：从概要升级为完整架构文档，覆盖全部 7 个子系统
- **新增** 子文档（按需拆分）：若单文件过大，将子系统详解拆分为独立文档
- **更新** `.agents/docs/knowledge/INDEX.md`：同步文档条目和摘要

## Capabilities

### New Capabilities

- `arch-doc-structure`：架构文档的组织结构规范——定义文档覆盖的子系统范围、每个子系统的记录深度、文档间的引用关系
- `arch-doc-content`：架构文档的内容规格——定义每个子系统需要记录的要素（类层次、数据流、关键 API、设计决策）

### Modified Capabilities

_无_

## Impact

- 修改文件：`.agents/docs/knowledge/project-architecture.md`（全面重写）
- 新增文件：可能需要拆分的子系统子文档（如 `module-device-architecture.md` 等）
- 修改文件：`.agents/docs/knowledge/INDEX.md`（同步更新）
- 不影响任何应用代码
