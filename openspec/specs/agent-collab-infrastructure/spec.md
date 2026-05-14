## ADDED Requirements

### Requirement: AGENTS.md 入口文档

项目根目录 SHALL 存在一个 `AGENTS.md` 文件，作为 AI Agent 进入项目的首个着陆点，包含项目概述、开发环境、协同指引和行为准则四个部分。

#### Scenario: Agent 首次进入项目

- **WHEN** Agent 被分配到本项目的开发任务
- **THEN** Agent 首先读取 `AGENTS.md`，从中了解项目基本信息、技术栈、目录结构，并获知 `.agents/` 和 `openspec/` 的存在和用途

#### Scenario: 人类开发者查看协同开发指引

- **WHEN** 人类开发者打开 `AGENTS.md`
- **THEN** 看到清晰的项目概述和 Agent 协同开发的工作方式说明

---

### Requirement: .agents 目录结构

项目根目录 SHALL 存在 `.agents/` 目录，包含 docs（文档库）、reference-projects（参考项目）、resource（本地资源）三个子目录，每个子目录及其子类目目录根部均含 INDEX.md 索引文件。

#### Scenario: Agent 导航文档库

- **WHEN** Agent 需要查找项目相关知识文档
- **THEN** Agent 依次读取 `.agents/INDEX.md` → `docs/INDEX.md` → 具体类目 `INDEX.md`，在 3 步内定位到目标文档

#### Scenario: 扩展新子目录

- **WHEN** 未来需要在 `.agents/` 下新增 skills 子目录
- **THEN** skills 目录遵循相同的 INDEX.md 索引策略

---

### Requirement: 每层 INDEX.md 索引文件

`.agents/` 下每个目录根部 SHALL 包含一个 `INDEX.md` 文件，以表格形式列出该目录下所有子节点（目录或文档），包含名称、类型、摘要等元信息。

#### Scenario: INDEX.md 表格结构

- **WHEN** Agent 读取任意层级的 `INDEX.md`
- **THEN** 看到该目录下所有子节点的列表表格，Agent 通过摘要字段判断是否需要进一步读取具体文档

---

### Requirement: reference-projects 参考项目管理

`.agents/reference-projects/` 目录 SHALL 包含一个 `INDEX.md` 文件，记录所有可参考项目的名称、描述和 git 仓库地址。除 INDEX.md 外，该目录下所有文件和子目录均被 .gitignore 忽略。

#### Scenario: Agent 需要参考外部项目

- **WHEN** 用户提及某个可参考的项目
- **THEN** Agent 首先查阅 `reference-projects/INDEX.md`，检查项目是否在索引中，然后检查本地是否已有 clone 代码，若没有则通过 git clone 拉取到该目录下

#### Scenario: 参考项目代码不被提交

- **WHEN** Agent 将参考项目代码 clone 到 reference-projects/ 下
- **THEN** git 不会追踪这些文件，不会意外提交到仓库

---

### Requirement: resource 本地资源管理

`.agents/resource/` 目录 SHALL 包含一个 `INDEX.md` 文件，记录所有可查阅资源的名称、描述和获取方式。除 INDEX.md 外，该目录下所有文件均被 .gitignore 忽略。

#### Scenario: Agent 需要本地资源

- **WHEN** Agent 需要使用某个本地参考资源
- **THEN** Agent 先查阅 `resource/INDEX.md`，若资源存在则直接使用，若不存在则告知用户需要提供该资源

---

### Requirement: .gitignore 忽略规则

项目根目录的 `.gitignore` SHALL 包含规则，使 `.agents/reference-projects/` 和 `.agents/resource/` 下除 INDEX.md 外的所有文件和目录不被 git 追踪。

#### Scenario: git status 不显示实体文件

- **WHEN** reference-projects/ 或 resource/ 下有 clone 的代码或下载的资源文件
- **THEN** `git status` 不显示这些文件为未追踪，不会误加入暂存区
