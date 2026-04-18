# claude-codex-thread

A Claude Code plugin for multi-turn conversations with Codex. Uses `codex exec resume` to maintain session context across Bash calls.

## Prerequisites

- [Codex CLI](https://github.com/openai/codex) installed
- `~/.codex/config.toml` configured with your preferred model:

```toml
model = "gpt-5.4"
model_reasoning_effort = "xhigh"
```

## Installation

```
/plugin marketplace add reoLantern/claude-codex-thread
```

## Usage

```
/claude-codex-thread:codex-chat 你的问题
```

续接已有会话：

```
/claude-codex-thread:codex-chat --session <SESSION_ID> 追问
```

Claude 会在每次启动时读取你的 config.toml 并汇报当前模型配置。新 session 结束后会显示 SESSION_ID，供后续续接使用。
