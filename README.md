# claude-codex-thread

A Claude Code plugin for multi-turn conversations with external AI CLIs. Uses `codex exec resume` / `opencode run --session` to maintain session context across Bash calls. Currently supports **Codex** and **opencode**.

## Prerequisites

### Codex

- [Codex CLI](https://github.com/openai/codex) installed
- `~/.codex/config.toml` configured with your preferred model:

```toml
model = "gpt-5.5"
model_reasoning_effort = "xhigh"
```

### opencode

- [opencode](https://opencode.ai) installed (`opencode --version` works)
- A provider configured in `~/.config/opencode/opencode.json` for the model the skill uses (default: `deepseek/deepseek-v4-pro`). Run `opencode auth` to set up credentials.
- `jq` available on `PATH` (used to parse opencode's JSON event stream).
- The skill always passes `--dangerously-skip-permissions` and `-m deepseek/deepseek-v4-pro --variant max`, so it does not depend on your global `permission` / default-model settings. To change the default model, edit `skills/opencode-chat/SKILL.md`.

## Installation

在 Claude Code 中运行：

```
/plugin marketplace add git@github.com:reoLantern/claude-codex-thread.git
/plugin install claude-codex-thread@claude-codex-thread
```

## Usage

**自然语言触发**（主要方式）：安装后无需任何命令，在与 Claude 的对话中直接说：

> "你也可以去问问 Codex 关于 xxx 的看法"
>
> "让 opencode 看看这段代码"

如果同时安装了其他相关插件，可以明确指定方式：

> "按 claude-codex-thread 的方式，去问问 Codex 关于 xxx 的看法"

Claude 会自主规划提问、管理 session、多轮推进，最后向你汇总结论。Session ID 由 Claude 全程维护，用户无需感知。

**显式触发**：

```
/claude-codex-thread:codex-chat 话题或问题
/claude-codex-thread:opencode-chat 话题或问题
```

启动时 Claude 会读取/汇报当前模型配置（Codex 读 `~/.codex/config.toml`；opencode 用 skill 内置的默认模型与 variant）。
