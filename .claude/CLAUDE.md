# Codex 多轮对话

你可以在对话中随时使用 Codex CLI 进行多轮讨论。当用户提示"可以问问 Codex"、"和 Codex 讨论一下"或类似表达时，主动使用以下方式。

## 开启新 session

```bash
TMPSTDERR=$(mktemp)
RESPONSE=$(codex exec -s danger-full-access "$QUESTION" < /dev/null 2>"$TMPSTDERR")
SESSION_ID=$(grep "session id" "$TMPSTDERR" | awk '{print $NF}')
rm "$TMPSTDERR"
```

不能加 `--ephemeral`，session 必须保存到磁盘供后续 resume。

## 续接同一 session

```bash
TMPSTDERR=$(mktemp)
RESPONSE=$(codex exec resume --dangerously-bypass-approvals-and-sandbox \
  "$SESSION_ID" "$QUESTION" < /dev/null 2>"$TMPSTDERR")
rm "$TMPSTDERR"
```

`codex exec resume` 没有 `-s` 选项，用 `--dangerously-bypass-approvals-and-sandbox` 获取 full access。始终用精确的 SESSION_ID，不用 `--last`（多 session 并发时会接错）。

## Session 管理原则

- SESSION_ID 由你全程维护，用户不需要感知
- 同一对话中如需继续讨论，直接用已有的 SESSION_ID resume，无需重新开 session
- 自行规划向 Codex 问什么、问几轮，不需要每步请示用户
- 每次新对话前先运行 `grep -E "^model" ~/.codex/config.toml` 确认当前模型和思考深度
- 对话结束后向用户汇总 Codex 的核心观点
