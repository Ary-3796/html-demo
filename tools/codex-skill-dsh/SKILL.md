---
name: "dsh"
description: "Use when the user asks to hand a coding task to DeepSeek Harness / DSH (\"让 dsh 去做\", \"用 dsh 写\", \"交给 dsh\", \"call dsh\"), or when a well-scoped change in this repository should be implemented by a second agent. Runs DSH one-shot headless mode through the `dsh` command."
---

# DSH Skill（把任务交给 DeepSeek Harness）

## 何时使用

- 用户明确说：`用 dsh 做…` / `让 dsh 去改…` / `交给 dsh` / `call dsh`。
- 某个改动范围清晰、与你自己在改的文件不重叠，适合并行交给另一个 agent（例如"补单元测试""把内联样式抽到 CSS"）。
- **不要**在你正在改同一批文件时调用——两个 agent 并行写同一文件会互相覆盖。

## 核心命令

`dsh` 已在 PATH 上（`%APPDATA%\npm\dsh.cmd`，包装桌面版内置 CLI `0.2.0-rc.2`）。

```bat
dsh --profile headless "<任务描述>"
```

账号已经配好，开箱即用。背景：`@deepseek-ai/dsh-headless` 的 bundle 层默认把 `credentials`、`authorization`、`deepseek-account` 三个服务 `disabled`（官方设计是"一次性任务只认启动时的 `DEEPSEEK_API_KEY`"），导致复用不了桌面端登录态、报 `ACCOUNT_SIGN_IN_REQUIRED`。已经在 `~/.dsh/profiles/headless/cordis.patch.yml` 里用 `disabled: false` 覆盖回来了（profile 层在 bundle 层之后应用）。若该 profile 被重置，把 `tools/dsh/enable-account.yml` 的内容抄回去即可。

- 在**当前工作目录**干活（沙箱 workspace-root = cwd）。
- 最终答案走 **stdout**，推理过程走 **stderr**。
- 退出码 `0` = 轮次完成；`1` = 中止或出错（stderr 里有 `dsh: <code>: <message>`）。
- 命令**阻塞**直到任务结束，不是后台进程，不要用 `--wait` 之类的东西等它。

> 注意：这里的 `--profile headless` 是 **DSH 的** profile，不是 Codex 的 `--profile`。
> `codex --profile headless` 是完全不同的东西（它会去找 `~/.codex/headless.config.toml`）。

## 常用变体

| 需求 | 写法 |
|---|---|
| 结构化进度（给脚本消费） | `dsh --profile headless --json "<任务>"` |
| 多轮 / 续同一段对话 | 先从 `--json` 的 `session` 事件取 id，再 `dsh --profile headless --session-id <id> "<后续>"` |
| 长任务描述（避免引号地狱） | `echo "<任务>" \| dsh --profile headless` |
| 全盘权限（仅用户明确要求时） | 先设 `DSH_PERMISSION_MODE=danger-full-access` 再调用 |

## 沙箱与审批（重要）

headless 默认 `workspace-write` + 审批 `ask`，但它**没有审批界面**：

- 工作目录内的读写 → 直接放行，无需审批。
- 越界操作（写 `C:\`、装全局依赖、读工作区外路径）→ 直接失败，不会弹窗等待。

所以任务里涉及工作区外的路径时，那一步由你自己（Codex）先做完，或明确告诉用户需要提权。

## 任务描述怎么写

把"改哪几个文件 + 验收标准"写清楚，而不是给一句模糊目标：

```bat
dsh --profile headless "修改 src/utils.js：把 formatDate 里的手工拼接换成 Intl.DateTimeFormat，保持对外导出和签名不变；改完在 test/utils.test.js 补 2 个用例覆盖跨时区。不要动其他文件。"
```

## 调用之后必须做的复核

1. 读 `dsh` 的 stdout 结论，但别只信它说"改完了"。
2. 自己 `git diff` 复核实际改动，并跑一遍测试。
3. 若 `dsh` 返回 exit 1，把 stderr 的错误码一并回报，不要静默重试。

## 侧记

- 项目根目录的 `AGENTS.md` 里也有一份说明，Codex 进入该项目时会自动读到。
- `~/.dsh/.credentials.yaml` 里的账号凭据由 DSH 桌面端和 headless 共用；若出现登录失效，重新登录 DSH 桌面端即可。
