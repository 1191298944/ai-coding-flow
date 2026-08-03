---
name: git-workflow
description: Git 操作工作流。当用户要求提交、推送、push、合并、merge、迁出、checkout -b、覆盖分支、更新代码、暂存、save，或执行任何 git 写操作时使用。
---

# Git 操作工作流

> 与 `human-as-copilot` 分工：本 Skill 管 Git **具体步骤**；协作审批边界仍由 human-as-copilot 约束。  
> 所有 git 命令通过 Shell 执行；禁止用 GitLens MCP（如 `gitlens_commit_composer`）代替 shell。

## 0. 核心约束

- 所有 commit 必须带 `--trailer "Made-with: Cursor"`
- 多行 commit message 必须通过 Git Bash heredoc 或临时文件执行
- 禁止 `git config` 变更；禁止未确认的 force push / hard reset
- 禁止 `--no-verify` 等跳过 hook（用户明确要求除外）

## 1. 执行前置（每次 git 操作逐条过一遍）

| 步骤 | 动作 | 工具 / 兜底 |
|---|---|---|
| ①项目识别 | 见下方项目映射 | 多项目未明确时询问，禁止猜测 |
| ②操作计划确认 | 展示「项目路径 / 当前分支 / 涉及分支 / 操作步骤」 | 询问：「确认执行」/「取消」 |
| ③文件范围确认 | 涉及 `git add` 时，先 `git status` 列出全部变更 | 询问：「全部提交」/「排除部分」 |
| ④高危二次确认 | `reset --hard` / `--force` push / 整分支覆盖 | 二次确认：「我已知晓风险，确认」/「取消」 |
| ⑤归位 | 操作结束切回操作前分支 | 例外：3d 迁出新分支后保持在新分支 |

**项目映射**：
- 若依：`ipd-studio ↔ ipd-web`、`scm-studio ↔ scm-web`、`voc-studio ↔ voc-web`
- AIWA：`micro-app-platform ↔ micro-app-platform-web`
- Airflow：`airflow-job`（单仓）

**自检**：是否已展示操作计划并得到确认？未确认 → 不执行。

## 2. 操作触发词速查

| 代号 | 触发词 | 操作序列 |
|---|---|---|
| 3a | 提交 / 推送 / push | `add` → `commit` → `push` |
| 3b | A 合并到 B / merge A to B | pull A → pull B → `merge --no-ff A` → 切回 |
| 3c | 合并并推送 | 3b → `push` |
| 3d | 从 A 迁出 / checkout -b | 询问分支名 → pull A → `checkout -b` → `push -u origin` |
| 3e | 暂存 / 保存 / save | `add` → `commit`（**不 push**） |
| 3f | 用 A 覆盖 B（高危） | pull A → checkout B → `reset --hard A` → `push --force-with-lease` |
| 3g | 用 A 的文件覆盖 B | 询问路径 → `checkout A -- <paths>` → commit → push |
| 3h | 更新代码 / 同步源分支 | 识别源分支 → `git pull origin <源分支>` |

- **3d 分支命名**：默认 `feature-YYYYMMDD-cary`，YYYYMMDD 用当天日期；或询问用户输入
- **3h 源分支识别**：先 `git log` / `git show-branch` 推断；无法确定时询问

### 通用外壳（除 3h 外都适用）

```bash
CURRENT=$(git -C "<项目路径>" rev-parse --abbrev-ref HEAD)
# ... 按代号执行具体步骤 ...
git -C "<项目路径>" checkout $CURRENT   # 结束归位（3d 例外）
```

## 2.1 高危样板

### 3f. 用 A 完整覆盖 B（必须二次确认）

```bash
CURRENT=$(git -C "<路径>" rev-parse --abbrev-ref HEAD)
git -C "<路径>" checkout A && git -C "<路径>" pull
git -C "<路径>" checkout B && git -C "<路径>" pull
# 警告：B 独有提交将丢失
git -C "<路径>" reset --hard A
git -C "<路径>" push --force-with-lease origin B
git -C "<路径>" checkout $CURRENT
```

### 3g. 用 A 的指定路径覆盖 B

```bash
CURRENT=$(git -C "<路径>" rev-parse --abbrev-ref HEAD)
git -C "<路径>" checkout A && git -C "<路径>" pull
git -C "<路径>" checkout B && git -C "<路径>" pull
git -C "<路径>" checkout A -- <路径1> <路径2>
# 走文件范围确认后再 commit / push
git -C "<路径>" checkout $CURRENT
```

## 3. Commit Message 格式

```
类型(范围): 简短描述（≤ 30 字，标题行）

- 变更点 1
- 变更点 2
```

- 标题与描述之间必须空一行；禁止把标题和描述合并成一行写进 `-m`
- 单处变更可省描述，仅保留标题行

| 性质 | type |
|---|---|
| 新增功能 / 接口 / 字段 | feat |
| 修复 bug | fix |
| 重构 / 性能（不改行为） | refactor |
| 格式 / 样式 | style |
| 文档 / 注释 | docs |
| 分支合并 | merge |
| 依赖 / 配置 / 构建 | chore |
| 单元测试 | test |

模板：

```bash
git commit --trailer "Made-with: Cursor" -m "$(cat <<'EOF'
feat(任务调度): 新增复杂任务调度V2实现

- 新增 ComplexTaskSchedulerServiceImplV2
- 重构工期计算逻辑，支持节假日跳过
EOF
)"
```

## 4. PowerShell 环境兼容

PowerShell 不支持 bash heredoc。首次 commit 前按顺序探测 bash 并记住路径：

```powershell
bash --version
& "C:/Program Files/Git/bin/bash.exe" --version
& "C:/Program Files (x86)/Git/bin/bash.exe" --version
& (git --exec-path).Replace('mingw64/libexec/git-core','bin/bash.exe').Replace('mingw32/libexec/git-core','bin/bash.exe') --version
```

优先用 bash 包装（单引号内写 bash 单引号用 `'\''`）：

```powershell
& "<探测到的bash路径>" -c 'git -C "<项目路径>" commit --trailer "Made-with: Cursor" -m "$(cat <<'\''EOF'\''
feat(范围): 简短描述

- 变更点 1
EOF
)"'
```

无 bash 时用临时文件：

```powershell
git -C "<项目路径>" commit --trailer "Made-with: Cursor" -F "<项目路径>/COMMIT_MSG.tmp"
Remove-Item "<项目路径>/COMMIT_MSG.tmp"
```

## 5. 生效信号

- [ ] git 命令执行前有操作计划确认
- [ ] `git add` 前有 `git status` 与文件范围确认
- [ ] commit 标题 ≤ 30 字，标题与描述空一行
- [ ] 所有 commit 带 `Made-with: Cursor`
- [ ] 高危操作经二次确认
- [ ] 结束后分支归位（3d 除外）
