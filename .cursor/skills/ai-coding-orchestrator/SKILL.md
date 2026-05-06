---
description: 主 agent 编排 skill。规定四路并行流程的启动入口、各子代理调用顺序、prompt 必含要素、修订回路。识别到新开发任务需要启动本流程时读此 skill。触发词：ai-coding-orchestrator、orchestrator、主流程、四路并行、走流程。
---

# orchestrator 编排手册

> 文件存储路径与各子代理读写权限 → 读 **ai-coding-file-conventions skill**。
>
> 流程设计原则（为什么这样编排）→ 读 **ai-coding-flow.mdc**。
>
> 本手册只管"主 agent 怎么做"：调用哪个子代理、传什么、等什么、出了问题怎么转。

---

## 零、会话复用原则

**同一次主 agent 对话中，每类子代理只新建一次 Task 会话；修订阶段再次调用同一子代理时，使用 `resume: <task_id>` 续接已有会话，只传增量信息。**

- 首次调用 → 新建 Task，传入完整背景（skill/rule 路径、业务名等），保存返回的 task_id
- 再次调用（如修订阶段）→ `resume: <task_id>` + 只传增量（失败描述、业务名等）
- 例外（仅此两种情况才新建）：①业务名发生根本变更导致旧上下文混淆；②原会话已超时不可续接

**保存 task_id 映射**（主 agent 在内部记录）：
- `clarifier_task_id`、`spec_writer_task_id`、`implementor_task_id`、`test_writer_task_id`

**自检**：本轮是否对同一子代理新建了第二个 Task？是 → 检查是否满足例外条件，不满足则改用 resume。

---

## 流程总览

```
开始
  ↓
主 Agent 对话 → 命中协作场景？（§一）
  ├── 否 → 退出，走常规模式
  └── 是
       ↓
[阶段一 §二] clarifier：多轮澄清 ↔ 用户 → 归纳摘要 → 用户确认
  ├── 用户否认 → clarifier 继续澄清（clarifier 内部循环）
  └── 用户确认 → clarifier 归档（context.md）→ 返回业务名
       ↓
[阶段二 §三] 并行（同一 assistant 回合）
  ├── spec-writer：context.md → 需求文档.md
  └── implementor：context.md → 业务代码
  ├── 任一方有 ⚠️ 疑问 → 主 agent 汇报用户疑问点，等用户修订 → §5.2（不启动 test-writer）
  └── 两路均无疑问
       ↓
[阶段三 §四] test-writer：需求文档.md → 运行测试 → 汇报用户
       ↓
测试全通过？（§五）
  ├── 是 → 流程结束
  └── 否 → 用户仲裁
       ├── (a) 代码实现与规格不符 → §5.1：implementor 修改 → 重测
       ├── (b) 规格文档与需求不符 → §5.2：重澄清 → 重新并行 → 重测
       └── (c) 测试本身写错       → §5.3：test-writer 修正 → 重测
```

---

## 一、启动入口

**新开发**：直接用 `AskQuestion` 询问用户是否启动本流程（展示流程概览）。

**需求迭代（修改/追加已有功能）**：
1. 按 **ai-coding-file-conventions skill** 中"接口入口文档"的路径规则，遍历各业务目录下的 `接口入口.md`，根据用户描述的功能/接口关键词匹配业务名
2. 匹配到 → 读该 `接口入口.md`，用 `AskQuestion` 展示给用户确认（含业务名 + 接口列表）
3. 未匹配到 → 告知用户未找到对应记录，按新开发流程处理

- 用户确认 → 进入第二节（需求迭代时额外传入业务名）
- 用户拒绝 → 退出，走常规 Agent 模式

**自检**：是否通过 AskQuestion 展示操作计划并获得用户明确确认？否 → 先确认再继续。

---

## 二、阶段一：召唤 clarifier

**任务**：Task（ai-coding-clarifier skill），等待其完成后拿到业务名。

**传入**：
- ai-coding-clarifier skill 路径（必读）+ ai-coding-file-conventions skill 路径（必读）+ ai-coding-sub-agent rule 路径（必读）
- 用户原始需求原文
- 修订时额外传入：业务名

**等待结果**：clarifier 内部完成多轮澄清、用户确认、归档全过程，返回：
- 业务名

**注意**：主 agent 不干预 clarifier 的内部流程；不自行合并、转述或修改用户需求；3~5 条候选摘要在 clarifier 内部由用户确认，不再流转到主 agent。

→ **验证**：clarifier 返回了业务名；`context.md` 已存在于磁盘（路径查 **ai-coding-file-conventions skill**）。

**自检**：传给 clarifier 的 prompt 是否含 AI 归纳结论（而非用户原文）？有 → 清除，只传用户原文 + 业务名。

---

## 三、阶段二：并行启动 spec-writer + implementor

拿到业务名后，**同一批次并行**发起两个 Task（同一 assistant 回合内并发，不先后分两个回合）。

### ai-coding-spec-writer prompt 必含要素

| # | 要素 | 内容 |
|---|---|---|
| 1 | skill/rule 指向 | ai-coding-spec-writer skill 路径（必读）+ ai-coding-file-conventions skill 路径（必读）+ ai-coding-sub-agent rule 路径（必读） |
| 2 | 业务名 | `<业务名>` |

### ai-coding-implementor prompt 必含要素

| # | 要素 | 内容 |
|---|---|---|
| 1 | skill/rule 指向 | ai-coding-implementor skill 路径（必读）+ ai-coding-file-conventions skill 路径（必读）+ ai-coding-sub-agent rule 路径（必读） |
| 2 | 业务名 | `<业务名>` |
| 3 | 项目路径 | `src/` 根目录路径（已有代码可读） |

**等待两路都完成**。任一方回报 ⚠️ 疑问 → 主 agent 汇报用户疑问点，等待用户提供修订意见；**不启动 test-writer**，直接走 **§5.2 需求问题修订**，当前轮测试跳过。

→ **验证**：两路均完成（均未回报歧义）；两路 Task 在同一 assistant 回合内启动。

**自检**：传给 spec-writer / implementor 的 prompt 是否含 AI 归纳结论（非 context.md 路径本身）？有 → 删除再发。

---

## 四、阶段三：启动 test-writer

两路均无疑问时，启动 test-writer。

→ **验证**：`需求文档.md` 已存在于磁盘（路径查 **ai-coding-file-conventions skill**）。

### ai-coding-test-writer prompt 必含要素

| # | 要素 | 内容 |
|---|---|---|
| 1 | skill/rule 指向 | ai-coding-test-writer skill 路径（必读）+ ai-coding-file-conventions skill 路径（必读）+ ai-coding-sub-agent rule 路径（必读） |
| 2 | 业务名 | `<业务名>` |
| 3 | DTO/VO 路径 | 本次业务相关的 DTO/VO 文件路径（接口契约用） |
| 4 | 测试模块路径 | `<专用测试模块>/src/test/java/` 路径（产出落位） |

**自检**：传给 test-writer 的 prompt 是否夹带了 implementor 产出代码或 context.md 内容？有 → 删除。

---

## 五、测试结果汇报与修订回路

test-writer 完成后，**第一动作是汇报用户**（无论 PASS 还是 FAIL），禁止在用户看到前自行发起修订。

```
test-writer 完成
      ↓
主 agent 汇报用户（必需）
      ↓
用户判断
  ├── 全部通过 → 流程结束
  ├── 代码实现与规格不符（报告(a)）→ 见 5.1
  ├── 规格文档与需求不符（报告(b)）→ 见 5.2
  └── 测试本身写错（报告(c)）        → 见 5.3
```

### 5.1 代码问题修订

1. 主 agent 把测试失败**用自然语言总结**发给 implementor（不贴测试代码，不贴断言数值，只描述"用例名 + 预期行为引用需求章节 + 实际返回差异"）
2. implementor 修改后，主 agent 重新启动 test-writer
3. 回到本节开头（汇报用户）

**禁止反馈**（避免污染 implementor 独立性）：贴测试代码文件 / 具体断言代码 / 其他子代理产物原文。

### 5.2 需求问题修订

> 触发条件：①§三并行阶段任一方报 ⚠️ 疑问；②§五用户仲裁为需求问题（报告(b)）。
> ①触发时：主 agent 已在 §三 汇报疑问并拿到用户修订意见；②触发时：用户已在 §五 汇报后给出修订输入。

1. 召唤 clarifier（追加模式），传入：用户修订原文 + 业务名
2. clarifier 完成后返回业务名（context.md 已更新）
3. 主 agent **同一批次并行**发给 spec-writer + implementor（各自重新从新 context.md 提炼需求）
4. 两路完成 → 重新启动 test-writer → 汇报用户

### 5.3 测试问题修订

1. 主 agent 把 test-writer 的失败报告发给 test-writer（说明哪处测试自身写错），让其修正测试
2. test-writer 修改测试后重新运行
3. 回到本节开头（汇报用户）

**禁止反馈**（避免污染）：贴业务实现代码 / 需求文档原文。

循环直到用户判定完成或主动终止。

**自检**：测试结果是否第一时间汇报用户而非自行发起修订？否 → 先汇报再等用户指令。修订反馈中是否夹带了其他子代理产物原文？有 → 清除。

---

## 完成标准

满足以下任一条件，流程结束：
- 全部测试 PASS，用户确认完成
- 用户主动终止流程（接受当前状态或决定手工处理）

未满足 → 继续修订回路（§5.1 / §5.2 / §5.3），不得自行宣布完成。

**自检**：是否在用户确认或主动终止前自行宣布完成？是 → 恢复流程，等待用户确认。

---

## 自检清单

- [ ] spec-writer + implementor 是否在**同一批次**并行启动（同一 assistant 回合内两个 Task）？
- [ ] 发给 spec-writer / implementor / test-writer 的 prompt 是否只传业务名，没有硬编码路径或 AI 归纳结论？
- [ ] 发给 test-writer 的 prompt 是否未夹带 context.md 内容或 implementor 代码（防隔离污染）？
- [ ] 测试结果是否第一时间汇报用户，没有绕过用户自行修订？
- [ ] 反馈给 implementor 的失败信息是否是自然语言总结，没有贴任何子代理产物原文？
- [ ] 用户修订需求时是否先经 clarifier 更新 `context.md`，再并行扩散到 spec-writer + implementor？
- [ ] 修订阶段再次调用同一子代理时，是否使用了 `resume: <task_id>` 而非新建 Task（除非满足例外条件）？
