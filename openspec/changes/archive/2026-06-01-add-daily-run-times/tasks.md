## 1. 配置模型扩展

- [x] 1.1 在 `tasks/Component/config_scheduler.py` 的 `Scheduler` 类中新增 `daily_run_times: list[Time]` 字段，默认空列表
- [x] 1.2 在 `tasks/Component/config_scheduler.py` 的 `Scheduler` 类中新增 `daily_run_offset: Time` 字段，默认 `00:00:00`
- [x] 1.3 验证 Pydantic 序列化/反序列化正常：空列表不应写入 JSON 造成冗余（可选优化）

## 2. 调度器核心逻辑

- [x] 2.1 在 `module/config/config.py` 的 `task_delay()` 方法中增加 `daily_run_times` 非空时的分支判断
- [x] 2.2 实现最近未来时刻计算：遍历 `daily_run_times` 中各时刻，对每个时刻生成今天和明天的候选时间，只保留大于当前时间的，取最小值
- [x] 2.3 实现 `daily_run_offset` 偏移逻辑：候选时间减去偏移量
- [x] 2.4 实现 `float_time` 随机浮动在每日时刻模式下的应用
- [x] 2.5 确保 `success=False`（失败重试）时不走每日时刻逻辑，仍使用 `failure_interval`

## 3. WantedQuests 软迁移

- [x] 3.1 修改 `tasks/WantedQuests/script_task.py` 的 `next_run()` 方法：从 `self.config` 读取 `daily_run_times`，为空时兜底默认值 `[time(5,0), time(18,0)]`
- [x] 3.2 将原有的时间判断逻辑（5:00/18:00 三个分支）中的硬编码时刻替换为从 `daily_run_times` 读取
- [x] 3.3 优先读 `daily_run_offset`，为 `00:00:00` 时回退读 `before_end`
- [x] 3.4 在 `tasks/WantedQuests/config.py` 的 `WantedQuestsConfig` 中对 `before_end` 字段添加注释标记为废弃

## 4. 验证

- [x] 4.1 验证旧配置文件（不含 `daily_run_times` 和 `daily_run_offset`）可正常加载，所有现有任务调度行为不变
- [x] 4.2 验证新配置 `daily_run_times: ["05:00", "18:00"]` 的调度逻辑正确性（手动时间推算或单元测试）
- [x] 4.3 验证 WantedQuests 在无 `daily_run_times` 配置时使用兜底默认值，行为与原硬编码一致
