## Why

当前任务调度系统只支持「相对间隔」（`success_interval`/`failure_interval`）和「每日单个固定时刻」（`server_update`），无法用纯配置表达「每天在多个固定时刻运行」。像悬赏封印（每天5:00和18:00刷新）这类任务只能在 `next_run()` 中硬编码时间逻辑，每个有此需求的任务都要重复实现一套时间计算。需要将「每日多时刻」提升为调度器的原生能力，让配置即意图。

## What Changes

- 在 `Scheduler` 配置模型中新增 `daily_run_times: list[Time]` 字段，支持配置一个或多个每日运行时刻
- 在 `Scheduler` 配置模型中新增 `daily_run_offset: Time` 字段，支持配置提前量（如提前25分钟运行）
- 修改 `task_delay()` 调度逻辑：当 `daily_run_times` 非空且任务成功时，自动计算列表中最近的下一个未来时刻作为 `next_run`
- 重构 `WantedQuests` 任务：删除硬编码的时间判断逻辑，改为读取配置中的 `daily_run_times`，并提供兜底默认值
- 向后兼容：`daily_run_times` 默认为空列表，所有现有任务行为不变

## Capabilities

### New Capabilities

- `daily-run-times-scheduling`: 调度器支持每日多时刻配置，在 `Scheduler` 中通过 `daily_run_times` 和 `daily_run_offset` 表达固定时刻的调度意图

### Modified Capabilities

无。此变更不影响任何已有 spec。

## Impact

- **配置模型**: `tasks/Component/config_scheduler.py` — `Scheduler` 新增两个字段
- **调度逻辑**: `module/config/config.py` — `task_delay()` 新增每日时刻计算分支
- **任务重构**: `tasks/WantedQuests/script_task.py` — 删除硬编码的 `next_run()`，改为软迁移（读配置 + 兜底默认值）
- **任务配置**: `tasks/WantedQuests/config.py` — `before_end` 字段标记废弃
- **用户配置**: `config/*.json` — 无需手动修改（Pydantic 自动补全新字段默认值）
- **GUI**: 后续需为 `daily_run_times` 和 `daily_run_offset` 渲染对应的输入控件（不在本次范围）
