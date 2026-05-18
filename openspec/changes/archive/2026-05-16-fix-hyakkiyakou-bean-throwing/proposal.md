## Why

百鬼夜行模块存在两个影响核心功能的 bug：豆子数量偶尔不自动切换，以及整局完全无撒豆操作导致零收获。决策链中存在多处日志盲区（`I_HFREEZE` 检测结果、`Focus.r()` 计算过程、`bean_05to10()` 执行结果均无记录），无法定位根因。需要先插桩照亮决策链，再根据日志修复。

## What Changes

- **修改** `script_task.py`：在 `I_HFREEZE` 检测、`bean_05to10()` 调用后、`do_action()` 执行处加诊断日志
- **修改** `agent/focus.py`：在 `decision()` 和 `r()` 方法中加决策因子日志（omega、tau、upsilon、result、throw）
- **修改** `agent/agent.py`：在每局结束时输出 `dbg_throw` / `dbg_throw_n` 计数器
- **修复 Bug1**：`bean_05to10()` 调用后验证豆子是否确实切换到 10，失败则重试
- **修复 Bug2**：根据日志定位根因后修复（候选：`I_HFREEZE` 误匹配阈值、SSR/SP 惩罚逻辑过严、`r()` 阈值参数）

## Capabilities

### New Capabilities

- `hyakkiyakou-diagnostics`：百鬼夜行模块的诊断日志能力，覆盖决策链全部关键节点

### Modified Capabilities

_无_

## Impact

- 修改文件：`tasks/Hyakkiyakou/script_task.py`、`tasks/Hyakkiyakou/agent/focus.py`、`tasks/Hyakkiyakou/agent/agent.py`
- 不影响其他任务模块
- 不修改 AI 模型或外部依赖
