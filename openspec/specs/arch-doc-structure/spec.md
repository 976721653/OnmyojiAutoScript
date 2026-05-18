## ADDED Requirements

### Requirement: 主文档组织

架构文档的主文件 `project-architecture.md` SHALL 按以下结构组织，每个子系统独立成章：

1. 整体架构（分层图 + 核心链路概述）
2. Config 配置系统
3. Device 设备抽象
4. 调度与任务执行
5. Tasks 任务体系
6. 页面导航系统
7. Atom 检测系统与 OCR
8. GUI 与 Server 交互层
9. 通知与多人协作
10. 异常体系与日志

#### Scenario: Agent 查找子系统信息

- **WHEN** Agent 需要了解某个子系统（如 Device）的架构
- **THEN** Agent 在 INDEX.md 中找到 `project-architecture.md` 的摘要，打开文档后可直接跳转到对应章节，无需通读全文

---

### Requirement: 子文档拆分阈值

当任一子系统的文档篇幅超过 200 行 markdown 时，该子系统的详细内容 SHALL 拆分为独立子文档，主文档仅保留概要（3-5 段关键描述 + 子文档链接）。

#### Scenario: 子系统文档过长

- **WHEN** Device 子系统的架构描述超过 200 行
- **THEN** 创建 `module-device-architecture.md` 独立文档，`project-architecture.md` 中 Device 章节缩减为概要并链接到子文档；同时更新 `docs/knowledge/INDEX.md` 添加新文档条目

---

### Requirement: 文档元信息

主文档和所有子文档开头 SHALL 包含符合 `agent-doc-standards` 规范的元信息表格头。

#### Scenario: 文档元信息表格

- **WHEN** 架构文档被创建或更新
- **THEN** 文档开头包含类型（知识库）、状态（草稿/已验证）、更新日期、标签（#架构）、摘要的元信息表格

---

### Requirement: ASCII 架构图

架构文档中 SHALL 使用 ASCII 图（嵌入 markdown 代码块）表示架构分层、类继承关系、数据流和状态机。

#### Scenario: 包含 ASCII 架构图

- **WHEN** 读者查看架构文档
- **THEN** 每个核心子系统章节至少包含一个 ASCII 图，帮助理解该系统的结构和关系
