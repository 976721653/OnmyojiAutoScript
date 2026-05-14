# docs/ 文档库索引

docs 是 Agent 协同开发的主要知识存储空间。文档按类型分入四个子目录：

| 目录 | 类型 | 说明 | 详情 |
|------|------|------|------|
| [knowledge/](knowledge/INDEX.md) | 知识库 | 项目架构、模块设计、技术原理等通用知识 | [→ INDEX](knowledge/INDEX.md) |
| [dev-logs/](dev-logs/INDEX.md) | 开发日志 | 开发过程的记录、决策上下文 | [→ INDEX](dev-logs/INDEX.md) |
| [pitfalls/](pitfalls/INDEX.md) | 常见场景 | 踩坑记录、已知问题、注意事项 | [→ INDEX](pitfalls/INDEX.md) |
| [rules/](rules/INDEX.md) | 规范 | 编码规范、命名约定、必须遵守的规则 | [→ INDEX](rules/INDEX.md) |

## 文档元信息规范

每个文档开头必须包含以下元信息表格：

```markdown
# 文档名称：XXX

| 属性 | 内容 |
|------|------|
| 类型 | 知识库 / 开发日志 / 常见场景 / 规范 |
| 状态 | 草稿 / 已验证 / 待更新 |
| 更新 | YYYY-MM-DD |
| 标签 | #tag1 #tag2 |
| 摘要 | 一句话描述文档核心内容（不超过80字） |
| 关联 | 可选，链接到其他相关文档 |

---
```
