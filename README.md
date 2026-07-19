# trae-context-keeper

> 让 TRAE / TRAE Work 的对话记忆不再“阅后即焚”。

TRAE 把完整对话存于服务器端，本地通常只保留蒸馏摘要。此 Skill 让 AI 在对话过程中主动把完整上下文（包括工具调用参数与返回值）写为本地 JSONL；即使后续发生压缩，也能回查已保存内容。

## 它解决的问题

| 没有 Skill | 使用 Skill |
|---|---|
| 关闭窗口后难以回溯完整对话 | 完整对话保存在本地 JSONL |
| 压缩后原文不可恢复 | 压缩前已保存的内容可查 |
| 跨会话只能依赖摘要 | 可读取完整历史记录 |
| 多任务混在一起 | 按项目与任务分目录，并由全局索引定位 |

## 安装

### 符号链接（推荐）

```bash
git clone https://github.com/{your-username}/trae-context-keeper.git ~/Skill/trae-context-keeper
mkdir -p .trae/skills
ln -s ~/Skill/trae-context-keeper .trae/skills/trae-context-keeper
```

### 直接复制

```bash
mkdir -p .trae/skills/trae-context-keeper
cp SKILL.md .trae/skills/trae-context-keeper/SKILL.md
```

在 TRAE 对话中说“保存上下文”。若 AI 开始执行保存流程，说明 Skill 已生效。

## 使用

任何时候输入：

```text
保存上下文
```

AI 会将未保存轮次追加到 JSONL。每5轮也会尽力自动保存，但这不是系统级定时任务，主动触发是可靠兜底。

## 文件结构

```text
~/TRAE Work/
├── context.md                                  # 全局索引
└── sessions/
    └── 2026-07-12_我的项目_具体任务/
        ├── context.jsonl                       # 完整对话记录（追加写入）
        └── execution-log.md                    # 保存质量、问题与结论

~/Skill/trae-context-keeper/
├── SKILL.md                                    # Skill 定义
└── README.md                                   # 使用说明
```

## 三个核心文件

| 文件 | 作用 | 更新方式 |
|---|---|---|
| `context.jsonl` | 完整对话记录，每行一个 JSON 对象 | 每次保存时追加 |
| `execution-log.md` | 保存质量、问题与处置、关键结论、当前保存状态 | 每次保存时更新 |
| `context.md` | 所有任务的全局索引与规则 | 首次保存或状态变化时更新 |

### execution-log.md 结构

```markdown
## 上下文保存记录
| 时间 | 保存轮次 | 完整性 | 备注 |

## 问题与处置
| 时间 | 问题 | 处置与影响 |

## 关键事件与结论
- 关键决策、技术结论、风险与待办

## 当前保存状态
- 上次保存：第M轮
- 下次自动保存：第N轮
```

- 时间统一使用 `YYYY-MM-DD HH:mm`。
- “完整性”仅使用“完整”或“有压缩”。
- 备注只描述哪些轮次完整或被压缩，不记录“追加N行”等实现细节。

## JSONL 字段

```jsonl
{"ts":"2026-07-12T13:30:00+08:00","round":1,"session_id":"abc","model":"unknown","role":"user","content":"用户消息"}
{"ts":"2026-07-12T13:30:02+08:00","round":1,"session_id":"abc","model":"unknown","role":"tool_call","tool":"LS","params":{"path":"/example"},"tool_call_id":"call_001"}
{"ts":"2026-07-12T13:30:02+08:00","round":1,"session_id":"abc","model":"unknown","role":"tool_result","tool_call_id":"call_001","result":"结果"}
{"ts":"2026-07-12T13:30:05+08:00","round":1,"session_id":"abc","model":"unknown","role":"assistant","content":"助手回复"}
```

| 字段 | 说明 |
|---|---|
| `ts` | ISO 8601 时间戳 |
| `round` | 一条用户消息及其完整助手回复为一轮 |
| `session_id` | 会话 ID |
| `model` | 模型名；无法获取时为 `unknown` |
| `role` | `user` / `assistant` / `tool_call` / `tool_result` |
| `content` | 用户或助手消息 |
| `tool` | 工具名 |
| `params` | 工具调用参数 |
| `tool_call_id` | 工具调用关联 ID |
| `result` | 工具返回值 |

## 关键规则

- 存储根目录只能是 `~/TRAE Work/`，不可改为或缩写为其他路径。
- 同一项目的上下文按任务追加到对应 session 文件夹，不因 `session_id` 不同而新建目录。
- 新增 `context.md` 规则前必须核对 `SKILL.md`，冲突时以 `SKILL.md` 为准。
- 平台压缩不可控；越早主动保存，越能保留原文。

## 已知限制

- 自动保存不保证100%执行：Skill 是提示词，不是守护进程。
- 平台压缩后无法恢复未保存的原文。
- 无法自动获取模型名时会写为 `unknown`。
- Write 工具可能无法直接写入 `~/TRAE Work/`，需先写临时文件再复制或追加。
- 此 Skill 仅面向 TRAE / TRAE Work；Claude Code、Codex CLI、OpenCode 等通常已有本地完整会话存储。

## License

MIT
