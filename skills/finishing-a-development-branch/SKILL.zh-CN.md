---
name: finishing-a-development-branch
description: 当实现完成、测试通过、需要决定如何整合工作时使用 — 通过呈现合并、PR或清理的结构化选项来指导完成开发工作
---

# 完成开发分支

## 概述

通过呈现清晰的选项和处理选定的工作流程来指导完成开发工作。

**核心原则：** 验证测试 → 检测环境 → 呈现选项 → 执行选择 → 清理。

**开头声明：** "我正在使用完成开发分支技能来结束这项工作。"

## 流程

### 步骤 1：验证测试

**在呈现选项之前，验证测试是否通过：**

```bash
# 运行项目的测试套件
npm test / cargo test / pytest / go test ./...
```

**如果测试失败：**
```
测试失败（<N> 个失败）。必须先修复才能完成：

[显示失败内容]

在测试通过之前无法继续合并/PR。
```

停止。不要继续步骤 2。

**如果测试通过：** 继续步骤 2。

### 步骤 2：检测环境

**在呈现选项之前，确定工作区状态：**

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
```

这决定了显示哪个菜单以及如何进行清理：

| 状态 | 菜单 | 清理 |
|-------|------|---------|
| `GIT_DIR == GIT_COMMON`（普通仓库） | 标准 4 选项 | 无工作树需要清理 |
| `GIT_DIR != GIT_COMMON`，命名分支 | 标准 4 选项 | 基于来源（见步骤 6） |
| `GIT_DIR != GIT_COMMON`，分离 HEAD | 精简 3 选项（无合并） | 无清理（外部管理） |

### 步骤 3：确定基础分支

```bash
# 尝试常见的基础分支
git merge-base HEAD main 2>/dev/null || git merge-base HEAD master 2>/dev/null
```

或询问："此分支从 main 分出——这是正确的吗？"

### 步骤 4：呈现选项

**普通仓库和命名分支工作树 — 只呈现这 4 个选项：**

```
实现完成。您想怎么做？

1. 在本地合并回 <基础分支>
2. 推送并创建 Pull Request
3. 保持现状（我稍后处理）
4. 丢弃此工作

选择哪个选项？
```

**分离 HEAD — 只呈现这 3 个选项：**

```
实现完成。您处于分离 HEAD 状态（外部管理的工作区）。

1. 推送为新分支并创建 Pull Request
2. 保持现状（我稍后处理）
3. 丢弃此工作

选择哪个选项？
```

**不要添加解释** — 保持选项简洁。

### 步骤 5：执行选择

#### 选项 1：在本地合并

```bash
# 获取主仓库根目录以确保 CWD 安全
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"

# 先合并 — 确认成功后再删除任何内容
git checkout <基础分支>
git pull
git merge <功能分支>

# 在合并结果上验证测试
<测试命令>

# 仅在合并成功后：清理工作树（步骤 6），然后删除分支
```

然后：清理工作树（步骤 6），然后删除分支：

```bash
git branch -d <功能分支>
```

#### 选项 2：推送并创建 PR

```bash
# 推送分支
git push -u origin <功能分支>

# 创建 PR
gh pr create --title "<标题>" --body "$(cat <<'EOF'
## 摘要
<2-3 条变更说明>

## 测试计划
- [ ] <验证步骤>
EOF
)"
```

**不要清理工作树** — 用户需要保持工作树以便在 PR 反馈上继续迭代。

#### 选项 3：保持现状

报告："保持分支 <名称>。工作树保留在 <路径>。"

**不要清理工作树。**

#### 选项 4：丢弃

**先确认：**
```
这将永久删除：
- 分支 <名称>
- 所有提交：<提交列表>
- 工作树在 <路径>

输入 'discard' 确认。
```

等待精确确认。

如果确认：
```bash
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"
```

然后：清理工作树（步骤 6），然后强制删除分支：
```bash
git branch -D <功能分支>
```

### 步骤 6：清理工作区

**仅在为选项 1 和 4 时运行。** 选项 2 和 3 始终保留工作树。

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
WORKTREE_PATH=$(git rev-parse --show-toplevel)
```

**如果 `GIT_DIR == GIT_COMMON`：** 普通仓库，无工作树需要清理。完成。

**如果工作树路径在 `.worktrees/`、`worktrees/` 或 `~/.config/superpowers/worktrees/` 下：** 由 Superpowers 创建的工作树 — 我们负责清理。

```bash
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"
git worktree remove "$WORKTREE_PATH"
git worktree prune  # 自愈：清理任何过期的注册
```

**否则：** 主机环境（harness）拥有此工作区。不要删除。如果您的平台提供工作区退出工具，请使用它。否则，将工作区留在原处。

## 快速参考

| 选项 | 合并 | 推送 | 保留工作树 | 清理分支 |
|--------|-------|------|---------------|----------------|
| 1. 本地合并 | 是 | - | - | 是 |
| 2. 创建 PR | - | 是 | 是 | - |
| 3. 保持现状 | - | - | 是 | - |
| 4. 丢弃 | - | - | - | 是（强制） |

## 常见错误

**跳过测试验证**
- **问题：** 合并损坏的代码，创建失败的 PR
- **修复：** 始终在呈现选项前验证测试

**开放式问题**
- **问题：** "接下来我应该做什么？" 模糊不清
- **修复：** 呈现恰好 4 个结构化选项（或分离 HEAD 的 3 个）

**为选项 2 清理工作树**
- **问题：** 删除用户需要用于 PR 迭代的工作树
- **修复：** 仅对选项 1 和 4 进行清理

**在移除工作树之前删除分支**
- **问题：** `git branch -d` 失败，因为工作树仍引用该分支
- **修复：** 先合并，移除工作树，然后删除分支

**从工作树内部运行 git worktree remove**
- **问题：** 当 CWD 在被移除的工作树内部时，命令静默失败
- **修复：** 在 `git worktree remove` 之前始终 `cd` 到主仓库根目录

**清理 harness 拥有的工作树**
- **问题：** 删除 harness 创建的工作树会导致幽灵状态
- **修复：** 仅清理 `.worktrees/`、`worktrees/` 或 `~/.config/superpowers/worktrees/` 下的工作树

**丢弃时无确认**
- **问题：** 意外删除工作
- **修复：** 要求输入 "discard" 确认

## 危险信号

**绝对不能：**
- 在测试失败的情况下继续
- 在结果上不验证测试就合并
- 不确认就删除工作
- 未经明确请求就强制推送
- 在确认合并成功前移除工作树
- 清理非你创建的工作树（来源检查）
- 从工作树内部运行 `git worktree remove`

**始终：**
- 在呈现选项前验证测试
- 在呈现菜单前检测环境
- 呈现恰好 4 个选项（或分离 HEAD 的 3 个）
- 为选项 4 获取输入确认
- 仅对选项 1 和 4 清理工作树
- 在工作树移除前 `cd` 到主仓库根目录
- 在移除后运行 `git worktree prune`