# 文档名称：编码规范

| 属性 | 内容 |
|------|------|
| 类型 | 规范 |
| 状态 | 草稿 |
| 更新 | 2026-05-14 |
| 标签 | #编码规范 #python #命名 #导入 |
| 摘要 | OAS 项目的 Python 编码规范，涵盖导入风格、命名约定、文件组织、任务模式和使用模式 |

---

## 导入规范

- **统一使用绝对导入**，从项目根目录开始。不使用相对导入（`from .module import ...`）。
- 项目根目录为 Python path 根，`module/` 和 `tasks/` 均为顶层包。

```python
# 正确
from module.exception import TaskEnd
from module.config.config import Config
from module.logger import logger
from tasks.GameUi.game_ui import GameUi

# 错误
from ..module.exception import TaskEnd
```

## 命名约定

| 元素 | 规则 | 示例 |
|------|------|------|
| 任务类 | `ScriptTask`（固定名） | `class ScriptTask(GameUi, ...)` |
| 基类 | PascalCase | `BaseTask` |
| 配置类 | 任务名 PascalCase | `Nian`, `Orochi`, `NianConfig` |
| 资源类 | 任务名 + `Assets` 后缀 | `NianAssets`, `GlobalGameAssets` |
| 组件类 | PascalCase | `GeneralBattle`, `GeneralRoom` |
| 方法名 | snake_case | `run()`, `appear_then_click()` |
| 配置字段 | snake_case | `buff_gold_50_click`, `lock_team_enable` |
| 私有方法 | 下划线前缀 | `_burst()` |
| 资源命名 | 前缀区分类型 | `I_`=Image, `C_`=Click, `O_`=Ocr, `L_`=List |

## 文件组织

每个任务模块在 `tasks/TaskName/` 下，固定三文件结构：

```
tasks/<TaskName>/
├── script_task.py    ← ScriptTask 类，含 run() 方法
├── config.py         ← Pydantic 配置模型
└── assets.py         ← 资源定义（自动生成，禁止手动修改）
```

- `tasks/` 目录下**不使用 `__init__.py`**
- 资源文件 `assets.py` 由 `dev_tools/assets_extract.py` 自动生成，手动修改会在下次生成时丢失

## 任务类结构

```python
class ScriptTask(GameUi, GeneralBattle, SomeComponent, TaskNameAssets):
    def run(self) -> None:
        # 1. 导航到目标页面
        self.ui_get_current_page()
        self.ui_goto(page_team)
        # 2. 读取配置
        con = self.config.task_name.task_config
        # 3. 主循环：截图 → 检测 → 点击
        while 1:
            self.screenshot()
            if self.appear(assets.I_SOMETHING):
                break
            if self.appear_then_click(assets.I_BUTTON, interval=1):
                continue
        # 4. 调度下一个任务
        self.set_next_run(task='TaskName', success=True, finish=False)
        raise TaskEnd('TaskName')
```

- `run()` 是任务唯一入口，无参数，无返回值
- 结束时**必须**调用 `self.set_next_run(...)` + `raise TaskEnd(...)`
- 配置通过 `self.config.<task_name>.<section>.<field>` 访问

## 多继承 / Mixin 模式

任务类通过多继承复用组件行为。继承顺序：`GameUi` → 组件 Mixin → 资源类。

- `GameUi` 提供页面导航（`ui_goto`, `ui_get_current_page`）和 BaseTask 链路
- 组件 Mixin（`GeneralBattle`, `GeneralRoom`, `GeneralInvite` 等）提供可复用方法
- 资源类提供本任务的 `RuleImage`/`RuleClick`/`RuleOcr` 定义

## 日志规范

使用全局 logger，禁止创建新的 logger 实例：

```python
from module.logger import logger

logger.info('任务开始执行')
logger.warning(f'检测到异常情况: {detail}')
logger.error('执行失败')
```

- 关键路径加 `logger.info`，便于追踪任务执行流
- 异常情况用 `logger.warning`，不应中断任务
- 致命错误用 `logger.error` 或 `logger.critical`

## 异常处理

- 正常结束：`raise TaskEnd('TaskName')`
- 配置错误：`raise ScriptError(f'...')`
- 页面识别异常：`raise GamePageUnknownError`
- 致命错误需人工介入：`raise RequestHumanTakeover`

不要在任务主循环中捕获 `TaskEnd`——它由调度器捕获。
