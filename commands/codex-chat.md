---
description: "与 Codex 进行多轮对话（支持跨 Bash 调用保持上下文）"
argument-hint: "[问题] [--session SESSION_ID]"
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

[用户下次提问] → 用同一个 SESSION_ID 继续 resume
              → Codex 记得之前所有对话
```

## 启动时：确认模型配置

在执行任何 Codex 调用前，先读取当前配置：

```bash
grep -E "^model" ~/.codex/config.toml
```

将结果展示给用户，例如：
> 当前 Codex 配置：model = gpt-5.4, model_reasoning_effort = xhigh

若文件不存在或无相关配置，提示用户 Codex 将使用默认模型。

## 解析 $ARGUMENTS

- 若含 `--session SESSION_ID`，提取 SESSION_ID，其余文字作为问题或话题。
- 否则全部 $ARGUMENTS 作为问题或话题，开启新 session。
- 若 $ARGUMENTS 为空，先询问用户想讨论什么。

**若 $ARGUMENTS 是一个宽泛的话题**（而非具体问题），由你自行规划需要向 Codex 问什么、问几轮，主动推进对话，最后向用户汇总结论。不需要每一步都请示用户。

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

## 每轮结束后

1. 向用户展示 Codex 的回复。
2. 若是新 session，显示：`Session ID: <SESSION_ID>`，用户下次可用 `--session` 续接。
3. 询问是否继续追问。若是，用同一 SESSION_ID resume。
