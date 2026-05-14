# AGENTS.md — Agent 协同开发入口

## 1. 项目概述

OnmyojiAutoScript（OAS）是阴阳师手游的自动化脚本，基于 [ALAS](https://github.com/LmeSzinc/AzurLaneAutoScript) 框架开发。

- **语言**：Python 3.10
- **平台**：Windows
- **架构**：模块化任务调度系统，核心链路 `Config → Device → Task`
- **许可证**：GPL-3.0

### 项目结构速览

```
OnmyojiAutoScript/
├── module/          ← 核心框架层（base, config, device, gui, handler, ocr, daemon...）
├── tasks/           ← 50+ 游戏任务模块（每个任务继承 BaseTask）
├── config/          ← 配置模板和数据
├── assets/          ← 游戏 UI 资源（图片、OCR 模板等）
├── deploy/          ← 部署和打包相关
├── .agents/         ← Agent 协同开发知识库（详细见下方）
├── .claude/         ← Claude Code 配置（技能和命令）
└── openspec/        ← 规格驱动开发流程（propose → apply → archive）
```

## 2. 开发环境

### 环境搭建

```bash
# 推荐使用 Python 3.10 虚拟环境
python -m venv venv
venv\Scripts\activate
pip install -r requirements.txt
```

### 常用命令

```bash
# 运行主程序
python main.py

# 运行指定任务模块
python main.py --task <TaskName>
```

## 3. 协同开发指引

### 知识导航（.agents/）

Agent 进入项目后，通过以下路径定位知识文档：

```
AGENTS.md → .agents/INDEX.md → docs/INDEX.md → 具体类目/INDEX.md → 目标文档
```

| 资源 | 路径 | 说明 |
|------|------|------|
| 文档库 | `.agents/docs/` | 知识库、开发日志、踩坑记录、编码规范 |
| 参考项目 | `.agents/reference-projects/` | 外部参考项目索引，代码按需拉取 |
| 本地资源 | `.agents/resource/` | 本地参考资源索引 |

### 需求沟通流程（openspec/）

本项目使用 OpenSpec 进行"先对齐，再开发"的需求沟通：

```
/opsx:explore  → 探索需求，对齐理解
/opsx:propose  → 生成 proposal + design + specs + tasks
/opsx:apply    → 按 tasks 逐项实现
/opsx:archive  → 完成后归档
```

### 编码规范入口

详见 `.agents/docs/rules/INDEX.md`

## 4. Agent 行为准则

### 应当做

- 修改代码前先阅读相关规格和设计文档
- 遵循现有的代码组织和命名习惯
- 完成任务后更新 tasks.md 的 checkbox 状态
- 新增文档时同步更新对应的 INDEX.md

### 不应当做

- 不修改 `.claude/` 配置（除非任务明确要求）
- 不修改 `config/` 下的用户配置文件
- 不提交调试代码或临时文件

### 需要确认的边界

- `git commit` / `git push`：需用户明确许可
- 修改 `.gitignore`：需用户明确许可
- 安装新依赖：需用户明确许可
- 参考项目 `git clone`：需用户明确许可
