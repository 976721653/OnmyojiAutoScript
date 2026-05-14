## 1. 骨架层：目录结构和索引体系

- [x] 1.1 创建 `.agents/` 目录及所有子目录（docs/、docs/dev-logs/、docs/knowledge/、docs/pitfalls/、docs/rules/、reference-projects/、resource/）
- [x] 1.2 创建 `AGENTS.md`，包含项目概述、开发环境、协同指引、行为准则四部分
- [x] 1.3 创建 `.agents/INDEX.md`，列出 docs、reference-projects、resource 三个子目录的入口
- [x] 1.4 创建 `.agents/docs/INDEX.md`，列出四个类目的入口和简要说明
- [x] 1.5 创建 docs 下四个类目的 INDEX.md（`docs/dev-logs/INDEX.md`、`docs/knowledge/INDEX.md`、`docs/pitfalls/INDEX.md`、`docs/rules/INDEX.md`），初始为空文档列表
- [x] 1.6 创建 `.agents/reference-projects/INDEX.md`，包含可参考项目的索引格式模板
- [x] 1.7 创建 `.agents/resource/INDEX.md`，包含可查阅资源的索引格式模板
- [x] 1.8 更新根目录 `.gitignore`，追加 reference-projects 和 resource 的忽略规则（除 INDEX.md 外全部忽略）

## 2. 血肉层：实质性文档内容

- [x] 2.1 编写 `docs/rules/` 下的编码规范文档，包含 Python 编码风格、文件命名规范、模块组织方式
- [x] 2.2 编写 `docs/knowledge/` 下的项目架构文档，描述 module/ 层级、tasks/ 任务模型、Config-Device-Task 核心链路
- [x] 2.3 编写 `docs/pitfalls/` 下的已有踩坑记录，收集开发过程中的经验教训
- [x] 2.4 更新各层 INDEX.md，将新创建的文档条目和摘要同步到索引中
