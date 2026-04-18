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

在 Claude Code 中运行：

```
/plugin marketplace add git@github.com:reoLantern/claude-codex-thread.git
/plugin install claude-codex-thread@claude-codex-thread
```

## Usage

**提具体问题**（Claude 直接问 Codex）：

```
/claude-codex-thread:codex-chat 这段代码的性能瓶颈在哪里？
```

**给宽泛话题**（Claude 自主规划提问、多轮推进，最后汇总）：

```
/claude-codex-thread:codex-chat 深入探讨一下 imm32 的编码方案
```

**续接已有会话**：

```
/claude-codex-thread:codex-chat --session <SESSION_ID> 追问
```

启动时 Claude 会读取 `~/.codex/config.toml` 并汇报当前模型配置。新 session 结束后会显示 SESSION_ID，供后续续接使用。
