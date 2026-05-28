# 文档名称：同步上游 mine 分支 SOP

| 属性 | 内容 |
|------|------|
| 类型 | 规范 |
| 状态 | 已验证 |
| 更新 | 2026-05-28 |
| 标签 | #git #上游同步 #mine #sop #agent协作 |
| 摘要 | 规范 Agent 将 upstream/mine 同步到本地 mine，并按需推送 origin/mine 的操作流程与异常处理 |
| 关联 | [编码规范](coding-standards.md) |

---

## 适用场景

当用户明确要求以下任一操作时，使用本 SOP：

- “同步上游 mine 分支到本地 mine 分支”
- “拉取 upstream/mine 并合并到 mine”
- “把上游 mine 更新同步过来”
- 在完成同步后，用户进一步明确要求“推送到远端”

本 SOP 默认仓库关系为：

- `origin`：用户 fork，`https://github.com/976721653/OnmyojiAutoScript.git`
- `upstream`：上游仓库，`https://github.com/AzurTian/OnmyojiAutoScript.git`
- 本地工作分支：`mine`，跟踪 `origin/mine`

## 强制原则

- **没有用户明确许可，不执行** `git fetch`、`git merge`、`git rebase`、`git reset`、`git commit`、`git push`。
- 用户只要求“同步上游”时，只同步到本地 `mine`，**不自动推送** `origin/mine`。
- 用户明确要求“推送”时，才执行 `git push origin mine`。
- 标准同步使用普通 merge，保留历史：`git merge upstream/mine --no-edit`。
- 不使用 `rebase`、`reset --hard`、`push --force`，除非用户明确要求且已说明风险。
- 合并前必须确认工作区干净；如有未提交改动，先暂停并询问用户如何处理。
- 注意保护本仓库的 Agent 协作文档：`AGENTS.md`、`.agents/`、`openspec/`。合并或冲突处理时不要误删。

## 标准流程：同步 upstream/mine 到本地 mine

### 1. 只读检查当前状态

在仓库根目录执行：

```powershell
git status --short --branch
git branch -vv
git remote -v
```

确认：

- 当前分支是 `mine`。
- 工作区没有已修改、已暂存、未跟踪文件。
- `mine` 跟踪 `origin/mine`。
- `upstream` 指向上游仓库。

如果工作区不干净，进入“异常处理：工作区不干净”。

### 2. 拉取上游 mine 引用

仅在用户明确要求同步上游后执行：

```powershell
git fetch upstream mine
```

预期结果：

- `upstream/mine` 更新到最新。
- 命令成功返回，无认证或网络错误。

### 3. 合并前查看差异

```powershell
git log --oneline --left-right --graph HEAD...upstream/mine -30
git diff --stat HEAD..upstream/mine
```

说明：

- `git log HEAD...upstream/mine` 用于查看本地与上游两侧各自新增的提交。
- `git diff HEAD..upstream/mine` 是树差异，不是最终 merge 结果；它可能把本地独有文件显示为删除，不代表 merge 一定会删除这些文件。
- 如果差异涉及 `AGENTS.md`、`.agents/`、`openspec/` 等协作文档，合并后必须额外确认这些文件仍符合预期。

### 4. 执行合并

```powershell
git merge upstream/mine --no-edit
```

预期结果：

- 如果本地和上游均有新增提交，会创建 merge commit。
- 如果本地可快进，Git 可能 fast-forward。
- 如果已经同步，会提示 `Already up to date.`。
- 如发生冲突，进入“异常处理：merge 冲突”。

### 5. 合并后验证

```powershell
git status --short --branch
git log -1 --oneline
git merge-base --is-ancestor upstream/mine HEAD; if ($LASTEXITCODE -eq 0) { 'upstream/mine is ancestor of HEAD' } else { 'upstream/mine is NOT ancestor of HEAD' }
Test-Path AGENTS.md; Test-Path .agents\INDEX.md
```

确认：

- 工作区干净。
- 最新提交是合并提交、快进后的上游提交，或原 HEAD。
- `upstream/mine` 已经是当前 `HEAD` 的祖先。
- `AGENTS.md`、`.agents/INDEX.md` 等本地协作文档仍存在。

### 6. 向用户汇报

同步完成后报告：

- fetch 更新范围，例如 `36df0105..1070a836`。
- 合并方式：merge / fast-forward / already up to date。
- 冲突情况。
- 当前分支状态，例如 `mine...origin/mine [ahead N]`。
- 明确说明是否执行了 push。

## 标准流程：按需推送 origin/mine

仅当用户明确要求“推送到远端”时执行。

### 1. 推送前检查

```powershell
git status --short --branch
git log --oneline origin/mine..HEAD
```

确认：

- 当前分支是 `mine`。
- 工作区干净。
- `origin/mine..HEAD` 中的提交都是准备推送的提交。

### 2. 推送

```powershell
git push origin mine
```

### 3. 推送后验证

```powershell
git status --short --branch
git log -1 --oneline origin/mine
```

确认：

- `mine` 与 `origin/mine` 已同步。
- `origin/mine` 指向预期最新提交。

## 异常处理

### 工作区不干净

表现：

- `git status --short --branch` 显示 `M`、`A`、`D`、`??` 等。

处理：

1. 暂停同步，不执行 merge。
2. 向用户说明当前未提交内容。
3. 让用户选择：提交、暂存、丢弃、或暂不处理。
4. 只有用户明确要求后，才能执行 `git add`、`git commit`、`git stash` 或清理操作。

禁止：

- 不要为了同步自动 `git stash`。
- 不要自动删除未跟踪文件。
- 不要使用 `git reset --hard`。

### 当前不在 mine 分支

表现：

- `git branch -vv` 显示当前分支不是 `mine`。

处理：

1. 如果工作区干净，询问或确认是否切换到 `mine`。
2. 用户确认后执行：

```powershell
git switch mine
```

3. 切换后重新执行只读检查。

如果工作区不干净，先按“工作区不干净”处理。

### 缺少 upstream 远端

表现：

- `git remote -v` 没有 `upstream`。
- `git fetch upstream mine` 报错找不到远端。

处理：

1. 向用户确认是否添加上游远端。
2. 用户确认后执行：

```powershell
git remote add upstream https://github.com/AzurTian/OnmyojiAutoScript.git
```

3. 再执行 `git fetch upstream mine`。

禁止：

- 不要未经确认自动添加或修改远端。

### fetch 失败

常见原因：

- 网络不可用。
- GitHub 访问失败。
- 远端地址错误。
- 权限或认证问题。

处理：

1. 保留错误输出。
2. 检查 `git remote -v`。
3. 向用户报告失败原因和下一步建议。
4. 不继续执行 merge。

### merge 冲突

表现：

- `git merge upstream/mine --no-edit` 返回冲突。
- `git status --short` 显示 `UU`、`AA`、`DU`、`UD` 等。

处理：

1. 立即运行：

```powershell
git status --short --branch
```

2. 逐个读取冲突文件，判断冲突来源。
3. 优先保留用户本地明确维护的内容，尤其是：
   - `AGENTS.md`
   - `.agents/`
   - `openspec/`
   - 用户明确说明过的本地补丁
4. 对上游任务代码、资源、配置模型冲突，优先理解双方改动后再合并，不要机械选择一边。
5. 解决冲突后执行：

```powershell
git add <resolved-files>
git commit
```

6. 如果冲突超出当前上下文或存在高风险，询问用户是否中止合并：

```powershell
git merge --abort
```

禁止：

- 不要对整个仓库批量执行 `git checkout --ours .` 或 `git checkout --theirs .`。
- 不要在未理解冲突时批量 `git add -A && git commit`。
- 不要手动修改自动生成的 `tasks/*/assets.py`，除非这是解决冲突所必需且已确认风险。

### untracked files would be overwritten

表现：

- merge 或 checkout 报错未跟踪文件会被覆盖。

处理：

1. 暂停操作。
2. 列出会被覆盖的文件。
3. 询问用户是否提交、移动、删除或暂存这些文件。
4. 不要自动删除未跟踪文件。

### 合并后发现重要本地文档缺失

表现：

- `Test-Path AGENTS.md` 或 `Test-Path .agents\INDEX.md` 返回 `False`。
- `git status` 或 `git diff` 显示协作文档被删除。

处理：

1. 立即检查最近合并提交的改动。
2. 如果仍在 merge 过程中，按冲突方式恢复本地版本。
3. 如果 merge 已完成但尚未推送，先不要推送。
4. 向用户说明风险，并根据需要用 Git 恢复文件。

### push 被拒绝

表现：

- `git push origin mine` 报 non-fast-forward。
- 远端有本地没有的新提交。

处理：

1. 不要强推。
2. 运行只读检查：

```powershell
git status --short --branch
git log --oneline --left-right --graph HEAD...origin/mine -30
```

3. 向用户说明远端存在新提交。
4. 只有用户明确要求后，才 fetch/merge `origin/mine` 或采取其他处理。

禁止：

- 不要执行 `git push --force` 或 `git push --force-with-lease`，除非用户明确要求并已确认风险。

### 已经是最新

表现：

- `git fetch upstream mine` 后，`git merge upstream/mine --no-edit` 显示 `Already up to date.`。

处理：

1. 仍执行合并后验证。
2. 向用户报告无需合并，当前本地 `mine` 已包含 `upstream/mine`。

## 完成检查清单

同步上游完成时，确认：

- `git status --short --branch` 无未提交内容。
- `upstream/mine` 是 `HEAD` 的祖先。
- 当前分支仍是 `mine`。
- 本地协作文档没有被误删。
- 已明确告知用户是否推送到 `origin/mine`。

推送完成时，额外确认：

- `mine` 与 `origin/mine` 已同步。
- `origin/mine` 指向预期最新提交。
