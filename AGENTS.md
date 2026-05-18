# AGENTS.md — Agent 协同开发入口

## 1. 项目概述

OnmyojiAutoScript（OAS）是阴阳师手游的自动化脚本，基于 [ALAS](https://github.com/LmeSzinc/AzurLaneAutoScript) 框架思想开发。

- **语言**：Python 3.10
- **平台**：Windows 优先
- **架构**：模块化任务调度系统，核心链路 `Config → Device → Task`
- **许可证**：GPL-3.0
- **主要能力**：自动执行阴阳师日常、周常、阴阳寮、副本、限时活动等任务

当前仓库是用户从上游 fork 后维护的工作副本：

- **origin**：`https://github.com/976721653/OnmyojiAutoScript.git`
- **upstream**：`https://github.com/AzurTian/OnmyojiAutoScript.git`
- **当前工作分支**：`mine`，跟踪 `origin/mine`
- **协作原则**：将 `upstream` 视为上游，将 `origin/mine` 视为用户工作分支；未经用户明确许可，不执行 `fetch` / `merge` / `commit` / `push`

## 2. 项目结构速览

```
OnmyojiAutoScript/
├── script.py        ← 调度器主入口，动态加载并执行任务
├── server.py        ← FastAPI Web 管理服务入口
├── gui.py           ← PySide6/QML 本地 GUI 入口
├── module/          ← 核心框架层（config, device, server, gui, ocr, image, atom...）
├── tasks/           ← 50+ 游戏任务模块和复用组件
├── config/          ← 配置模板和用户配置 JSON
├── assets/          ← 游戏 UI 资源、图片、OCR 模板等
├── deploy/          ← 部署、打包、环境管理相关代码
├── dev_tools/       ← 资源提取、模板更新等开发工具
├── fluentui/        ← GUI 相关资源
├── .agents/         ← Agent 协同开发知识库
├── .claude/         ← Claude Code 配置（非明确要求不修改）
└── openspec/        ← 规格驱动开发流程（propose → apply → archive）
```

## 3. 核心架构地图

### 3.1 Config 配置系统

核心文件：

- `module/config/config.py`
- `module/config/config_model.py`
- `module/config/scheduler.py`
- `tasks/Component/config_scheduler.py`

关键点：

- `Config` 持有 `ConfigModel`，通过 `__getattr__` 代理访问配置模型
- `ConfigModel` 是 Pydantic 模型树，包含所有任务配置
- 配置持久化到 `config/<name>.json`
- 每个可调度任务通常包含 `scheduler`：
  - `enable`
  - `next_run`
  - `priority`
  - `success_interval`
  - `failure_interval`
  - `server_update`
  - `delay_date`
  - `float_time`

### 3.2 Script 调度执行

核心文件：

- `script.py`
- `module/script/runtime_controller.py`

主链路：

```
Config.update_scheduler()
  → Config.get_next()
  → Script.get_next_task()
  → ScriptRuntimeController.prepare_task_execution()
  → Script.run()
  → 动态加载 tasks/<TaskName>/script_task.py::ScriptTask
  → ScriptTask.run()
  → set_next_run()
  → raise TaskEnd
```

调度策略：

- `FILTER`：按预定义规则排序
- `FIFO`：按 `next_run` 排序，`Restart` 优先
- `PRIORITY`：按优先级分组，组内 FIFO

`ScriptRuntimeController` 负责空闲策略、模拟器预热、游戏恢复、`Restart` 恢复和服务器维护窗口延后。

### 3.3 Device 设备抽象

核心文件：

- `module/device/device.py`

`Device` 采用多继承组合：

```
Device(Platform, Screenshot, Control, AppControl)
```

职责：

- 模拟器生命周期管理
- ADB / uiautomator2 连接
- 多后端截图
- 点击、长按、滑动
- 游戏启动/停止
- 卡死检测和重复点击检测

常见截图后端包括 `adb`、`uiautomator2`、`droidcast`、`scrcpy`、`nemu_ipc`、`window_background`。

### 3.4 Task 任务体系

核心文件：

- `tasks/base_task.py`
- `tasks/<TaskName>/script_task.py`
- `tasks/<TaskName>/config.py`
- `tasks/<TaskName>/assets.py`

任务目录通常固定为：

```
tasks/<TaskName>/
├── script_task.py    ← class ScriptTask(...)
├── config.py         ← Pydantic 配置模型
└── assets.py         ← 自动生成的资源规则
```

任务类固定命名为 `ScriptTask`，入口是无参 `run()`。典型继承顺序：

```
GameUi → Component Mixins → TaskAssets
```

`BaseTask` 提供截图、图像识别、OCR、点击、滑动、列表查找、奖励领取、任务重调度等基础 API。

### 3.5 页面导航、Atom、OCR

相关目录：

- `tasks/GameUi/`
- `module/atom/`
- `module/image/`
- `module/ocr/`

关键点：

- `GameUi` 提供页面识别和 Dijkstra 页面导航
- `module/atom` 定义 `RuleImage`、`RuleClick`、`RuleOcr`、`RuleSwipe`、`RuleList` 等规则类型
- OCR 基于 `ppocr-onnx`，通过 zerorpc 独立服务调用
- 图像匹配通过 `module/image/rpc.py` 交给图像服务进程处理

### 3.6 GUI 与 Web Server

本地 GUI：

- 入口：`gui.py`
- 技术栈：PySide6 + QML + FluentUI
- 通过进程管理、zerorpc、日志回调与脚本实例通信

Web Server：

- 入口：`server.py`
- 应用：`module/server/app.py`
- 技术栈：FastAPI + WebSocket
- 核心管理器：`module/server/main_manager.py`
- 脚本进程：`module/server/script_process.py`

## 4. 开发环境

### 环境搭建

```bash
python -m venv venv
venv\Scripts\activate
pip install -r requirements.txt
```

### 常用入口

```bash
# 启动 Web 管理服务
python server.py

# 启动本地 GUI
python gui.py

# 直接运行调度器调试入口（当前源码默认使用 oas1）
python script.py
```

如果需要运行某个任务，优先查看该任务 `tasks/<TaskName>/script_task.py` 底部是否提供 `__main__` 调试入口。

## 5. 协同开发知识导航

Agent 进入项目后，通过以下路径定位知识文档：

```
AGENTS.md → .agents/INDEX.md → .agents/docs/INDEX.md → 具体类目/INDEX.md → 目标文档
```

| 资源 | 路径 | 说明 |
|------|------|------|
| 架构知识库 | `.agents/docs/knowledge/` | 项目架构、模块设计、技术原理 |
| 开发日志 | `.agents/docs/dev-logs/` | 开发过程、决策上下文 |
| 踩坑记录 | `.agents/docs/pitfalls/` | 常见问题和注意事项 |
| 编码规范 | `.agents/docs/rules/` | 命名、导入、任务结构等规范 |
| 参考项目 | `.agents/reference-projects/` | 外部开源项目索引 |
| 本地资源 | `.agents/resource/` | 本地参考资源索引 |

重点文档：

- `.agents/docs/knowledge/project-architecture.md`
- `.agents/docs/rules/coding-standards.md`

## 6. OpenSpec 需求沟通流程

本项目使用 OpenSpec 进行“先对齐，再开发”的需求沟通：

```
/opsx:explore  → 探索需求，对齐理解
/opsx:propose  → 生成 proposal + design + specs + tasks
/opsx:apply    → 按 tasks 逐项实现
/opsx:archive  → 完成后归档
```

适用场景：

- 新增较大功能
- 改动调度器、配置系统、设备层等核心架构
- 涉及多文件、多模块协作的任务

小型 bugfix 或明确的单点修改，可以直接阅读相关代码后实施。

## 7. 编码规范与任务约定

### 导入与命名

- 使用项目根目录开始的绝对导入
- 不使用相对导入
- 任务类固定命名为 `ScriptTask`
- 方法和配置字段使用 `snake_case`
- 资源命名前缀：
  - `I_`：Image
  - `C_`：Click
  - `O_`：Ocr
  - `L_`：List

### 任务实现约定

- `run()` 是任务唯一入口，无参数
- 正常结束时调用 `self.set_next_run(...)`
- 正常结束通过 `raise TaskEnd(...)` 通知调度器
- 不在任务主循环中捕获 `TaskEnd`
- 任务配置通过 `self.config.<task_name>.<section>.<field>` 访问

### 文件修改约定

- `tasks/*/assets.py` 通常由工具生成，不手动修改，除非任务明确要求
- `config/` 下用户配置文件不随意修改
- 新增文档时同步更新对应 `INDEX.md`
- 完成 OpenSpec 任务后更新 `tasks.md` checkbox

## 8. Agent 行为准则

### 应当做

- 修改前先定位权威实现和相关文档
- 遵循现有架构、命名和多继承模式
- 对核心链路改动保持保守，优先做最小必要修改
- 修改调度、设备、配置、页面导航时同步检查调用方
- 完成后说明修改范围、验证方式和风险点

### 不应当做

- 不修改 `.claude/` 配置，除非任务明确要求
- 不随意修改 `config/` 下用户配置文件
- 不手动改自动生成的 `assets.py`，除非任务明确要求
- 不提交调试代码、临时文件或一次性脚本
- 不在未确认的情况下引入新依赖

### 需要用户确认的边界

- `git commit` / `git push`
- `git fetch` / `git merge` / `git rebase`
- 修改 `.gitignore`
- 安装新依赖
- 参考项目 `git clone`
- 对上游同步、分支整理、历史改写等 Git 操作

## 9. 开发时优先检查清单

处理任务前优先确认：

- 当前分支和工作区状态
- 是否涉及用户配置文件
- 是否涉及生成文件
- 是否已有 OpenSpec change
- 是否需要更新 `.agents/docs/` 或相关 `INDEX.md`
- 是否会影响 `Config → Device → Task` 主链路

常用只读检查：

```bash
git status --short --branch
git branch -vv
git remote -v
```
