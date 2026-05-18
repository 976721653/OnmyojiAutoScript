## 1. 诊断日志插桩

- [x] 1.1 在 `script_task.py` 主循环中添加 `I_HFREEZE` 检测日志和 `do_action()` 执行日志
- [x] 1.2 在 `agent/focus.py` 的 `r()` 方法中添加决策因子日志（omega、tau、upsilon、result、特殊惩罚原因）
- [x] 1.3 在 `agent/agent.py` 中激活 `dbg_throw` / `dbg_throw_n` 计数器：每局结束时通过 `hya_info` 开关输出统计
- [x] 1.4 为上述日志统一添加 `hya_info` 开关控制，确保关闭时无性能影响

## 2. Bug1 修复：豆子切换验证

- [x] 2.1 在 `bean_05to10()` 调用后添加验证逻辑：等待 0.5s 后检查豆子数，失败重试一次，两次失败记录 warning
- [x] 2.2 添加切换结果日志（成功/重试成功/失败）

## 3. Bug2 修复：撒豆决策修复

- [x] 3.1 分析诊断日志输出，确定 Bug2 的根因：I_HFREEZE 单帧误匹配导致整局零抛豆（冻结状态下 tracks=[]，完全跳过追踪）；SSR/SP 无概率 UP 时 punishment=-999.0 也导致稀有式神零投豆
- [x] 3.2 实施针对性修复：`_detect_freeze_stable()` 三帧连续确认防误匹配；SSR/SP 惩罚日志透明化（惩罚逻辑保留但可通过日志验证）
