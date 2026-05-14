## Context

项目目前已有 `.claude/`（Claude Code 工具配置）和 `openspec/`（规格驱动开发流程），但缺少 Agent 协同开发的通用基础设施。AGENTS.md 和 `.agents/` 目录体系填补这个空白，为 Agent 提供结构化的项目知识导航系统。

约束：不影响现有代码和目录结构，仅新增文件。

## Goals / Non-Goals

**Goals:**
- 建立 Agent 进入项目的统一入口（AGENTS.md）
- 建立结构化文档库（.agents/docs/），让 Agent 高效查阅项目知识
- 建立参考项目管理系统（.agents/reference-projects/），Agent 按需拉取
- 建立本地资源管理系统（.agents/resource/）
- 统一文档元信息格式，支持 Agent 索引式读取
- 正确配置 .gitignore，避免第三方代码和资源入库

**Non-Goals:**
- 不填充 docs/ 下的血肉内容（编码规范、架构文档等）——那是 Phase 2 的工作
- 不创建 .agents/skills/（未来扩展）
- 不修改现有 module/、tasks/、config/ 中的代码
- 不修改 openspec/ 和 .claude/ 的结构

## Decisions

### 1. 目录结构设计

```
.agents/
├── INDEX.md                 ← 总索引
├── docs/
│   ├── INDEX.md             ← 文档库索引
│   ├── dev-logs/
│   │   └── INDEX.md
│   ├── knowledge/
│   │   └── INDEX.md
│   ├── pitfalls/
│   │   └── INDEX.md
│   └── rules/
│       └── INDEX.md
├── reference-projects/
│   └── INDEX.md             ← 参考项目大全（唯一提交的文件）
├── resource/
│   └── INDEX.md             ← 资源大全（唯一提交的文件）
└── skills/                  ← 未来扩展，暂不创建
```

**理由**：每层 INDEX.md 作为 Agent 路由表，3 步定位：总索引 → 子目录索引 → 类目索引 → 具体文档。

### 2. 索引策略

Agent 读取路径：
```
AGENTS.md → .agents/INDEX.md → docs/INDEX.md → 具体类目/INDEX.md → 目标文档
```

每层 INDEX.md 以表格列出子节点（目录或文档），包含元信息的关键字段（名称、类型、摘要）。Agent 先读摘要，按需打开正文，避免全量读取。

**备选方案**：直接让 Agent glob 所有文件后用摘要判断。不采用——glob 依赖工具能力，跨平台不一致，且无摘要预判。

### 3. 文档元信息格式

每个 docs 下文档开头使用纯 markdown 表格头：

```markdown
# 文档名称：XXX

| 属性 | 内容 |
|------|------|
| 类型 | 知识库 / 开发日志 / 常见场景 / 规范 |
| 状态 | 草稿 / 已验证 / 待更新 |
| 更新 | YYYY-MM-DD |
| 标签 | #tag1 #tag2 |
| 摘要 | 一句话描述文档核心内容 |
| 关联 | 可选，链接到其他相关文档 |
---
```

**设计考量**：
- 纯 markdown，人和 Agent 均可读
- 表格在解析和渲染上都直观
- "摘要"是 Agent 预判文档相关性的关键字段
- "状态"支持文档生命周期管理，防止 Agent 引用过时内容

### 4. .gitignore 策略

reference-projects 和 resource 目录下，仅 INDEX.md 提交，其他实体文件和目录全部忽略：

```gitignore
# .agents - reference projects and resources (entities only, INDEX.md tracked)
.agents/reference-projects/*
!.agents/reference-projects/INDEX.md
.agents/resource/*
!.agents/resource/INDEX.md
```

**理由**：参考项目代码和资源文件可能体积大、有独立许可证、与项目无关，不应入库。Agent 按需从 INDEX.md 中的源信息 git clone / 下载。

### 5. AGENTS.md 内容骨架

AGENTS.md 是 Agent 的着陆点，内容分四部分：
1. 项目概述（一句话 + 技术栈 + 结构速览）
2. 开发环境（环境搭建、运行、常用命令）
3. 协同指引（.agents/ 说明、openspec/ 说明、规则入口）
4. 行为准则（可做/不可做/需确认的边界）

AGENTS.md 本身不包含过多细节——细节通过链接指向 .agents/docs/ 下的具体文档。

## Risks / Trade-offs

- **INDEX.md 维护负担**：每次新增/修改文档需同步更新 INDEX.md。→ 可在后续制定规则，要求新增文档时强制更新索引；未来可考虑自动化脚本
- **文档元信息格式靠人工遵守**：无自动校验机制。→ 暂接受，后续可加 lint 脚本
- **reference-projects 拉取成本**：Agent 需要 clone 参考项目到本地。→ 仅在使用到时才拉取，且拉取后本地缓存，避免重复
