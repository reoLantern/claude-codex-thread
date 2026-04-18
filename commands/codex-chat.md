---
description: "Start or continue a multi-turn Codex conversation"
argument-hint: "[question] [--session SESSION_ID]"
allowed-tools:
  - "Bash(codex exec*)"
---

# Codex Chat

Enables multi-turn conversations with Codex across multiple Bash calls. Each call is a new process; session files on disk carry the context forward.

## Parsing $ARGUMENTS

- If `--session SESSION_ID` is present, extract `SESSION_ID` and treat the remaining text as the question.
- Otherwise, treat all of `$ARGUMENTS` as the question for a new session.
- If `$ARGUMENTS` is empty, ask the user what they want to discuss with Codex before proceeding.

## Starting a new session

```bash
TMPSTDERR=$(mktemp)
RESPONSE=$(codex exec -s danger-full-access "$QUESTION" < /dev/null 2>"$TMPSTDERR")
SESSION_ID=$(grep "session id" "$TMPSTDERR" | awk '{print $NF}')
rm "$TMPSTDERR"
```

- Do NOT use `--ephemeral` — the session file must be saved to disk for resume.
- After getting the response, **tell the user the SESSION_ID** so they can resume later.

## Resuming an existing session

```bash
TMPSTDERR=$(mktemp)
RESPONSE=$(codex exec resume --dangerously-bypass-approvals-and-sandbox \
  "$SESSION_ID" "$QUESTION" < /dev/null 2>"$TMPSTDERR")
rm "$TMPSTDERR"
```

- `codex exec resume` has no `-s` flag; use `--dangerously-bypass-approvals-and-sandbox` for full access.
- Always use the exact `SESSION_ID`, never `--last` (unsafe when multiple sessions exist).

## After each response

1. Show the response to the user.
2. If this was a new session, show: `Session ID: <SESSION_ID>` — the user needs this to continue the conversation later.
3. Ask if the user wants to follow up. If yes, resume using the same `SESSION_ID`.
