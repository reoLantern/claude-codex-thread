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

**自然语言触发**（主要方式）：安装后无需任何命令，在与 Claude 的对话中直接说：

> "你也可以去问问 Codex 关于 xxx 的看法"

如果同时安装了其他 Codex 相关插件，可以明确指定方式：

> "按 claude-codex-thread 的方式，去问问 Codex 关于 xxx 的看法"

Claude 会自主规划提问、管理 session、多轮推进，最后向你汇总结论。Session ID 由 Claude 全程维护，用户无需感知。

**显式触发**：

```
/claude-codex-thread:codex-chat 话题或问题
```

启动时 Claude 会读取 `~/.codex/config.toml` 并汇报当前模型配置。
