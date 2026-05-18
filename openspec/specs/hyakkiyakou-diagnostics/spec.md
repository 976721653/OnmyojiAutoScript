## ADDED Requirements

### Requirement: I_HFREEZE 状态日志

在每帧的撒豆循环中，系统 SHALL 记录 `I_HFREEZE` 的检测结果。当 `hya_info` 配置为 True 时，日志输出 SHALL 包含冻结状态和当前帧的追踪是否被跳过。

#### Scenario: 冻结状态激活

- **WHEN** `I_HFREEZE` 被匹配到且 `hya_info` 开启
- **THEN** 日志输出 `[Frame] FREEZE=True, skipping tracking`

#### Scenario: 冻结状态未激活

- **WHEN** `I_HFREEZE` 未被匹配到且 `hya_info` 开启
- **THEN** 日志输出 `[Frame] FREEZE=False, proceeding with tracking`

---

### Requirement: Focus.r() 决策因子日志

在 `Focus.r()` 方法中，当 `hya_info` 开关开启时，系统 SHALL 输出决策因子的中间值：omega（效用评分）、tau（时间冷却）、upsilon（豆子进度比）和最终的 result 值。

#### Scenario: 正常决策日志

- **WHEN** `r()` 被调用且 `hya_info` 开启且无特殊惩罚触发
- **THEN** 日志包含 `[Decision]` 前缀和 omega、tau、upsilon、result 的具体数值

#### Scenario: SSR/SP 惩罚触发

- **WHEN** 追踪目标为 SSR 或 SP 稀有度且"概率 UP"buff 未激活
- **THEN** 日志输出 `[Decision] SSR/SP blocked: no prob_up buff`，result 强制为 -999.0

#### Scenario: FREEZE 惩罚触发

- **WHEN** 全局冻结标志为 True
- **THEN** 日志输出 `[Decision] blocked: freeze state`，result 强制为 -999.0

#### Scenario: 分数不足跳过

- **WHEN** `r()` 返回值 ≤ 0 且不属于上述惩罚情况
- **THEN** 日志输出 `[Decision] skipped: r=<value> <= 0`

---

### Requirement: bean 切换验证

`bean_05to10()` 调用后，系统 SHALL 等待 0.5 秒后检查当前豆子数量是否已切换到 10。若未成功切换，SHALL 重试一次。两次失败后记录 warning 日志。

#### Scenario: 切换成功

- **WHEN** `bean_05to10()` 执行后检测到豆子数为 10
- **THEN** `hya_info` 开启时日志输出 `[Bean] switched to 10 beans OK`

#### Scenario: 首次失败，重试成功

- **WHEN** 第一次切换后豆子数仍为 5，重试后切换成功
- **THEN** 日志输出 `[Bean] retry OK` 并继续执行

#### Scenario: 两次失败

- **WHEN** 两次切换后豆子数仍是 5
- **THEN** 日志输出 `[Bean] WARNING: failed to switch to 10 beans after retry`，继续执行

---

### Requirement: 每局决策统计

每局百鬼夜行结束时，系统 SHALL 输出本局的投豆/跳过计数统计。Agent 的 `dbg_throw` 和 `dbg_throw_n` 计数器 SHALL 在 `one()` 结束时被读取并记录。

#### Scenario: 一局结束输出统计

- **WHEN** `one()` 方法检测到 `I_HEND` 并准备返回
- **THEN** `hya_info` 开启时日志输出 `[Stats] throws=<dbg_throw> skips=<dbg_throw_n> rate=<throw_pct>%`
