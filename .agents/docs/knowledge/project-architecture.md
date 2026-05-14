# 文档名称：项目架构

| 属性 | 内容 |
|------|------|
| 类型 | 知识库 |
| 状态 | 草稿 |
| 更新 | 2026-05-14 |
| 标签 | #架构 #module #tasks #BaseTask |
| 摘要 | OAS 整体架构：module 核心框架层与 tasks 任务层的分离设计，Config-Device-Task 核心链路，组件系统和页面导航机制 |

---

## 整体架构

```
┌─────────────────────────────────────────────────────────────┐
│                      tasks/  任务层                          │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐       │
│  │  Nian    │ │  Orochi  │ │Exploration│ │  ...     │       │
│  │ (ScriptTask)│(ScriptTask)│(ScriptTask)│          │       │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └──────────┘       │
│       │            │            │                            │
│       └────────────┼────────────┘                            │
│                    │  继承 Component Mixin                   │
├────────────────────┼────────────────────────────────────────┤
│              module/  核心框架层                              │
│  ┌─────────┐ ┌──────┐ ┌──────┐ ┌─────┐ ┌────┐ ┌────────┐  │
│  │ config  │ │device│ │ gui  │ │ ocr │ │map │ │handler │  │
│  │ (配置)  │ │(设备)│ │(界面)│ │(识别)│ │(地图)│ │(调度) │  │
│  └─────────┘ └──────┘ └──────┘ └─────┘ └────┘ └────────┘  │
│  ┌─────────┐ ┌──────┐ ┌──────┐                             │
│  │  base   │ │atom  │ │daemon│                             │
│  │ (基础)  │ │(原子)│ │(守护)│                             │
│  └─────────┘ └──────┘ └──────┘                             │
└─────────────────────────────────────────────────────────────┘
```

- **module/**：核心框架层，提供配置管理、设备控制、OCR 识别、GUI 界面、地图行走、守护进程等基础设施，与具体游戏逻辑无关
- **tasks/**：具体游戏任务，每个任务是一个独立目录，继承 module 提供的基础能力，实现具体游戏逻辑

## 核心链路：Config → Device → Task

```
Config ──────────> Device ──────────> Task
(配置模型)        (设备抽象)          (任务执行)
    │                 │                   │
    │  .config        │  .device          │  .run()
    ▼                 ▼                   ▼
Pydantic          截图 + 点击         调度 + 循环
BaseModel         + 卡死检测          + 导航 + 异常
```

1. **Config**：Pydantic 模型树，`self.config.<task>.<section>.<field>` 访问。每个任务有独立配置类，通过 `Scheduler` 子对象配置调度参数
2. **Device**：设备抽象层，继承 `Platform + Screenshot + Control + AppControl`，提供 `screenshot()`, `click()`, `swipe()`, `long_click()` 和卡死检测
3. **Task**：继承 `BaseTask`（通过 `GameUi` 间接继承），通过 `run()` 执行任务逻辑

`BaseTask` 提供的方法（子类直接使用，无需重写）：
- `screenshot()`, `appear()`, `appear_then_click()` — 截图 + 检测 + 交互
- `wait_until_appear()`, `wait_until_disappear()` — 等待逻辑
- `click()`, `swipe()` — 触控操作
- `ocr_appear()`, `list_find()` — OCR 和列表操作
- `set_next_run()` — 调度下一个任务

## 组件系统（Component）

`tasks/Component/` 下的可复用 Mixin 类，通过多继承注入任务：

| 组件 | 功能 |
|------|------|
| `GeneralBattle` | 战斗循环：绿标队友、等待胜负、处理奖励 |
| `GeneralRoom` | 创建/加入/退出组队房间 |
| `GeneralInvite` | 邀请队友、接受邀请 |
| `GeneralBuff` | 开启/关闭经验金币加成 |
| `SwitchSoul` | 切换式神御魂套装 |
| `GameUi` | 页面检测和导航（所有任务的必经继承路径） |
| `SwitchAccount` | 账号切换 |

## 页面导航系统

页面构成有向图，`GameUi` 负责导航：

```
page_main ──→ page_team ──→ page_soul_zones
    │              │
    └──→ page_exploration
```

- 每个 `Page` 有一个 `check_button`（页面识别锚点）和 `links` 字典（目标 → 按钮映射）
- `ui_get_current_page()` 遍历所有页面，通过 `check_button` 判断当前所在
- `ui_goto(target)` 沿 link 图最短路径逐级点击导航

## 资源系统

每个任务的 `assets.py` 定义 UI 元素的匹配规则：

- `RuleImage` — 模板匹配（图片识别）
- `RuleClick` — 坐标点击（固定位置）
- `RuleOcr` — 文字识别（OCR）
- `RuleList` — 可滚动列表

`assets.py` 由 `dev_tools/assets_extract.py` 自动生成，**禁止手动修改**。命名前缀：`I_`=Image, `C_`=Click, `O_`=Ocr, `L_`=List。

## 调度系统

`scheduler.py` 管理任务队列，根据 `Scheduler` 配置（`enable`, `next_run`, `priority`, `success_interval`, `failure_interval`）调度任务。每个任务结束时通过 `set_next_run()` 注册下次执行时间。

## 异常体系

`module/exception.py` 定义所有异常，均为 `Exception` 子类：

- `TaskEnd`：任务正常完成，由调度器捕获
- `ScriptError`：代码/配置逻辑错误
- `GameStuckError` / `GameBugError`：游戏卡死/异常
- `GamePageUnknownError`：页面识别失败
- `RequestHumanTakeover`：需要人工介入
