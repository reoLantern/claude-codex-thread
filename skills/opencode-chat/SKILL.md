---
name: opencode-chat
description: |
  与 opencode 进行多轮对话，使用 opencode run --session 跨 Bash 调用保持上下文。
  TRIGGER when: 用户说"问问 opencode"、"让 opencode 看看"、"和 opencode 讨论"或类似表达；想要第二个 AI 视角且明确指定 opencode（而非 Codex）。
  SKIP: 普通对话、不涉及 opencode 的任务；用户明确要 Codex 的场景。
argument-hint: "[话题或问题]"
allowed-tools:
  - "Bash(opencode run*)"
  - "Bash(jq*)"
  - "Monitor"
---

# Opencode Chat

核心机制：每轮 Bash 调用是独立进程，opencode session 保存在磁盘，`--session $SID` 续接是跨调用保持对话的唯一方法。`--format json` 输出 NDJSON 流，每行事件都带 `sessionID`，文本回复在 `type=text` 事件的 `part.text`。

典型流程：
```
[用户请求] → opencode run → 得到 RESPONSE + SESSION_ID
           → 思考，决定追问
           → opencode run --session $SESSION_ID → 得到 RESPONSE
           → 继续 resume...
```

## 默认模型与权限

每次调用都显式传以下参数，不依赖 `~/.config/opencode/opencode.json` 的全局配置（这样 skill 行为可移植）：

- `-m deepseek/deepseek-v4-pro` —— 默认模型
- `--variant max` —— 思考深度
- `--dangerously-skip-permissions` —— 跳过权限确认（独立于 config 的 `permission` 字段）

如需改默认模型，编辑此 SKILL.md。首次调用前向用户告知：

> 当前 opencode 调用：model = deepseek/deepseek-v4-pro, variant = max

## 解析用户输入

- 若是具体问题，直接向 opencode 提问。
- 若是宽泛话题，由你规划提问角度和轮次，主动推进，最后汇总结论，不必每步请示用户。
- 若没有具体内容，先询问用户想讨论什么。

## 开启新 session

```bash
TMPOUT=$(mktemp)
opencode run --format json --dangerously-skip-permissions \
  -m deepseek/deepseek-v4-pro --variant max \
  "$QUESTION" < /dev/null > "$TMPOUT" 2>/dev/null
RESPONSE=$(jq -r 'select(.type=="text")|.part.text' "$TMPOUT")
SESSION_ID=$(head -1 "$TMPOUT" | jq -r '.sessionID')
rm "$TMPOUT"
```

## 续接已有 session

```bash
TMPOUT=$(mktemp)
opencode run --session "$SESSION_ID" --format json --dangerously-skip-permissions \
  -m deepseek/deepseek-v4-pro --variant max \
  "$QUESTION" < /dev/null > "$TMPOUT" 2>/dev/null
RESPONSE=$(jq -r 'select(.type=="text")|.part.text' "$TMPOUT")
rm "$TMPOUT"
```

注意：
- **始终用精确的 SESSION_ID，不用 `--continue`**。`--continue` 会接到 last session，多个并发时会错。
- `--format json` 必传 —— 否则 stdout 是给人看的格式化输出，难以可靠解析。
- `jq` 用 `select(.type=="text")` 拼接所有文本事件（一次回复可能分多段）。

## Session 管理

SESSION_ID 由你全程维护，用户不需要感知。同一对话中如需继续讨论，直接用已有的 SESSION_ID resume。

## 长任务模式

Claude Code 的 Bash tool 默认 2 分钟超时、最长 10 分钟。max variant + 大 prompt + 多文件输入时，单次 opencode 调用可能超过这些限制。按预估耗时分级：

**预计 2–10 分钟**：前台 Bash 加大 timeout，或用 `run_in_background: true` 拿完成通知。调用 Bash 时把 `timeout` 设到 `600000`（10 分钟），命令结束你会收到通知。

**预计可能超过 10 分钟**：用 `nohup` 让 opencode 脱离 Bash tool 进程组，再用 Monitor 轮询 exit 哨兵文件。

```bash
WORK=$(mktemp -d)
nohup bash -c "
  opencode run --format json --dangerously-skip-permissions \
    -m deepseek/deepseek-v4-pro --variant max \
    \"\$QUESTION\" < /dev/null > $WORK/stdout 2> $WORK/stderr
  echo \$? > $WORK/exit
" > /dev/null 2>&1 &
disown
echo "WORK=$WORK"
```

接着用 Monitor 等待（内部 `sleep` 不受 Bash tool 超时约束）：

```
until [ -f $WORK/exit ]; do sleep 5; done; echo "DONE"
```

Monitor 事件到来后，读文件取结果：

```bash
RESPONSE=$(jq -r 'select(.type=="text")|.part.text' $WORK/stdout)
SESSION_ID=$(head -1 $WORK/stdout | jq -r '.sessionID')
rm -rf $WORK
```

Resume 长任务同理：把 `opencode run --session $SESSION_ID ...` 放进同样的 nohup 壳即可。

## 每轮结束后

1. 向用户展示 opencode 的回复（或汇总结论）。
2. 询问是否继续追问。若是，用同一 SESSION_ID resume。
