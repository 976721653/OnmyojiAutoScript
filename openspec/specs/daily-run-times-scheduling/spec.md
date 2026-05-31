# daily-run-times-scheduling

## Purpose

为调度器提供每日多固定时刻的配置能力，使用户无需修改源码即可表达"每天在 N 个指定时刻运行任务"的调度意图。

## Requirements

### Requirement: Scheduler 支持每日运行时刻列表

`Scheduler` 配置模型 SHALL 包含一个 `daily_run_times` 字段，类型为 `list[Time]`，默认为空列表。用户可通过此字段配置一个或多个每日固定时刻（如 `["05:00", "18:00"]`），表达任务应在这些时刻运行的意图。

#### Scenario: daily_run_times 为空列表

- **WHEN** `daily_run_times` 为空列表
- **THEN** 调度行为与之前完全一致，走 `success_interval` / `failure_interval` 间隔逻辑

#### Scenario: daily_run_times 包含多个时刻

- **WHEN** `daily_run_times` 配置为 `["05:00", "18:00"]`
- **THEN** 调度器识别这两个时刻为任务的每日目标运行时刻

### Requirement: 调度器自动选取最近未来时刻

当 `daily_run_times` 非空且任务成功时，`task_delay()` SHALL 自动从列表中计算最近的下一个未来时刻作为 `next_run`，不再使用 `success_interval`。

#### Scenario: 当前时间在两个时刻之间

- **WHEN** 当前时间为 14:30，`daily_run_times` 为 `["05:00", "18:00"]`，任务成功
- **THEN** `next_run` 设为当天 18:00 + 随机浮动

#### Scenario: 当前时间在所有时刻之后

- **WHEN** 当前时间为 23:00，`daily_run_times` 为 `["05:00", "18:00"]`，任务成功
- **THEN** `next_run` 设为次日 05:00 + 随机浮动

#### Scenario: 当前时间在所有时刻之前

- **WHEN** 当前时间为 03:00，`daily_run_times` 为 `["05:00", "18:00"]`，任务成功
- **THEN** `next_run` 设为当天 05:00 + 随机浮动

### Requirement: 失败重试不受每日时刻影响

当任务失败（`success=False`）时，调度器 SHALL 始终使用 `failure_interval` 计算 `next_run`，不受 `daily_run_times` 配置影响。

#### Scenario: 任务失败后的快速重试

- **WHEN** 任务失败，`failure_interval` 为 `"00 00:30:00"`，`daily_run_times` 为 `["05:00", "18:00"]`
- **THEN** `next_run` 设为 `now + 30 分钟 + 随机浮动`，而不是等待下一个 daily run time

### Requirement: Scheduler 支持每日运行提前量

`Scheduler` 配置模型 SHALL 包含一个 `daily_run_offset` 字段，类型为 `Time`，默认为 `00:00:00`。当 `daily_run_times` 非空时，实际运行时刻为列表中时刻减去此偏移量。

#### Scenario: 配置了提前量

- **WHEN** `daily_run_times` 为 `["18:00"]`，`daily_run_offset` 为 `"00:25:00"`，当前时间为 17:00
- **THEN** `next_run` 设为当天 17:35:00 + 随机浮动

#### Scenario: 未配置提前量

- **WHEN** `daily_run_times` 为 `["18:00"]`，`daily_run_offset` 为 `"00:00:00"`
- **THEN** `next_run` 设为当天 18:00 + 随机浮动

### Requirement: float_time 在每日时刻模式下继续生效

当 `daily_run_times` 非空时，`float_time` SHALL 作为随机浮动窗口应用于计算出的 `next_run`。

#### Scenario: 每日时刻模式下的随机浮动

- **WHEN** `daily_run_times` 为 `["18:00"]`，`float_time` 为 `"00:03:00"`，任务成功
- **THEN** `next_run` 的范围在 18:00:00 到 18:03:00 之间

### Requirement: 向后兼容

新增字段 SHALL 不影响任何已有任务的调度行为。已有 JSON 配置文件无需手动修改即可正常加载运行。

#### Scenario: 旧配置文件加载

- **WHEN** 加载一个不包含 `daily_run_times` 和 `daily_run_offset` 字段的旧 JSON 配置文件
- **THEN** Pydantic 自动填充默认值（空列表和 `00:00:00`），任务按间隔模式正常运行
