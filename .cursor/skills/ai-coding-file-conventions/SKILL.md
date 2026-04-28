---
description: 四路并行流程的文件存储路径与各子代理读写权限约定。ai-coding-orchestrator skill 和所有子代理 skill 均从此处引用，是唯一真相源，方便用户集中查看和修改。触发词：ai-coding-file-conventions、file-conventions、文件约定。
---

# 文件约定

> 被 ai-coding-orchestrator skill 和所有子代理 skill 引用。修改路径或权限时只改这里。

## 存储路径

| 产物 | 路径 |
|---|---|
| 各轮澄清增量原文 | `~/.cursor/.llm/<业务名>/chatHistory/V<n>.md` |
| 当前有效需求陈述 | `~/.cursor/.llm/<业务名>/chatHistory/Final.md` |
| 接口入口文档 | `~/.cursor/.llm/<业务名>/接口入口.md` |
| 需求文档 | `~/.cursor/.llm/<业务名>/需求文档.md` |
| 业务代码 | 按项目既有模块规范落位 |
| 集成测试 | `<专用测试模块>/src/test/java/<包路径>/<Controller名>Test.java` |

**命名与路径规则**：
- `<业务名>` 为英文短词或拼音，全小写 + 连字符（如 `order-export`），无中文 / 空格 / 特殊符号
- Windows 下 `~` 解析为 `%USERPROFILE%`（如 `C:\Users\yuwen\.cursor\.llm\<业务名>\`）
- `Final.md` 记录当前有效需求（贴近用户原文），每次归档覆盖写入，内容与 `V<n>.md` **不同**
- `V<n>.md` 只记录本轮对话原文，各轮独立存档，不含前版内容，不累积；编号从 1 起，每次归档递增；全部拼接可还原完整澄清过程
- `接口入口.md` 覆盖式更新；内容严格限于接口契约（URL / Method / 核心入参字段 / 核心出参字段）；禁止写入业务规则、SQL、边界场景；用于需求迭代时反向定位业务名
- `需求文档.md` 覆盖式更新，不分版本
- 同业务多接口：需求文档按子章节区分，测试类按被测 Controller 各建一个

---

## 各子代理读写权限

### clarifier

| 操作 | 范围 |
|---|---|
| 允许读 | 用户需求原文；`Final.md`（修订时必读，作为本轮起点）；`V<n>.md`（AI 理解存疑时可追溯，非必读）；数据库 schema |
| 禁止读 | `src/` 下任何业务代码；spec-writer / implementor / test-writer 产物 |
| 允许写 | `chatHistory/V<n>.md`；`chatHistory/Final.md` |
| 约束 | 归档前（用户确认前）禁止写任何文件 |

### spec-writer

| 操作 | 范围 |
|---|---|
| 允许读 | `Final.md`（路径见上方存储路径表）；数据库 schema；业务样本数据；其他需求文档（仅作结构参考） |
| 禁止读 | `src/main/java/` 下任何业务代码；`V<n>.md` 历史快照；implementor / test-writer 产物 |
| 允许写 | `~/.cursor/.llm/<业务名>/需求文档.md` |

### implementor

| 操作 | 范围 |
|---|---|
| 允许读 | `Final.md`（路径见上方存储路径表）；`src/` 下已有业务代码；数据库 schema；项目规则 |
| 禁止读 | `需求文档.md`（spec-writer 产物）；`V<n>.md` 历史；测试模块下任何文件 |
| 允许写 | 业务代码（Controller / Service / Mapper / XML / VO / DTO）按项目模块规范 |
| 禁止写 | 测试模块下任何文件；`~/.cursor/.llm/` 下任何文件 |

### test-writer

| 操作 | 范围 |
|---|---|
| 允许读 | `需求文档.md`（路径见上方存储路径表）；本次业务 DTO/VO；数据库 schema；无关模块既有代码（测试数据构造参考）；本次业务 Controller HTTP 映射注解（@RequestMapping / @PostMapping 等注解 + 参数声明，**不读方法体**，用于生成 `接口入口.md`） |
| 禁止读 | `Final.md`；`V<n>.md` 历史；implementor 产出的 Controller 方法体 / Service / Mapper / XML 内部逻辑 |
| 允许写 | `<专用测试模块>/src/test/java/...` 下测试类；`~/.cursor/.llm/<业务名>/接口入口.md` |

---

## 违规处理

不在"允许"列表内的读/写操作 → **立即停止，回报主 agent**，不得继续执行。具体处理见 **ai-coding-sub-agent rule**。

## 生效信号

- 每个子代理的产物只落在上表"允许写"的路径下
- 没有子代理读取过"禁止读"范围内的文件
- 路径中的 `<业务名>` 符合全小写 + 连字符格式
