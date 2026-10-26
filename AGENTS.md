# 多 Agent 协作总规则

## 规则优先级

1. 本文件中的全局规则
2. `roles/<role>.md` 中的角色规则
3. `tasks/<task-id>/contract.md` 中的任务要求

发生冲突时，优先级更高的规则生效。

## 强制要求

- 只执行任务契约明确授权的工作。
- 除了主Agent外的其余agent不得调用或委派给其他 Agent。
- 不得自行宣布整个项目完成。
- 不得伪造文件内容、测试结果或资料来源。
- 修改前必须读取目标文件。
- 发现信息不足时停止并报告。
- 完成后必须提交变更列表、验证结果和剩余风险。

## 角色加载

执行任务前，必须读取任务契约中的 `role` 字段，并加载对应的：

`roles/<role>.md`

 若角色文件不存在，禁止开始任务。

## 完成条件

子 Agent 只能将结果标记为“已提交”。

只有主 Agent 审查并通过验证后，任务才能标记为“已完成”。

## 外部调度（Codex CLI → DSH）

在 Codex CLI 或任何外部终端里，需要 DSH 参与写代码时，使用一次性模式——同一个模型、同一套工具与沙箱策略，在**当前工作目录**里干活，没有 GUI、不开端口：

```sh
dsh --profile headless "把 src/utils.js 里的日期格式化抽成函数并补单元测试"
```

- 账号已固化，直接就能用：`~/.dsh/profiles/headless/cordis.patch.yml` 里用 `disabled: false` 重新打开了 `credentials` / `authorization` / `deepseek-account`。`@deepseek-ai/dsh-headless` 的 bundle 层默认把这三个服务禁用（官方设计是"一次性任务只认启动时的 `DEEPSEEK_API_KEY`"），不禁用就会报 `ACCOUNT_SIGN_IN_REQUIRED`；而 profile 层在 bundle 层之后应用，所以能覆盖回来，复用桌面端已登录的账号。
- 若该 profile 被重置（重装、删掉 `~/.dsh/profiles/headless`），会退回报 `ACCOUNT_SIGN_IN_REQUIRED`：把仓库里 `tools/dsh/enable-account.yml` 的内容抄回 `cordis.patch.yml` 即可。
- 已实测（用 PATH 上的 `dsh`）：`dsh --profile headless "只回复：ok"` → `ok`，exit 0，并从 `api.deepseek.com` 正常拉到模型列表。
- 风险：headless 与桌面端共用同一份账号凭据，可能互相踢登录；长期高频用建议给 headless 单独配 `DEEPSEEK_API_KEY`。
- `dsh` 由 `%APPDATA%\npm\dsh.cmd` 提供，包装的是桌面版内置 CLI（`0.2.0-rc.2`）。
- 最终答案走 stdout，推理增量走 stderr；退出码 `0` 表示轮次完成，`1` 表示中止或出错。
- `--json` 输出行 JSON 事件流（`session` / `tool_call` / `tool_result` / `final`），适合被脚本逐行消费。
- `--session-id <id>` 续同一段对话，id 从 `--json` 的 `session` 事件里取；未知 id 会直接失败。
- 沙箱默认 `workspace-write` + 审批 `ask`：目录内读写免审批，越界操作会被拒绝（headless 模式下没有审批界面）。
- 同一仓库里多个 agent 并行改文件极易互相覆盖：主Agent只下发“改哪几个文件、验收标准是什么”，具体实现交给Coder，避免双方同时动同一文件。
