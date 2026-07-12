# trae-context-keeper

> 🧠 让 TRAE / TRAE Work 的对话记忆不再"阅后即焚"。

TRAE 把完整对话存在服务器端，本地只留蒸馏摘要——关掉窗口，细节就没了。这个 Skill 让 AI 在对话过程中主动把完整上下文（含工具调用参数和返回值）写成 JSONL 存到本地磁盘，压缩了也能查。

## 它解决什么问题

| 没有 it | 有了 it |
|---------|---------|
| 关窗口 = 丢失完整对话 | 完整对话存在本地 JSONL |
| TRAE 压缩后原文不可恢复 | 压缩前已保存，随时可查 |
| 跨会话只能靠蒸馏摘要 | 跨会话可读完整历史 |
| 多任务对话混在一起 | 按任务分目录，有索引 |

## 安装

### 方式一：符号链接（推荐）

```bash
# 1. 克隆仓库到任意位置
git clone https://github.com/{your-username}/trae-context-keeper.git ~/Skill/trae-context-keeper

# 2. 在 TRAE 项目中创建符号链接
mkdir -p .trae/skills
ln -s ~/Skill/trae-context-keeper .trae/skills/trae-context-keeper
```

### 方式二：直接复制

```bash
# 把 SKILL.md 放到项目的 .trae/skills/trae-context-keeper/ 目录
mkdir -p .trae/skills/trae-context-keeper
cp SKILL.md .trae/skills/trae-context-keeper/SKILL.md
```

### 验证安装

在 TRAE 对话中说"保存上下文"，如果 AI 开始执行保存流程，说明 Skill 已生效。

## 使用

### 保存上下文

在对话中任何时候说：

```
保存上下文
```

AI 会立即把所有未保存的轮次写成 JSONL 追加到本地文件。

### 自动保存

每 5 轮对话 AI 会**尽量**自动保存。但因为 Skill 是提示词不是后台程序，AI 专注回答复杂问题时可能遗漏。建议偶尔看一眼轮次，到 5 的倍数时说"保存上下文"兜底。

### 查看保存的内容

```bash
# 看全局索引
cat ~/TRAE\ Work/context.md

# 看某个任务的执行日志
cat ~/TRAE\ Work/sessions/2026-07-12_我的项目/execution-log.md

# 看完整对话（JSONL，每行一条记录）
cat ~/TRAE\ Work/sessions/2026-07-12_我的项目/context.jsonl | jq .
```

## 文件结构

```
~/TRAE Work/                                    # 上下文存储根目录
├── context.md                                  # 全局索引（所有任务的清单）
└── sessions/
    └── 2026-07-12_我的项目/                    # 按日期+任务名分目录
        ├── context.jsonl                       # 完整对话记录（追加写入）
        └── execution-log.md                    # 执行日志（保存记录/异常/压缩检测）

~/Skill/trae-context-keeper/                    # Skill 源码（克隆位置）
├── SKILL.md                                    # Skill 定义文件
└── README.md                                   # 你正在看的这个

~/TRAE/我的项目/.trae/skills/trae-context-keeper # 符号链接 -> ~/Skill/trae-context-keeper
```

### 三个核心文件

| 文件 | 作用 | 谁写 |
|------|------|------|
| `context.jsonl` | 完整对话记录，每行一条 JSON | AI 在保存时追加 |
| `execution-log.md` | 保存记录、压缩检测、异常记录 | AI 在保存时更新 |
| `context.md` | 全局索引，列出所有任务 | AI 在首次保存时创建 |

### JSONL 字段

```jsonl
{"ts":"2026-07-12T13:30:00+08:00","round":1,"session_id":"abc","model":"unknown","role":"user","content":"用户消息"}
{"ts":"2026-07-12T13:30:02+08:00","round":1,"session_id":"abc","model":"unknown","role":"tool_call","tool":"LS","params":{"path":"/example"},"tool_call_id":"call_001"}
{"ts":"2026-07-12T13:30:02+08:00","round":1,"session_id":"abc","model":"unknown","role":"tool_result","tool_call_id":"call_001","result":"结果"}
{"ts":"2026-07-12T13:30:05+08:00","round":1,"session_id":"abc","model":"unknown","role":"assistant","content":"助手回复"}
```

| 字段 | 说明 |
|------|------|
| `ts` | ISO 8601 时间戳 |
| `round` | 对话轮次（用户消息 + 助手回复 = 1 轮） |
| `session_id` | 会话 ID |
| `model` | 模型名（无法获取时填 unknown） |
| `role` | user / assistant / tool_call / tool_result |
| `content` | 消息内容（user / assistant） |
| `tool` | 工具名（tool_call） |
| `params` | 工具参数（tool_call） |
| `tool_call_id` | 工具调用 ID |
| `result` | 工具返回结果（tool_result） |

## 已知限制

- **自动保存不保证 100%**：Skill 是提示词不是守护进程，AI 可能遗漏。双保险最靠谱。
- **压缩不可控**：TRAE 的上下文压缩是系统级行为，AI 无法阻止。对策就是勤保存。
- **model 字段填 unknown**：AI 无法自动获取当前模型名。
- **路径绕路**：TRAE 的 Write 工具无法直接写 `~/TRAE Work/`，需要先写临时文件再用 `cat >>` 追加。
- **仅适用 TRAE / TRAE Work**：TRAE 的完整对话只在服务器端，本地没有，所以必须靠 AI 主动保存。Claude Code / Codex CLI / OpenCode 等工具本地已有完整对话存储，不需要这个 Skill。

## 适用场景

- 用 TRAE / TRAE Work 做长期项目，需要回溯历史对话
- 对话很长，担心被 TRAE 压缩后丢失细节
- 跨会话恢复上下文时需要完整而非蒸馏的信息
- 想把 AI 编程过程中的决策、调试、技术讨论归档保存

## License

MIT
