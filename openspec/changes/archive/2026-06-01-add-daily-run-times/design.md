## Context

当前调度系统通过 `task_delay()` 计算任务的 `next_run`。其核心流程为：

```
set_next_run(task, success, target)
  → task_delay()
    → target 有值 → nearest_future(targets)
    → 无 target → start_time + interval
    → server_update != 09:00 → 覆盖为"明天+server_update时刻"
    → + random(0, float_time)
```

两个问题：
1. `server_update` 只能表达单个每日时刻，且有隐式哨兵值（`09:00:00` = 关闭）
2. 缺乏"每日多时刻"的原生支持，WantedQuests / FrogBoss / Dokan 各自硬编码

## Goals / Non-Goals

**Goals:**
- 为 `Scheduler` 增加通用的「每日运行时刻列表」配置能力
- 调度层根据配置自动计算最近未来时刻
- 重构 WantedQuests 为软迁移示例（读配置 + 兜底默认值）
- 保持完全向后兼容

**Non-Goals:**
- 不引入 cron 表达式或其他复杂调度语法
- 不迁改 FrogBoss / Dokan / FindJade 的调度逻辑（它们有更复杂的动态行为）
- 不改动 GUI 渲染逻辑
- 不修改 `server_update` 的现有行为

## Decisions

### 1. 新增字段放在 Scheduler 模型中

**选择**：在 `tasks/Component/config_scheduler.py` 的 `Scheduler` 类中新增两个字段。

```python
daily_run_times: list[Time] = Field(default_factory=list, description='...')
daily_run_offset: Time = Field(default=Time(hour=0, minute=0, second=0), description='...')
```

**理由**：所有任务的 scheduler 都使用同一个 `Scheduler` 模型，不需要单独配置。字段默认空列表保证向后兼容。

**备选方案**：在 `ConfigModel` 中新增一个全局调度配置。不采纳——这不是全局行为，而是每个任务各自的属性。

### 2. `task_delay()` 中的调度分支逻辑

**选择**：在 `module/config/config.py` 的 `task_delay()` 中，增加一个判断分支：

```
if daily_run_times 非空 and success is True:
    offset = -daily_run_offset
    candidates = []
    for t in daily_run_times:
        # 今天 + 明天各生成一次, 只保留 > now 的
        candidates.append(...)
    next_run = min(candidates) + random(0, float_time)
    # 不再看 interval
else:
    # 现有逻辑不变
```

**关键设计点**：
- 只在 `success=True` 时走每日时刻逻辑。`success=False`（失败重试）始终走 `failure_interval`，保证能快速重试
- `daily_run_times` 非空时与 `server_update` 互斥——前者优先。逻辑上 `daily_run_times` 替代 `server_update` 的"每日定时"语义
- `float_time` 继续生效，用于打散多设备同时运行造成的峰值

### 3. WantedQuests 软迁移策略

**选择**：保留 `next_run()` 方法，但从硬编码改为读配置 + 兜底：

```python
def next_run(self):
    scheduler = self.config.task_delay  # 实际应从 task 对象取 scheduler
    daily_times = scheduler.daily_run_times
    if not daily_times:
        daily_times = [time(5, 0), time(18, 0)]  # 兜底默认值

    before_end = self.get_config().before_end
    # ...用 daily_times 替代硬编码的 time(hour=5) 和 time(hour=18) ...
    self.set_next_run(task='WantedQuests', target=next_run_datetime)
```

**理由**：现有用户的配置没有 `daily_run_times`，兜底默认值保证行为不变。新用户可以自定义时间。

**备选方案**：直接删除 `next_run()`，完全依赖调度层。不采纳——`before_end` 偏移逻辑仍在任务层（`daily_run_offset` 是调度层的提前量，但 WantedQuests 的 `before_end` 字段在任务配置里，暂不删除，保持软兼容）。

### 4. `daily_run_offset` 与 `before_end` 的关系

**选择**：两者共存，`daily_run_offset` 是调度层的通用字段，`before_end` 是 WantedQuests 自己的历史字段。

在 WantedQuests 的 `next_run()` 中优先使用 `daily_run_offset`，为空时回退读 `before_end`。长期方向是废弃 `before_end`，但本次不强制迁移。

### 5. 时间计算算法

```
输入: daily_run_times: [T1, T2, ...], offset: Delta, now: datetime

candidates = []
today = now.date()
for T in daily_run_times:
    for day_offset in [0, 1]:  # 今天 + 明天
        dt = datetime.combine(today + timedelta(days=day_offset), T) - offset
        if dt > now:
            candidates.append(dt)

next_run = min(candidates) + random(0, float_seconds)
```

`delay_date` 字段在此模式下不生效（`delay_date` 的语义是"从现在起延后 N 天"，与固定时刻模式冲突）。

## Risks / Trade-offs

- **[风险] `daily_run_times` 与 `success_interval` 的值不一致**：当用户在 GUI 同时设置了两个字段时，`daily_run_times` 优先。→ 缓解：后续 GUI 应在 `daily_run_times` 非空时置灰 interval 字段。
- **[风险] `before_end` 被 `daily_run_offset` 意外覆盖**：WantedQuests 软迁移期间两个字段共存。→ 缓解：优先读 `daily_run_offset`，为 `00:00:00` 时回退读 `before_end`。
- **[权衡] FrogBoss / Dokan 仍需硬编码**：它们的调度不是简单的固定时刻列表，而是有时段映射逻辑。→ 本次不做变更，但将来可以把 `daily_run_times` 作为它们的基础数据源。
- **[权衡] 不引入 cron**：对阴阳师场景来说 cron 过重。如果未来真有"每周三19:00"的需求，可以再加 `weekly_run_times` 等字段，仍比 cron 简单。

## Migration Plan

1. 部署：代码改动全部向后兼容，可直接合并部署
2. 回滚：字段默认空列表，回滚代码后配置行为自动恢复
3. 后续迁移：FrogBoss / Dokan 等任务可逐步将硬编码时间改为读 `daily_run_times`，每次迁移一个任务
