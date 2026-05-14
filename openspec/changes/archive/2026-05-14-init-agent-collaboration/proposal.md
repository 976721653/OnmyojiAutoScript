## Why

当前项目缺乏 AI Agent 协同开发的基础设施——没有项目入口文档、没有结构化知识库、没有统一的文档元信息标准，Agent 无法高效理解和参与项目开发。需要在项目根目录建立 `.agents/` 目录体系，作为 Agent 协同开发的长期沉淀空间，让 Agent 进入项目时能够快速定位所需信息。

## What Changes

- **新增 `AGENTS.md`**：Agent 进入项目的首个着陆点，概述项目结构、开发环境和协同指引
- **新增 `.agents/` 目录体系**：包含 docs（文档库）、reference-projects（参考项目）、resource（本地资源）三大子目录，未来可扩展 skills
- **新增全量 `INDEX.md`**：每层目录根部的索引文件，Agent 通过 3 步读取定位具体文档，避免全量遍历
- **新增文档元信息标准**：每个 docs 下文档开头必须有结构化表格头（类型、状态、更新、标签、摘要），便于 Agent 索引
- **新增 `.gitignore` 规则**：reference-projects 和 resource 的实体文件忽略，仅保留 INDEX.md 提交
- 三层职责分离：`.agents/`（长期沉淀）、`openspec/`（沟通流程）、`.claude/`（工具配置）

## Capabilities

### New Capabilities

- `agent-collab-infrastructure`：.agents/ 目录结构、AGENTS.md 入口、索引策略和 gitignore 配置
- `agent-doc-standards`：文档元信息表格头格式规范、文档类型定义、状态生命周期

### Modified Capabilities

_无（首次初始化，不涉及已有 capabilities 修改）_

## Impact

- 新增文件：`AGENTS.md`、`.agents/` 下的全量 INDEX.md 和目录结构
- 修改文件：`.gitignore`（追加 reference-projects 和 resource 忽略规则）
- 不影响现有 module/、tasks/、config/ 中的任何代码
- 与 `.claude/` 和 `openspec/` 无冲突，各司其职
