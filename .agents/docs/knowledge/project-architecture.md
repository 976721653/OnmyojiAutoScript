# 文档名称：项目架构

| 属性 | 内容 |
|------|------|
| 类型 | 知识库 |
| 状态 | 草稿 |
| 更新 | 2026-05-14 |
| 标签 | #架构 #完整参考 |
| 摘要 | OAS 完整架构参考文档，覆盖 Config、Device、调度执行、Tasks、页面导航、Atom/OCR、GUI/Server、通知与多人协作、异常与日志等全部核心子系统 |

---

## 整体架构

```
┌──────────────────────────────────────────────────────────────────┐
│                        交互层                                     │
│  ┌─────────────────────┐  ┌─────────────────────────────────┐   │
│  │  GUI (PySide6+QML)  │  │  Server (FastAPI + WebSocket)   │   │
│  │  本地客户端           │  │  Web 管理后台 + 规则标注工具     │   │
│  └─────────┬───────────┘  └───────────────┬─────────────────┘   │
│            │                              │                      │
│            └──────────────┬───────────────┘                      │
│                           │                                      │
├───────────────────────────┼──────────────────────────────────────┤
│                     任务执行层                                    │
│  ┌────────────────────────┼──────────────────────────────────┐   │
│  │                   Script (主循环)                           │   │
│  │  ┌──────────┐  ┌─────────────┐  ┌────────────────────┐   │   │
│  │  │ Scheduler │  │ RuntimeCtrl │  │   Task Loader      │   │   │
│  │  │ (调度器)  │  │ (空闲策略)  │  │ (动态加载任务模块)  │   │   │
│  │  └──────────┘  └─────────────┘  └────────────────────┘   │   │
│  └────────────────────────┬─────────────────────────────────┘   │
│                           │                                      │
│  ┌────────────────────────┼─────────────────────────────────┐   │
│  │                   Tasks 层                                 │   │
│  │  ┌──────────────────────────────────────────────────┐    │   │
│  │  │              BaseTask                             │    │   │
│  │  │  screenshot / appear / click / swipe / ocr / ...  │    │   │
│  │  └────────┬─────────────────────────────────────────┘    │   │
│  │           │                                               │   │
│  │  ┌────────┴──────────┬──────────────┬──────────────┐    │   │
│  │  │   Component 复用层                              │    │   │
│  │  │ GeneralBattle │ GeneralInvite │ SwitchSoul │ ... │    │   │
│  │  └────────┬───────┴──────┬───────┴──────┬───────┘    │   │
│  │           │              │              │             │   │
│  │  ┌────────┴──────────────┴──────────────┴──────┐    │   │
│  │  │     50+ ScriptTask (具体游戏任务)             │    │   │
│  │  │     Nian / Orochi / Exploration / ...       │    │   │
│  │  └─────────────────────────────────────────────┘    │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│                      核心框架层 (module/)                      │
│  ┌────────┐ ┌───────┐ ┌─────┐ ┌──────┐ ┌──────┐ ┌───────┐  │
│  │ Config │ │Device │ │ OCR │ │ Atom │ │ Map  │ │ Notify│  │
│  │(配置)  │ │(设备) │ │(识别)│ │(检测)│ │(地图)│ │(通知) │  │
│  └────────┘ └───────┘ └─────┘ └──────┘ └──────┘ └───────┘  │
│  ┌────────┐ ┌───────┐ ┌──────┐ ┌──────────┐ ┌──────────┐   │
│  │  Base  │ │Daemon │ │Logger│ │Exception │ │Team Flow │   │
│  │(基础)  │ │(守护) │ │(日志)│ │(异常体系)│ │(多人协作)│   │
│  └────────┘ └───────┘ └──────┘ └──────────┘ └──────────┘   │
└──────────────────────────────────────────────────────────────┘
```

**核心链路**：`Config(配置模型树) → Device(设备抽象) → Task(游戏逻辑)`

- **Config**：Pydantic 模型树，JSON 持久化，`self.config.<task>.<section>.<field>` 访问
- **Device**：多继承钻石结构，统一截图/触控/卡死检测接口
- **Task**：BaseTask 提供 30+ 交互 API，Component 提供可复用行为，ScriptTask 组装具体逻辑

---

## 1. Config 配置系统

### 1.1 类层次结构

```
ConfigBase (Pydantic BaseModel)           ← tasks/Component/config_base.py
  ├── ConfigModel                          ← module/config/config_model.py
  │     持有所有 50+ 任务配置模型
  │     JSON 序列化/反序列化
  │
  ├── Scheduler                            ← 每个任务的调度配置
  │     enable, next_run, priority
  │     success_interval, failure_interval
  │     server_update, delay_date, float_time
  │
  └── [各任务 Config 类]                    ← 如 Nian, Orochi, Exploration...

ConfigState                                ← 运行时状态
ConfigManual                               ← 硬编码常量和调度优先级
ConfigWatcher                              ← 文件修改时间轮询
ConfigMenu                                 ← GUI 菜单结构
    ↓ (多重继承)
Config                                     ← 主配置类
  持有 ConfigModel 作为 self.model
  提供 get_next() / task_delay() / save() / reload()
```

**设计关键**：`Config` 刻意不从 `ConfigModel` 继承（Pydantic 会接管属性访问），而是通过 `__getattr__` 代理到 `self.model`。

### 1.2 配置数据流

```
磁盘                              运行时
┌──────────────────┐          ┌────────────────────────────┐
│ config/oas.json  │ ──读取──▶│ ConfigModel("oas")          │
│ (嵌套 JSON)       │          │   Pydantic 验证 + 递归构造  │
│                  │          │                             │
│ argument/*.yaml  │          │ Config(config_name)         │
│ (模板/默认值)     │          │   .model = ConfigModel(...) │
└──────────────────┘          │   .get_next() → Function    │
                              │   .task_delay() → 更新调度   │
┌──────────────────┐          │   .save() → 写回 JSON       │
│ config/oas.json  │ ←──写入──│                             │
│ (更新后的配置)    │          └────────────────────────────┘
└──────────────────┘
```

配置是**最终一致性**的：GUI 通过 `ConfigModify` 写入 JSON → Script 下一轮循环重建 Config → 从磁盘重新加载。

### 1.3 调度器

两层调度系统：

| 层 | 位置 | 职责 |
|----|------|------|
| Config 层 | `module/config/config.py` + `scheduler.py` | 构建 Function 队列，按策略排序 |
| Script 层 | `script.py` + `runtime_controller.py` | 执行任务，管理空闲期 |

**三种排序策略**：
- `FILTER`：按预定义的正则优先级字符串排序
- `FIFO`：按 next_run 时间排序（Restart 始终第一）
- `PRIORITY`：按 priority 分组，组内 FIFO，组间按优先级合并

**调度数据流**：`update_scheduler()` 扫描所有启用任务 → 构建 pending（next_run < now）和 waiting（next_run >= now）队列 → `get_next()` 返回 pending[0]

### 1.4 自定义 Pydantic 类型

| 类型 | Python 类型 | 序列化格式 | 用途 |
|------|------------|-----------|------|
| `TimeDelta` | `timedelta` | `"DD HH:MM:SS"` | 任务间隔 |
| `DateTime` | `datetime` | `"YYYY-MM-DD HH:MM:SS"` | 下次执行时间 |
| `Time` | `time` | `"HH:MM:SS"` | 服务器更新时间 |
| `MultiLine` | `str` | 原始字符串 | 多行文本输入 |

---

## 2. Device 设备抽象

### 2.1 多继承钻石结构

```
Device(Platform, Screenshot, Control, AppControl)
  │
  ├── Platform → PlatformWindows → PlatformBase
  │     └── EmulatorManager → Connection → ConnectionAttr
  │          持有: config, serial, adb, u2
  │
  ├── Screenshot
  │     ├── Adb (adb exec-out screencap)
  │     ├── DroidCast (HTTP 截图服务)
  │     ├── Scrcpy (H264 视频流解码)
  │     ├── NemuIpc (MuMu12 C DLL 直接捕获)
  │     └── Window (Win32 GDI BitBlt)
  │
  ├── Control
  │     ├── Minitouch (TCP 协议，贝塞尔曲线轨迹)
  │     ├── Adb (input tap / swipe)
  │     ├── Scrcpy (控制套接字二进制协议)
  │     └── Window (Win32 SendMessage)
  │
  └── AppControl
        ├── Adb (am start/stop, monkey)
        └── Uiautomator2 (u2.app_start/stop)
```

所有路径最终收敛到 `ConnectionAttr`，它提供 `self.config`、`self.serial`、`self.adb`、`self.u2`。

### 2.2 截图管道

7 种截图方法，通过 `config.script.device.screenshot_method` 配置调度：

| 方法 | 机制 | 延迟 |
|------|------|------|
| `ADB` | `adb exec-out screencap -p` → PNG 解码 | 高 |
| `ADB_nc` | netcat 端口转发 + RGBA 解码 | 中 |
| `uiautomator2` | u2 Python API | 中 |
| `DroidCast` | HTTP GET → PNG 解码 | 中 |
| `DroidCast_raw` | HTTP GET → RGB565 按位解码 | 低 (~3ms) |
| `scrcpy` | H264 视频流 → av 解码 | 低 |
| `nemu_ipc` | MuMu12 C DLL 直接捕获 | 最低 |
| `window_background` | Win32 GDI BitBlt | 低 |

**自动基准测试**：当 `screenshot_method='auto'` 时，Device 初始化时会对所有可用方法进行 3 次基准测试，自动选择最快的方法。

### 2.3 触控方法

每种触控操作（点击/长按/滑动）都有 4-5 种后端实现。

**滑动轨迹**：Minitouch 和 Scrcpy 使用三次贝塞尔曲线生成路径，模拟人类手指的自然运动——端点密集、中间稀疏，加入随机偏移。

### 2.4 卡死检测算法

双重定时器 + 白名单机制：

```
screenshot() 时触发 stuck_record_check()
  │
  ├── stuck_timer (60s) 未触发 → 正常
  ├── stuck_timer 触发 + stuck_timer_long (300s) 未触发
  │     └── detect_record 中有白名单项 → 正常（战斗中/登录中等）
  └── 两个定时器都触发 → GameStuckError / GameNotRunningError
```

白名单：`BATTLE_STATUS_S`, `PAUSE`, `LOGIN_CHECK`, `PREPARE_BEFORE_BATTLE`。

点击记录检测：最近 15 次点击中任意按钮 ≥10 次 → `GameTooManyClickError`，防止无休止重复点击。

### 2.5 平台抽象

**PlatformWindows**：完整的 Windows 模拟器生命周期管理。

- **Handler 模式**：每种模拟器家族（Nox、BlueStacks、LDPlayer、MuMu、MEmu）实现 `EmulatorHandler` 接口
- **启动监视状态机**：6 步依次检查（启动确认 → 窗口隐藏 → ADB 设备 → Shell 响应 → 包查询 → 窗口稳定），120 秒超时
- **MuMu12 特殊支持**：最复杂的生命周期，支持 `MuMuManager info` JSON 状态查询、后台模式窗口隐藏

---

## 3. 调度与任务执行

### 3.1 Script 主循环

`Script.loop()` 是系统的心脏，运行在 `script.py` 中：

```
while True:
    1. 检查日期，必要时轮转日志
    2. get_next_task() → 阻塞等待直到有任务可执行
    3. 跳过启动时的第一个 Restart（已被外部调度）
    4. runtime.prepare_task_execution(task) → 确保模拟器和游戏在线
    5. self.run(command) → 动态加载 ScriptTask 并执行
    6. 清理运行状态，更新失败计数
    7. 删除缓存的 Config → 强制下次循环重新加载
```

### 3.2 空闲策略

`ScriptRuntimeController` 管理任务之间的等待期：

| 策略 | 行为 |
|------|------|
| `goto_main` | 停在主界面，保持游戏运行 |
| `close_game` | 关闭游戏，下次任务前重启 |
| `close_emulator_or_goto_main` | 等待超过阈值关闭模拟器，否则回主界面 |
| `close_emulator_or_close_game` | 同上，降级为关闭游戏 |
| `stay_there` | 不做任何操作 |

支持模拟器预热：在 `next_run - startup_lead_time` 前自动重启。

### 3.3 任务生命周期

```
发现 (文件系统) → 调度 (Config.get_next) → 执行 (ScriptTask.run)
    → 完成 (raise TaskEnd) → 重调度 (config.task_delay)
```

**异常恢复**：
- `GameNotRunningError` / `GameStuckError` → 触发 Restart 重启游戏
- 连续 3 次失败 → 推送通知 + 可选关闭模拟器
- `GamePageUnknownError` 在服务器维护窗口 → 延迟所有任务
- 严重错误保存最后 60 张截图到 `./log/error/<timestamp>/`

---

## 4. Tasks 任务体系

### 4.1 BaseTask API

BaseTask 是所有任务的根，提供 30+ 交互方法，按功能分组：

**截图与检测**：
`screenshot()`, `appear()`, `appear_then_click()`, `wait_until_appear()`, `wait_until_disappear()`, `wait_until_stable()`, `wait_animate_stable()`

**触控操作**：
`click()`, `swipe()`, `ocr_appear()`, `ocr_appear_click()`

**列表操作**：
`list_find()`, `list_appear_click()`

**UI 辅助**：
`ui_reward_appear_click()`, `ui_get_reward()`, `ui_click()`, `ui_click_until_disappear()`

**调度**：
`set_next_run()`, `custom_next_run()`

**事件**：
`_burst()`（好友邀请检测和处理）

### 4.2 Component 组件清单

所有可复用组件位于 `tasks/Component/`：

| 组件 | 核心能力 | 配置类 |
|------|---------|--------|
| **GeneralBattle** | 战斗 Page-FSM：准备→战斗中→结果→奖励；绿标队友、预设队伍、持续战斗 | `GeneralBattleConfig` |
| **GeneralRoom** | 房间创建/退出、公/私切换、副本选择 | — |
| **GeneralInvite** | 队长邀请流（OCR 好友名匹配）、队员接受流、房间类型检测 | `InviteConfig` |
| **GeneralBuff** | 经验/金币/御魂/觉醒 buff 开关 | `BuffConfig` |
| **SwitchSoul** | 式神御魂预设切换（按序号/名称） | `SwitchSoulConfig` |
| **SwitchOnmyoji** | 切换出战阴阳师角色 | `Onmyoji` enum |
| **Costume** | 动态替换庭院/战斗/式神录界面素材 | `CostumeConfig` |
| **ReplaceShikigami** | 式神选择和替换（成长界面） | — |
| **Summon** | 抽卡（普通/神秘图案） | — |
| **Buy** | 商店购买（单品/批量） | — |
| **SwitchAccount** | 账号切换 | — |
| **Login** | 登录流程：处理弹窗、选服、选角色 | — |
| **RightActivity** | 右侧活动面板管理 | — |
| **LootStatistics** | 掉落统计（桩） | — |

**继承模型**：每个 Component 继承 BaseTask + 自身的 Assets 类，通过多继承混入 ScriptTask。

### 4.3 三种任务模式

**模式 A：简单顺序任务** (如 Nian)
```
run(): 导航 → 选副本 → 等匹配 → 战斗 → 结束
```
直接继承 GameUi + GeneralBattle + GeneralRoom + GeneralInvite，顺序执行。

**模式 B：多角色任务** (如 Orochi)
```
run(): 预处理 → 分支(UserStatus)
  ├── run_leader():   组队 + 邀请 + 连续战斗
  ├── run_member():   等邀请 + 接战
  ├── run_alone():    单人战斗
  └── run_wild():     公开房间
```
`UserStatus` 枚举（LEADER/MEMBER/ALONE/WILD）驱动完全不同的执行路径。

**模式 C：页面 FSM 任务** (如 Exploration)
```
run(): 匹配当前页面 → 执行页面逻辑 → 转换到下一页面
  page_main      → 收奖励、切阵型、点出战
  page_battle    → run_general_battle()
  page_unknown   → 定时器恢复
```

### 4.4 任务文件结构

每个任务固定三文件：
```
tasks/<TaskName>/
├── script_task.py    ← class ScriptTask(GameUi, ..., TaskNameAssets)
├── config.py         ← Pydantic 配置模型（Scheduler + 特定配置）
└── assets.py         ← 自动生成的原子规则定义（I_*/C_*/O_*）
```

---

## 5. 页面导航系统

### 5.1 核心数据结构

**Page**：
- `recognizer: Matcher` — 页面识别条件（`all_of`/`any_of`/`not_` 组合）
- 属性：`key`/`name`/`category`/`priority`/`cost`
- `transitions: list[Transition]` — 出边
- 钩子：`on_enter_success/failure`、`on_leave_success/failure`

**Transition**：
- `source`/`destination: Page`、`action: ActionLike`、`cost: float`、`key: str`

### 5.2 有向图定义

页面通过 `connect()` 双向链接：

```python
page_main = Page(all_of(I_CHECK_MAIN, ...), category="global")
page_main.connect(page_shikigami_records, I_MAIN_GOTO_SHIKIGAMI_RECORDS)
page_shikigami_records.connect(page_main, I_UI_BACK_YELLOW)
```

### 5.3 Dijkstra 导航

`GameUi.goto_page(destination)`:
1. 识别当前页面（优先缓存，否则扫描允许的 Category）
2. 未知页面处理：尝试 `DEFAULT_UNKNOWN_CLOSERS`
3. Dijkstra 最短路径规划（权重 = page.cost + transition.cost + 边惩罚）
4. 逐步执行 Transition，等待目标页面确认
5. 失败恢复：3 次失败后尝试关闭未知页面；30 秒总超时

### 5.4 Category 与 Session 隔离

- 任务根据文件路径自动推断 Category（如 `tasks/Orochi/` → `orochi`）
- 导航时仅考虑 `当前任务 Category + global + 目标 Category` 内的页面
- 每个任务获得独立的 `NavigatorSession`，从注册表克隆页面定义，允许覆盖

---

## 6. Atom 检测系统与 OCR

### 6.1 Atom 规则类型

| 类型 | 文件 | 用途 |
|------|------|------|
| `RuleImage` | `atom/image.py` | 图像匹配（模板/多尺度/SIFT+FLANN） |
| `RuleClick` | `atom/click.py` | 点击区域定义 |
| `RuleLongClick` | `atom/long_click.py` | 长按区域（扩展 RuleClick + duration） |
| `RuleSwipe` | `atom/swipe.py` | 滑动手势（贝塞尔/线性轨迹） |
| `RuleOcr` | `atom/ocr.py` | OCR 文字检测（多继承所有 sub_ocr 类） |
| `RuleGif` | `atom/gif.py` | 动画检测（匹配多帧 RuleImage） |
| `RuleList` | `atom/list.py` | 可滚动列表（图片/OCR 两种模式） |
| `RuleAnimate` | `atom/animate.py` | 动画稳定性检测（帧间比较） |

### 6.2 ROI 两阶段设计

- **`roi_back`**：搜索区域（在完整截图的哪个范围查找），静态配置
- **`roi_front`**：结果区域（匹配成功后元素在屏幕上的精确位置），动态更新

### 6.3 图像匹配方法

| 方法 | 适用场景 | 特点 |
|------|---------|------|
| 模板匹配 (TM_CCOEFF_NORMED) | 固定尺寸/样式 | 快速，对缩放敏感 |
| 多尺度模板匹配 | 可能的缩放变化 | 0.6-1.2 范围，步长 0.1 |
| SIFT + FLANN | 特征丰富、可能变形 | 鲁棒，计算量大 |

匹配结果通过 `module/image/rpc.py` 的 RPC 客户端发送到图像服务进程处理。

### 6.4 OCR 引擎

**架构**：PaddleOCR ONNX（`ppocronnx`）+ zerorpc 多进程服务

```
OcrRuntime (独立进程)
  ├── ThreadPoolExecutor (OcrTaskScheduler)
  │     └── 每线程一个 TextSystem 实例（threading.local）
  └── zerorpc Server (127.0.0.1:22268)

ModelProxy (客户端)
  └── zerorpc Client → pickle 序列化图像 → 调用远程 OCR
```

**6 种识别模式**：

| 模式 | 返回类型 | 用途 |
|------|---------|------|
| `FULL` | `(x,y,w,h)` 区域 | 大范围多行文字，返回关键字位置 |
| `SINGLE` | `str` | 单行文字精确匹配 |
| `DIGIT` | `int` | 数字（含字符替换容错） |
| `DIGITCOUNTER` | `(current, remain, total)` | 计数器 "14/15" |
| `DURATION` | `timedelta` | 时长 "01:30:00" |
| `QUANTITY` | `int` | 大数字 "6.33亿", "1.2万" |

---

## 7. GUI 与 Server

### 7.1 GUI (本地客户端)

**技术栈**：PySide6 (Qt) + FluentUI (QML Fluent Design 组件库)

**Python-to-QML 桥接**：通过 `QQmlApplicationEngine.set_context_property()` 暴露 Python QObject 到 QML。

| 桥接类 | 职责 |
|--------|------|
| `ProcessManager` | 管理脚本子进程、配置参数、镜像流 |
| `Add` | 配置文件 CRUD |
| `Setting` | 读写 `setting.json` |
| `Utils` | 当前时间、通知测试 |
| `RuleFile` | JSON 规则文件读写 |

**进程管理**：每个脚本实例作为独立 `multiprocessing.Process` 运行，通过 zerorpc（端口 40000-40200）和日志队列与 GUI 通信。

### 7.2 Server (Web 管理后台)

**技术栈**：FastAPI + WebSocket

**四大路由模块**：

| 路由 | 路径 | 功能 |
|------|------|------|
| Home | `/home` | 健康检查、菜单、更新、翻译 |
| Script | `/script` | 配置 CRUD、启停脚本、参数编辑、WebSocket 实时状态 |
| Stats | `/stats` | 日志解析、运行统计、SSE 流 |
| Tool | `/tool` | Web 规则标注工具（截图捕获、规则测试、assets.py 生成） |

**MainManager**：服务器启动时扫描所有配置文件，为每个脚本创建 `ScriptProcess`（`multiprocessing.Process`），通过 Queue/Pipe 通信。

**WebSocket**：`/ws/{script_name}` → `ScriptWSManager` 广播状态和日志到所有连接的 Web 客户端。

### 7.3 双模式共存

```
本地模式: GUI(QML) → ProcessManager → multiprocessing → zerorpc → Script
Web 模式: Browser → FastAPI → MainManager → multiprocessing → Script
```

两种模式共享同一套 Script 进程管理机制。

---

## 8. 通知与多人协作

### 8.1 通知系统

基于 **onepush** 库，支持多渠道：

ServerChan / PushPlus / Bark / Telegram / WeCom Bot / DingTalk / PushDeer / Gocqhttp (QQ) / Gotify / SMTP Email / Custom HTTP

**触发条件**：连续 3 次任务失败 / `GameStuckError` / `GameBugError` / `GamePageUnknownError` / `ScriptError` / `RequestHumanTakeover`

### 8.2 Team Flow 多人协作

基于 **MQTT** 的多人协作系统，用于组队副本（Orochi、FallenSun、EternitySea 等）。

```
MQTT Broker (外部)
  ├── FirstNotice  ← 上线通知
  ├── LastWill     ← 离线遗嘱
  ├── TaskStart    ← 任务开始通知
  └── Strategy     ← 当前任务可用性广播
```

- **Host**：主协调器，组合 Mqtt + Player
- **Player**：持有 username 和 multi_tasks 字典
- **Task**：代表一个协作任务（role、limit_time、limit_count、next_run）
- **策略协商**：每个实例独立根据集体策略数据计算最优调度，自愈（成员加入/离开触发重新计算）

---

## 9. 异常体系

所有异常继承 `Exception`，定义在 `module/exception.py`：

| 异常 | 含义 | 处理方式 |
|------|------|---------|
| `TaskEnd` | 任务正常完成 | 调度器捕获，安排下次执行 |
| `ScriptEnd` | 脚本终止 | 退出主循环 |
| `ScriptError` | 代码/配置逻辑错误 | 记录日志，可能重试 |
| `GameStuckError` | 游戏卡死 | 触发 Restart |
| `GameBugError` | 游戏异常 | 触发 Restart |
| `GameTooManyClickError` | 过度点击 | 触发 Restart |
| `GameNotRunningError` | 游戏未运行 | 启动游戏 |
| `EmulatorNotRunningError` | 模拟器未运行 | 启动模拟器 |
| `GamePageUnknownError` | 未知页面 | 尝试恢复或人工介入 |
| `RequestHumanTakeover` | 需人工介入 | 暂停所有自动化，等待用户 |

---

## 10. 日志系统

`module/logger.py` 创建全局 `logger` 实例（名称为 `'oas'`），三种 Handler：

| Handler | 目标 | 格式 |
|---------|------|------|
| `RichHandler` | 控制台 | Rich 渲染，120 字符宽，彩色 |
| `RichFileHandler` | `./log/<日期>_<脚本名>.txt` | 完整时间戳，无色彩 |
| `FlutterHandler` | GUI 日志回调 | 通过 `set_func_logger(func)` 注入 |

增强方法：`logger.hr()`（水平分隔线）、`logger.attr()`（键值对格式化）、`logger.rule()`（Rich 规则线）

自动清理：7 天前的日志文件和错误目录。
