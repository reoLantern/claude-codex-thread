---
name: codex-chat
description: |
  与 Codex 进行多轮对话，使用 codex exec resume 跨 Bash 调用保持上下文。
  TRIGGER when: 用户说"问问 Codex"、"让 Codex 看看"、"和 Codex 讨论"、"按 claude-codex-thread 的方式"或类似表达；需要第二个 AI 视角参与分析时。
  SKIP: 普通对话、不涉及 Codex 的任务。
argument-hint: "[话题或问题]"
allowed-tools:
  - "Bash(codex exec*)"
  - "Bash(grep*)"
---

# Codex Chat

核心机制：每轮 Bash 调用是独立进程，通过磁盘上的 session 文件串联上下文。`codex exec resume` 是跨调用保持 Codex 对话的唯一方法。

典型流程：
```
[用户请求] → codex exec → 得到 RESPONSE + SESSION_ID
           → 思考，决定追问
           → codex exec resume $SESSION_ID → 得到 RESPONSE
           → 继续 resume...
```

## 启动时：确认模型配置

首次调用 Codex 前，读取当前配置并告知用户：

```bash
grep -E "^model" ~/.codex/config.toml
```

例如：
> 当前 Codex 配置：model = gpt-5.4, 思考深度 = xhigh

若文件不存在或无相关配置，提示用户 Codex 将使用默认模型。

## 解析用户输入

- 若是具体问题，直接向 Codex 提问。
- 若是宽泛话题，由你规划提问角度和轮次，主动推进，最后汇总结论，不必每步请示用户。
- 若没有具体内容，先询问用户想讨论什么。

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

1. 向用户展示 Codex 的回复（或汇总结论）。
2. 询问是否继续追问。若是，用同一 SESSION_ID resume。
