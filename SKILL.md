---
name: trae-context-keeper
description: "将TRAE/TRAE Work对话完整上下文以JSONL格式保存到本地。当用户说'保存上下文'或每5轮时调用。Save full conversation context locally as JSONL. Invoke when user says '保存上下文' or every 5 rounds."
---

# TRAE Context Keeper

> TRAE/TRAE Work 的对话上下文本地保存 Skill。
>
> TRAE 将完整对话存储在服务器端，本地仅有蒸馏摘要。本 Skill 通过 AI 在对话过程中主动保存，将完整上下文（含工具调用参数和返回值）以 JSONL 格式写入本地磁盘。

## 触发条件

1. **用户主动触发**：用户说“保存上下文”时，立即保存全部未保存轮次。
2. **自动触发（尽力执行）**：当前轮次是 5 的倍数时自动保存。

> 自动触发是提示词能力，不是后台定时任务。用户主动说“保存上下文”是可靠兜底。

## 目录结构

```text
~/TRAE Work/
├── context.md                              # 全局索引
└── sessions/
    └── {YYYY-MM-DD}_{项目名}_{任务名}/
        ├── context.jsonl                   # 完整对话记录（追加写入）
        └── execution-log.md                # 保存质量、问题与结论
```

- **存储根目录固定为 `~/TRAE Work/`**，不可更改，不可创建其他变体目录（如 `~/TR Work/`）。
- `项目名`：工作目录名或用户指定名称。
- `任务名`：本次会话的具体任务简称。
- 同一项目的上下文按任务追加到对应 session 文件夹，不因 `session_id` 不同而新建目录。
- 追加写入，永不覆盖。

## 规则管理

> 往 `context.md` 的“规则说明”新增任何规则前，必须先读取本文件并逐项核对。规则不得与本文件冲突；如有冲突，以本文件为准，不写入 `context.md`。

## 保存执行步骤

### Step 1：读取保存状态

读取当前任务的 `execution-log.md` 中“当前保存状态”。

- 文件不存在：首次保存，从第1轮开始。
- 文件存在：从“上次保存”后的下一轮开始追加。

### Step 2：检测完整性

对每批待保存轮次判断完整性，只能使用以下枚举：

- **完整**：用户、助手、工具调用参数与工具返回值均可取得，无摘要替代。
- **有压缩**：任一轮被平台摘要化、部分工具过程不可取得，或只能保存结构化摘要。

备注必须说明具体范围，例如“第10-14轮有压缩；第15-20轮完整”。不得记录“追加N行”等实现细节。

### Step 3：生成 JSONL

每轮对话生成多条 JSONL 记录，每行一个 JSON 对象：

```jsonl
{"ts":"2026-07-12T13:30:00+08:00","round":1,"session_id":"xxx","model":"unknown","role":"user","content":"用户消息原文"}
{"ts":"2026-07-12T13:30:02+08:00","round":1,"session_id":"xxx","model":"unknown","role":"tool_call","tool":"LS","params":{"path":"/example"},"tool_call_id":"call_001"}
{"ts":"2026-07-12T13:30:02+08:00","round":1,"session_id":"xxx","model":"unknown","role":"tool_result","tool_call_id":"call_001","result":"工具返回结果"}
{"ts":"2026-07-12T13:30:05+08:00","round":1,"session_id":"xxx","model":"unknown","role":"assistant","content":"助手回复原文"}
```

| 字段 | 类型 | 说明 |
|---|---|---|
| `ts` | string | ISO 8601 时间戳，带时区 |
| `round` | number | 对话轮次：一条用户消息及其完整助手回复（含工具调用） |
| `session_id` | string | 会话 ID |
| `model` | string | 模型名；无法获取时为 `unknown` |
| `role` | string | `user` / `assistant` / `tool_call` / `tool_result` |
| `content` | string | 用户或助手消息 |
| `tool` | string | 工具名 |
| `params` | object | 工具调用参数 |
| `tool_call_id` | string | 工具调用关联 ID |
| `result` | string | 工具返回值 |

### Step 4：写入文件

若 Write 工具受路径范围限制：

1. 先写入临时目录的 `context_append.jsonl`。
2. 使用命令追加到目标 `context.jsonl`。
3. 验证 JSONL 可解析、轮次连续，再清理临时文件。

### Step 5：更新 execution-log.md

必须更新以下三个区块：

1. **上下文保存记录**：新增一行，字段为“时间、保存轮次、完整性、备注”。
   - 时间统一为 `YYYY-MM-DD HH:mm`。
   - 保存轮次写 `第M-N轮`。
   - 完整性只能写“完整”或“有压缩”。
2. **问题与处置**：仅记录影响数据可靠性、路径正确性、规则一致性或后续运行的真实问题；无新问题时不新增“无异常”记录。
3. **当前保存状态**：更新“上次保存”和“下次自动保存”。

如发生重要决策、技术结论、风险或待办，同时更新“关键事件与结论”。不得把每次工具操作写入此区块。

### Step 6：更新 context.md

仅在首次创建 session、任务状态变化、目录改名或全局规则变更时更新。更新前必须执行“规则管理”核对。

## execution-log.md 模板

```markdown
# 执行分析 - {日期}_{任务名}

## 上下文保存记录
| 时间 | 保存轮次 | 完整性 | 备注 |

## 问题与处置
| 时间 | 问题 | 处置与影响 |

## 关键事件与结论
- {日期}：{关键决策、技术结论、风险或待办}

## 当前保存状态
- 上次保存：第{M}轮
- 下次自动保存：第{N}轮
```

## context.md 模板

```markdown
# 上下文索引
> ~/TRAE Work 目录的全局索引

## 任务列表
| 项目 | 任务 | 日期 | 状态 | session 目录 | 备注 |

## 异常记录
| 时间 | 项目 | 异常类型 | 详情 |

## 规则说明
- 存储根目录固定为 ~/TRAE Work/，不可更改。
- 同一项目的上下文按任务追加到对应 session 文件夹，不因 session_id 不同而新建目录。
- 往规则说明区新增规则前必须先核对 SKILL.md，不可与 SKILL.md 冲突。
- JSONL 保存完整上下文；execution-log.md 保存质量、问题与结论。
- 每5轮尽力自动保存；用户主动要求时立即保存。
- 追加写入，不覆盖。
```

## project_memory.md 模板

```markdown
# 项目记忆 - {项目名}
> 规则摘要，完整对话在 ~/TRAE Work/ 的 JSONL 中

## 保存触发机制
- 每5轮尽力自动保存。
- 用户说“保存上下文”时立即保存。

## 写入规则
- 存储根目录固定为 ~/TRAE Work/，不可更改、缩写或创建变体目录。
- 同一项目的上下文按任务追加到对应 session 文件夹。
- 写入前先读 execution-log.md 定位上次保存轮次。
- Write 路径受限时使用临时目录加复制或追加命令。
```

## 注意事项

1. `model` 无法自动获取时写 `unknown`。
2. 工具返回值很长时仍应完整保存；若平台已压缩，明确标为“有压缩”。
3. 追加使用 `>>`，不可覆盖已有 JSONL。
4. 自动保存不保证100%执行；用户主动保存是兜底。
5. 存储路径只能是 `~/TRAE Work/`；写入任何记忆文件时不得缩写。
6. 任何新增规则先核对本文件，避免 `context.md` 与 Skill 冲突。
