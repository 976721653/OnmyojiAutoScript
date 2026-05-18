## 1. 核心链路章节

- [x] 1.1 编写整体架构概览（分层图、核心链路 Config→Device→Task 概述）
- [x] 1.2 编写 Config 配置系统章节（类层次、数据流、调度器、更新机制）
- [x] 1.3 编写 Device 设备抽象章节（多继承 MRO、截图管道、触控方法、卡死检测、平台抽象）

## 2. 调度与任务体系章节

- [x] 2.1 编写调度与任务执行章节（Script 主循环、空闲策略、任务生命周期、异常恢复）
- [x] 2.2 编写 Tasks 任务体系章节（BaseTask API 分类、Component 清单、三种任务模式、多继承体系）

## 3. 导航与检测章节

- [x] 3.1 编写页面导航系统章节（Page/Transition 结构、有向图定义、Dijkstra 导航、Session 隔离）
- [x] 3.2 编写 Atom 检测系统与 OCR 章节（8 种规则类型、ROI 两阶段、OCR 引擎、匹配方法）

## 4. 交互层与辅助系统章节

- [x] 4.1 编写 GUI 与 Server 章节（PySide6+FluentUI、FastAPI 路由、Web 工具、双模式架构）
- [x] 4.2 编写通知与多人协作章节（onepush 推送、Team Flow MQTT）
- [x] 4.3 编写异常体系与日志章节

## 5. 收尾

- [x] 5.1 更新 `docs/knowledge/INDEX.md`，同步文档条目和摘要
- [x] 5.2 检查是否需要拆分子文档（无需拆分，最长章节约 100 行，未超过 200 行阈值）
