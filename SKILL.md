---
name: trae-context-keeper
description: "Save full TRAE/TRAE Work conversation context locally as JSONL. Invoke when user says '保存上下文' or every 5 rounds. Includes compression detection and execution logging."
---

# TRAE Context Keeper

> TRAE/TRAE Work 的对话上下文本地保存 Skill。
>
> TRAE 将完整对话存储在服务器端，本地仅有蒸馏摘要。本 Skill 通过 AI 在对话过程中主动保存，将完整上下文（含工具调用参数和返回值）以 JSONL 格式写入本地磁盘。

## 触发条件

1. **用户主动触发**：用户说"保存上下文" -> 立即保存全部未保存轮次
2. **自动触发（尽力执行）**：当前轮次是 5 的倍数（第5、10、15...轮）时自动保存

> ⚠️ **自动触发是尽力而为（best-effort）**：Skill 是提示词不是后台程序，AI 专注回答复杂问题时可能遗漏检查轮次。建议用户偶尔看一眼轮次，到 5 的倍数时说"保存上下文"作为兜底。

## 目录结构

```
~/TRAE Work/
├── context.md                              # 全局索引
└── sessions/
    └── {YYYY-MM-DD}_{任务名}/
        ├── context.jsonl                   # 完整对话记录（追加写入）
        └── execution-log.md                # 执行日志
```

- 同一会话窗口 = 同一任务，不拆分
- 追加写入，永不覆盖

## 保存执行步骤

### Step 1: 读取上次保存状态

读取 `~/TRAE Work/sessions/{当前任务}/execution-log.md`，获取"上次保存轮次"。
- 文件不存在 -> 首次保存，从第 1 轮开始
- 文件存在 -> 从"上次保存轮次 + 1"开始追加

### Step 2: 检测上下文压缩

检查当前对话内容是否为逐字原文：
- **原文**：内容完整、有工具参数和返回值、无概括性表述
- **摘要**：内容被概括、缺少细节、出现"用户询问了..."等第三人称转述

压缩后仍需保存（摘要总比不存好），但在备注中标注"已压缩"。

### Step 3: 生成 JSONL 记录

每轮对话生成多条 JSONL 记录，每行一个 JSON 对象：

```jsonl
{"ts":"2026-07-12T13:30:00+08:00","round":1,"session_id":"xxx","model":"unknown","role":"user","content":"用户消息原文"}
{"ts":"2026-07-12T13:30:02+08:00","round":1,"session_id":"xxx","model":"unknown","role":"tool_call","tool":"LS","params":{"path":"/example"},"tool_call_id":"call_001"}
{"ts":"2026-07-12T13:30:02+08:00","round":1,"session_id":"xxx","model":"unknown","role":"tool_result","tool_call_id":"call_001","result":"工具返回结果"}
{"ts":"2026-07-12T13:30:05+08:00","round":1,"session_id":"xxx","model":"unknown","role":"assistant","content":"助手回复原文"}
```

#### 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `ts` | string | ISO 8601 时间戳，带时区 |
| `round` | number | 对话轮次（用户消息 + 助手回复 = 1 轮） |
| `session_id` | string | 会话 ID，同一窗口内不变 |
| `model` | string | 模型名，无法获取时填 "unknown" |
| `role` | string | `user` / `assistant` / `tool_call` / `tool_result` |
| `content` | string | 消息内容（仅 user / assistant） |
| `tool` | string | 工具名（仅 tool_call） |
| `params` | object | 工具参数（仅 tool_call） |
| `tool_call_id` | string | 工具调用 ID（tool_call 和 tool_result 关联用） |
| `result` | string | 工具返回结果（仅 tool_result） |

### Step 4: 写入文件

TRAE 的 Write 工具无法直接写入 `~/TRAE Work/`（PathScopeExceed），需绕路：

1. Write 写入临时目录：`{工作目录}/context_append.jsonl`
2. RunCommand 追加：`cat 临时文件 >> ~/TRAE Work/sessions/{任务}/context.jsonl`
3. 清理临时文件

### Step 5: 更新 execution-log.md

更新保存记录表、压缩检测表、上次保存轮次。

### Step 6: 更新 context.md（仅首次保存或状态变更时）

## 三个文件模板

### execution-log.md

```markdown
# 执行分析 - {日期}_{任务名}

## 保存记录
| 时间 | 操作 | 轮数范围 | 状态 | 备注 |

## 异常记录
| 时间 | 异常类型 | 详情 |

## 压缩检测
| 时间 | 检测结果 | 备注 |

## 模型信息
- 模型名：{unknown}

## 上次保存轮次
- 上次保存到：第{M}轮
- 当前轮次：第{N}轮
- 下次自动保存：第{N+5}轮

## 会话变更记录
- {时间}：{事件}
```

### context.md

```markdown
# 上下文索引
> ~/TRAE Work 目录的全局索引

## 任务列表
| 任务名 | 日期 | 状态 | session 目录 | 备注 |

## 异常记录
| 时间 | 任务名 | 异常类型 | 详情 |

## 规则说明
- JSONL 存完整上下文，execution-log.md 存执行分析
- 每 5 轮自动 + 用户主动要求
- 追加写入，不覆盖
```

### project_memory.md（TRAE 记忆目录）

```markdown
# 项目记忆 - {项目名}
> 规则摘要，完整对话在 ~/TRAE Work/ 的 JSONL 中

## 保存触发机制
- 方式A：每5轮自动保存
- 方式B：用户说"保存上下文"

## 保存格式
- JSONL，完整上下文（非蒸馏）
- 字段：ts/round/session_id/model/role/content/tool/params/tool_call_id/result

## 写入规则
- 同一会话 = 同一任务
- 追加写入，先读 execution-log.md 定位
- Write 路径限制 -> 临时目录 + cp/cat
```

## 注意事项

1. **model 字段**：无法自动获取，填 "unknown"
2. **工具返回值**：可能很长，完整保存不截断
3. **路径限制**：Write 无法直接写 `~/TRAE Work/`，必须临时目录 + cp/cat
4. **追加写入**：用 `>>` 不用 `>`
5. **轮次计数**：用户消息 + 助手完整回复（含所有工具调用）= 1 轮
6. **压缩不可控**：TRAE 平台系统级行为，AI 无法阻止，唯一对策是勤保存
7. **自动保存不保证 100%**：AI 可能因注意力分配遗漏，建议双保险
