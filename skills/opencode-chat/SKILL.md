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
  - "Bash(setsid*)"
  - "Write"
  - "Monitor"
---

# Opencode Chat

核心机制：每轮 Bash 调用是独立进程，opencode session 保存在磁盘，`--session $SID` 续接是跨调用保持对话的唯一方法。`--format json` 输出 NDJSON 流，每行事件都带 `sessionID`，文本回复在 `type=text` 事件的 `part.text`。

典型流程：
```
[用户请求] → opencode run < prompt.txt → 得到 RESPONSE + SESSION_ID
           → 思考，决定追问
           → opencode run --session $SESSION_ID < prompt.txt → 得到 RESPONSE
           → 继续 resume...
```

## 默认模型与权限

每次调用都显式传以下参数，不依赖 `~/.config/opencode/opencode.json` 的全局配置（这样 skill 行为可移植、可调试）：

- `-m deepseek/deepseek-v4-pro` —— 默认模型
- `--variant max` —— 思考深度
- `--dangerously-skip-permissions` —— 跳过权限确认（独立于 config 的 `permission` 字段）

如需改默认模型，编辑此 SKILL.md。首次调用前向用户告知（措辞要让用户清楚这是 skill 内置默认、不读 opencode.json）：

> 本次 opencode 调用将使用 model = deepseek/deepseek-v4-pro, variant = max（skill 内置默认，独立于你的 opencode 全局配置）

## 解析用户输入

- 若是具体问题，直接向 opencode 提问。
- 若是宽泛话题，由你规划提问角度和轮次，主动推进，最后汇总结论，不必每步请示用户。
- 若没有具体内容，先询问用户想讨论什么。

## Prompt 处理（关键）

**绝不要把 prompt 作为 shell 命令行参数传**。这条是硬规则——实践中 `"$QUESTION"` 和 `$(cat ...)` 在 nohup/setsid 子 shell 里反复因变量未继承、嵌套引号、特殊字符（`$`、反引号、表格、emoji、Unicode）失败。

**正确做法**：每次调用前
1. 用 **Write 工具**（不是 echo / heredoc）把 prompt 写到 `$WORK/prompt.txt`
2. 用 **stdin 重定向** 喂给 opencode：`opencode run ... < "$WORK/prompt.txt"`

实测 opencode 在没有 message 位置参数时会从 stdin 读整个内容作为消息，多行 markdown / 中文 / Unicode 全部安全。

## 工作目录约定

每个任务用一个**确定性命名**的 WORK 目录（避免撞历史 / 通配符误匹配）：

```bash
WORK=/tmp/oc-task-$(date +%s)-$$
mkdir -p "$WORK"
```

后续所有路径都用 `$WORK/prompt.txt`、`$WORK/stdout` 等绝对路径。Monitor 等待时也写绝对路径，**绝不用 `/tmp/oc-task-*/exit` 这种通配符**。

## 开启新 session（短任务，预计 < 2 分钟）

第一步：用 Write 工具写 `/tmp/oc-task-XXX/prompt.txt`（XXX 自选）。

第二步：

```bash
WORK=/tmp/oc-task-XXX  # 与 Write 写入路径保持一致
opencode run --format json --dangerously-skip-permissions \
  -m deepseek/deepseek-v4-pro --variant max \
  < "$WORK/prompt.txt" > "$WORK/stdout" 2> "$WORK/stderr"
EXIT=$?
RESPONSE=$(jq -r 'select(.type=="text")|.part.text' "$WORK/stdout")
SESSION_ID=$(head -1 "$WORK/stdout" | jq -r '.sessionID // empty')
echo "EXIT=$EXIT  SID=$SESSION_ID"
```

**Sanity check**：拿到 SESSION_ID 后立刻验证：

```bash
[[ "$SESSION_ID" =~ ^ses_ ]] || echo "WARN: bad session id, opencode 可能没正常启动"
```

如果 EXIT≠0 或 SESSION_ID 不以 `ses_` 开头，去看 `$WORK/stderr` 找原因（常见：模型未配 provider、变体不存在、网络），再决定重试或报错给用户。

## 续接已有 session

```bash
opencode run --session "$SESSION_ID" --format json --dangerously-skip-permissions \
  -m deepseek/deepseek-v4-pro --variant max \
  < "$WORK/prompt.txt" > "$WORK/stdout" 2> "$WORK/stderr"
EXIT=$?
RESPONSE=$(jq -r 'select(.type=="text")|.part.text' "$WORK/stdout")
```

**Resume fallback**：如果 stderr 中出现 `session not found`、`invalid session` 之类，或 EXIT≠0 且没有任何文本输出，**自动开新 session 而不是卡住**：

```bash
if [[ $EXIT -ne 0 ]] && grep -qiE "session not found|invalid session|unknown session" "$WORK/stderr"; then
  echo "session $SESSION_ID 失效，开新 session" >&2
  # 回退到"开启新 session"流程，并告知用户上下文已断
fi
```

注意：
- **始终用精确的 SESSION_ID，不用 `--continue`**。`--continue` 接 last session，多个并发会错。
- `--format json` 必传，否则 stdout 是格式化输出难以解析。
- `jq 'select(.type=="text")|.part.text'` 拼接所有文本事件（一次回复可能分多段）。

## Session 管理

SESSION_ID 由你全程维护，用户不需要感知。同一对话中如需继续讨论，直接用已有的 SESSION_ID resume。Resume 失败时按上面 fallback 流程开新 session 并告知用户。

## 长任务模式（预计 > 2 分钟）

Claude Code 的 Bash tool 默认 2 分钟超时、最长 10 分钟。max variant + 大 prompt 可能超出。

**预计 2–10 分钟**：前台 Bash 加大 timeout（设到 `600000`），或 `run_in_background: true` 等通知，命令仍按上面"开启新 session"的形态写。

**预计可能超过 10 分钟**：用 setsid 把 opencode 脱离 Bash tool 进程组，再用 Monitor 等 exit 哨兵文件。

第一步：Write 写 `$WORK/prompt.txt`。

第二步：launch（用 setsid 替代 `nohup + disown` —— 后者在非交互式 shell 里偶发 `disown: current: no such job` 噪音）：

```bash
WORK=/tmp/oc-task-XXX
setsid bash -c "
  opencode run --format json --dangerously-skip-permissions \
    -m deepseek/deepseek-v4-pro --variant max \
    < $WORK/prompt.txt > $WORK/stdout 2> $WORK/stderr
  echo \$? > $WORK/exit
" > /dev/null 2>&1 < /dev/null &
```

要点：
- `bash -c "..."` 用**双引号**而不是单引号 —— 让外层 shell 把 `$WORK` 展开成字面路径再交给子 shell（这样路径不依赖子 shell 继承环境变量，绕开"`$QUESTION` 在子 shell 里是空"的坑）。
- `\$?` 留给子 shell（转义 `$`），这是子 shell 退出码。
- prompt 走 stdin，子 shell 命令行**不出现 prompt 内容**，无任何引号问题。

第三步：Monitor 等 exit 哨兵（绝对路径，不用通配符）：

```
until [ -f /tmp/oc-task-XXX/exit ]; do sleep 5; done; echo "DONE exit=$(cat /tmp/oc-task-XXX/exit)"
```

第四步：Monitor 事件后读结果 + sanity check：

```bash
WORK=/tmp/oc-task-XXX
EXIT=$(cat "$WORK/exit")
RESPONSE=$(jq -r 'select(.type=="text")|.part.text' "$WORK/stdout")
SESSION_ID=$(head -1 "$WORK/stdout" | jq -r '.sessionID // empty')
[[ "$SESSION_ID" =~ ^ses_ ]] || echo "WARN: bad sid，看 $WORK/stderr"
rm -rf "$WORK"
```

Resume 长任务同形：launch 命令里把第一步换成 `opencode run --session "$SESSION_ID" ...`。

## 每轮结束后

1. 向用户展示 opencode 的回复（或汇总结论）。
2. 询问是否继续追问。若是，用同一 SESSION_ID resume。

## 失败排查清单

收到坏结果（空 RESPONSE、bad SESSION_ID、EXIT≠0）时按顺序排查：

1. `cat $WORK/stderr` 看 opencode 报错原文。
2. `ls $WORK` 看是否所有文件齐全（prompt.txt / stdout / stderr / exit）。缺 stdout/stderr 通常是 setsid'd 子 shell 启动失败 —— 可能是路径含特殊字符或 setsid 未安装；改回前台 Bash + 加大 timeout 试试。
3. `head -3 $WORK/stdout` 看 NDJSON 是否真的产出过 step_start —— 没 step_start 通常是模型/provider 配置错。
4. SESSION_ID 不以 `ses_` 开头 → opencode 没正常完成首步，看 stderr。
