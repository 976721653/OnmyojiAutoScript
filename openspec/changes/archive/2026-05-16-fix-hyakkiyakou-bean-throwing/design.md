## Context

百鬼夜行模块的决策链从截图到最终抛豆经过多层判断：`I_CHECK_RUN` → `I_HFREEZE` → AI 模型追踪 → Agent 热力图 → Focus.r() 评分。当前这五层中，第二层（FREEZE 检测）和第五层（r() 评分）完全没有日志输出，其余层也只有 `hya_info` 开关控制的部分信息。两个报告的 bug 恰好落在这两个盲区。

此外，`agent.py` 中定义了 `dbg_throw` / `dbg_throw_n` 计数器但从未读取，存在闲置的诊断基础。

## Goals / Non-Goals

**Goals:**
- 在 `I_HFREEZE` 检测、`bean_05to10()` 执行、`Focus.r()` 计算、`do_action()` 执行处添加结构化日志
- 激活 `dbg_throw` / `dbg_throw_n` 计数器，每局结束时输出
- 修复 `bean_05to10()`：调用后验证结果，失败重试
- 修复 Bug2：根据日志分析，调整 FREEZE 匹配阈值或 r() 惩罚参数

**Non-Goals:**
- 不改动 AI 模型（oashya）的检测逻辑
- 不改变配置文件格式
- 不重构整个模块架构

## Decisions

### 1. 日志粒度：使用 `logger.info` 而非 `DEBUG`

**决策**：诊断日志使用 `logger.info` 级别，受 `hya_info` 配置开关控制。

**理由**：日志器级别为 INFO，DEBUG 消息会被丢弃。且已有 `hya_info` 开关被 Agent 和 ScriptTask 引用，用同一个开关可让用户无需修改 logger 级别即可看到诊断信息。

### 2. I_HFREEZE 检测日志

**决策**：在 `script_task.py` 第 258 行分支前后加日志，记录 FREEZE 检测结果和 `I_HFREEZE` 匹配的坐标（若有）。

```python
if not self.appear(self.I_HFREEZE):
    if hya_info: logger.info(f'[Frame] FREEZE=False, proceeding with tracking')
    ...
else:
    if hya_info: logger.info(f'[Frame] FREEZE=True, skipping tracking')
    tracks = []
```

### 3. Focus.r() 决策因子日志

**决策**：在 `r()` 方法中，当 `hya_info` 开关开启时，记录计算中间值。

```python
if debug_info:
    logger.info(f'[Decision] omega={omega:.4f} tau={tau:.4f} upsilon={upsilon:.4f} '
                f'result={result:.4f} throw={result > 0} target={target_id}')
```

额外记录三个特殊惩罚的触发：
- SSR/SP 无概率 UP → 日志标记 `[Decision] SSR/SP blocked: no prob_up buff`
- FREEZE 状态 → `[Decision] blocked: freeze state`
- 正常跳过 → `[Decision] skipped: r={result} <= 0`

### 4. bean_05to10() 验证

**决策**：调用后等待 0.5 秒，截图检查当前豆子数量是否为 10。如果不是，重试一次。两次都失败则记录 warning。

### 5. dbg_throw 计数器激活

**决策**：在 `one()` 方法结束时（每局结束后），输出 `dbg_throw`（投豆次数）和 `dbg_throw_n`（跳过次数）。这给出一局中的总决策次数和投豆率。

### 6. Bug 修复策略

**决策**：先加日志，让用户跑一次获取日志输出，根据日志确认根因后再修。不猜测修。

可能的修复方案预置：
- FREEZE 误匹配 → 提高 `I_HFREEZE` 匹配阈值或改用多帧确认
- SSR/SP 惩罚过严 → 将 `-999.0` 改为可配置的惩罚值，或放宽"概率 UP"条件
- `r()` 阈值 0 太严 → 若日志显示大量 `-0.1 ~ -0.01` 的近零值，可考虑将阈值降至 -0.2

## Risks / Trade-offs

- **日志量**：`hya_info` 开启后每帧可能输出多条日志，在高频（100ms/帧）下日志量较大。→ 使用 `hya_info` 开关控制，默认关闭
- **性能影响**：`logger.info` 相比纯计算有 I/O 开销。→ 仅在开关开启时才执行，对正常使用零影响
- **修复不确定性**：Bug2 的根因需等日志才能确认。→ 预设多套修复方案，日志出来后匹配最近的那个
