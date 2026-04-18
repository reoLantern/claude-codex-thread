---
description: "与 Codex 进行多轮对话，使用 codex exec resume 跨 Bash 调用保持上下文。当用户提示"问问 Codex"、"和 Codex 讨论"或需要让 Codex 参与分析时，主动调用此技能。"
argument-hint: "[话题或问题]"
allowed-tools:
  - "Bash(codex exec*)"
  - "Bash(grep*)"
---

# Codex Chat

核心机制：每轮 Bash 调用是独立进程，通过磁盘上的 session 文件串联上下文。`codex exec resume` 是跨调用保持 Codex 对话的唯一方法。

典型流程：
```
[用户提问] → codex exec → 得到 RESPONSE + SESSION_ID
           → 思考，决定追问
           → codex exec resume $SESSION_ID → 得到 RESPONSE
           → 继续 resume...
```

## 启动时：确认模型配置

在执行任何 Codex 调用前，先读取当前配置：

```bash
grep -E "^model" ~/.codex/config.toml
```

将结果展示给用户，例如：
> 当前 Codex 配置：model = gpt-5.4, 思考深度 = xhigh

若文件不存在或无相关配置，提示用户 Codex 将使用默认模型。

## 解析 $ARGUMENTS

- $ARGUMENTS 为具体问题时，直接向 Codex 提问。
- $ARGUMENTS 为宽泛话题时，由你规划提问角度和轮次，主动推进，最后汇总结论。
- $ARGUMENTS 为空时，先询问用户想讨论什么。

## 开启新 session

```bash
TMPSTDERR=$(mktemp)
RESPONSE=$(codex exec -s danger-full-access "$QUESTION" < /dev/null 2>"$TMPSTDERR")
SESSION_ID=$(grep "session id" "$TMPSTDERR" | awk '{print $NF}')
rm "$TMPSTDERR"
```

**不能加 `--ephemeral`** — session 必须保存到磁盘，resume 才能读取。

## 续接已有 session

```bash
TMPSTDERR=$(mktemp)
RESPONSE=$(codex exec resume --dangerously-bypass-approvals-and-sandbox \
  "$SESSION_ID" "$QUESTION" < /dev/null 2>"$TMPSTDERR")
rm "$TMPSTDERR"
```

注意：
- `codex exec resume` 没有 `-s` 选项，用 `--dangerously-bypass-approvals-and-sandbox` 获取 full access。
- TMPSTDERR 此处只是抑制 stderr 输出，不需要提取任何信息。
- **始终用精确的 SESSION_ID，不用 `--last`**。`--last` 在多个 session 并发时会接到错误的 session。

## Session 管理

SESSION_ID 由你全程维护，用户不需要感知。同一对话中如需继续讨论，直接用已有的 SESSION_ID resume。

## 每轮结束后

1. 向用户展示 Codex 的回复。
2. 询问是否继续追问。若是，用同一 SESSION_ID resume。
