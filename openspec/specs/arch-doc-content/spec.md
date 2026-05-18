## ADDED Requirements

### Requirement: Config 子系统记录

架构文档 SHALL 记录 Config 子系统的以下要素：

- 配置类层次结构（ConfigBase → ConfigModel → Config）
- 配置数据流：JSON 文件 → Pydantic 模型 → 任务访问路径
- 调度器系统：Function 对象、三种排序策略、pending/waiting 队列
- 配置更新机制：自动保存、循环重载、GUI 编辑路径
- 关键自定义类型：TimeDelta、DateTime、Time、Scheduler

#### Scenario: 开发者需要理解配置加载流程

- **WHEN** 开发者需要追踪从 JSON 配置文件到任务中 `self.config.xxx.yyy` 的完整路径
- **THEN** 架构文档中 Config 章节包含数据流图和每个环节的说明

---

### Requirement: Device 子系统记录

架构文档 SHALL 记录 Device 子系统的以下要素：

- 多继承钻石结构（完整 MRO）
- 截图管道：7 种方法的调度与对比
- 触控方法：点击/长按/滑动的多后端实现
- 卡死检测算法：双定时器 + 白名单 + 点击记录
- 平台抽象：PlatformWindows、Handler 模式、模拟器品牌适配
- 守护进程和基准测试

#### Scenario: 开发者需要新增截图方法

- **WHEN** 开发者要在 Device 中新增一种截图实现
- **THEN** 架构文档中 Device 章节说明如何注册新方法、方法调度逻辑、以及需要实现的接口

---

### Requirement: 调度与任务执行记录

架构文档 SHALL 记录调度系统的以下要素：

- Script.loop() 主循环的完整流程
- ScriptRuntimeController 的空闲策略
- 任务生命周期：发现 → 调度 → 执行 → 完成 → 重调度
- 异常恢复机制：3 次失败通知、游戏崩溃重启、服务器维护等待

#### Scenario: 开发者需要理解任务调度优先级

- **WHEN** 开发者需要调试为什么某个任务没被调度执行
- **THEN** 架构文档中调度章节说明了 pending/waiting 队列的构建和排序逻辑

---

### Requirement: Tasks 任务体系记录

架构文档 SHALL 记录 Tasks 层以下要素：

- BaseTask 完整 API 面（30+ 方法的分类说明）
- Component 清单：14 个组件的能力和配置
- 三种任务模式（简单日常、多阶段、页面 FSM）
- 多继承体系：GameUi → BaseTask → Component Mixin → Assets 的继承链
- 共享配置类型（ConfigBase、Scheduler、自定义 Pydantic 类型）

#### Scenario: 开发者需要创建新任务

- **WHEN** 开发者要新增一个游戏任务模块
- **THEN** 架构文档中 Tasks 章节说明了任务目录结构、ScriptTask 模式、可选 Component 和完整示例

---

### Requirement: 页面导航系统记录

架构文档 SHALL 记录页面导航系统的以下要素：

- Page 和 Transition 的数据结构
- 页面有向图的定义方式
- Dijkstra 最短路径导航算法
- Category 系统和 Session 隔离
- 自动页面发现机制

#### Scenario: 任务执行需要跳转页面

- **WHEN** 任务的 run() 方法调用 `self.ui_goto(page_target)`
- **THEN** 架构文档中导航章节说明了从当前页面到目标页面的完整导航流程

---

### Requirement: Atom 检测与 OCR 记录

架构文档 SHALL 记录检测系统的以下要素：

- 8 种 Atom 规则类型及其用途
- ROI 两阶段设计（roi_front vs roi_back）
- OCR 引擎架构：PaddleOCR ONNX + zerorpc 多进程服务
- OcrMode 枚举的 6 种识别模式
- 图像匹配的 3 种方法（模板匹配、多尺度、SIFT+FLANN）

#### Scenario: 开发者为新 UI 元素定义检测规则

- **WHEN** 开发者需要为游戏中的新按钮创建检测规则
- **THEN** 架构文档中 Atom 章节说明了应选择哪种规则类型及原因

---

### Requirement: GUI 与 Server 记录

架构文档 SHALL 记录交互层的以下要素：

- GUI：PySide6 + FluentUI QML，进程管理，Python-to-QML 桥接
- Server：FastAPI REST API + WebSocket，四大路由模块（home/script/stats/tool）
- Web 规则标注工具
- 双模式共存架构（本地 GUI vs 远程 Web）

#### Scenario: 开发者需要添加新的 API 端点

- **WHEN** 开发者要在 Web 管理后台新增功能
- **THEN** 架构文档中 Server 章节说明了路由注册方式和 MainManager 生命周期

---

### Requirement: 通知与多人协作记录

架构文档 SHALL 记录通知系统（onepush 多渠道推送、触发条件）和 Team Flow（MQTT 多人协作、策略协商、自愈机制）。

#### Scenario: 需要调试 Team Flow 协作问题

- **WHEN** 多个 OAS 实例协作异常
- **THEN** 架构文档中 Team Flow 章节说明了 MQTT Topic、消息格式和策略协商算法
