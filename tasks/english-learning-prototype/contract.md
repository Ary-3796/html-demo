# 任务契约：英语学习网页 · 接入真实 AI（DeepSeek）+ 后端代理

## 1. 任务元数据

- **task-id**：`english-learning-prototype`
- **主 Agent role**：`main agent`
- **Coder role**：`coder`
- **Checker role**：`checker`
- **状态**：`running`（用户已确认本阶段目标并授权启动，详见第 8 节）

## 2. 用户目标（本阶段）

在已完成并验收的网页可演示原型基础上，接入真实 AI 能力：

- 保留当前首页与各功能页布局（用户已认可排版，保持不变）。
- 新增本地后端/代理服务，DeepSeek API Key 只放服务端（环境变量），前端只调用代理，Key 不得写入前端或提交到仓库。
- 真实功能：
  1. 图片上传后真实读取并识别（OCR，走 DeepSeek 视觉模型）。
  2. 单词复习：识别出的单词真实生成释义/音标/提示；答案检查（拼写/造句/翻译）走真实 AI。
  3. 作业批改：真实批改（单词/句子/阅读题），返回批改说明与建议改写。
  4. 生成 Word 复习单：真实生成可下载的 `.docx` 文件。
- 变更：单词复习与作业批改的「图片上传」不再区分「手写/印刷」（移除来源切换及相关逻辑）。

## 3. 技术方案（主 Agent 决定，Coder 按此实施）

- 后端运行时：Node.js（当前环境 v24.12.0）+ Express + `docx`（生成 .docx）。前端仍为静态 HTML/JS，不引入前端框架。
- 新增目录 `server/`，不动 `src/` 以外的既有规则文件。
- 后端同时静态托管 `src/`：用户访问 `http://localhost:8787/` 即可；API 前缀 `/api`。
- 配置与环境变量（均不写死 Key、不提交真实 Key）：
  - `DEEPSEEK_API_KEY`：必填，未配置时相关接口返回 `503` 并给出明确提示。
  - `DEEPSEEK_BASE_URL`：默认 `https://api.deepseek.com`。
  - `PORT`：默认 `8787`。
  - `DEEPSEEK_CHAT_MODEL`：默认 `deepseek-chat`。
  - `DEEPSEEK_VISION_MODEL`：默认 `deepseek-flash`（视觉模型，做 OCR）。
- API 契约（JSON；上传图片用 base64 data URL 放在 JSON 里，避免 multipart）：
  - `GET /api/health` → `{ ok, configured }`
  - `POST /api/ocr` → 入参 `{ image, purpose: "review" | "grade" }` → `{ text, words: string[] }`
  - `POST /api/words` → 入参 `{ words: string[] }` → `[{ word, phonetic, meaning, hint }]`
  - `POST /api/grade` → 入参 `{ text, kind: "words" | "sentences" | "reading", context? }` → `{ summary, correction, suggestion }`
  - `POST /api/word-doc` → 入参 `{ words: [{ word, meaning }], title? }` → 返回 `.docx` 附件（`Content-Disposition: attachment`）
- 错误与回退：API 未配置或调用失败时，前端展示清晰错误态，并可回退到预置演示数据并明确标注「演示」。

## 4. Coder 任务

### 4.1 允许读取

- `D:\Project\HTML\AGENTS.md`
- `D:\Project\HTML\roles\coder.md`
- 本契约
- `D:\Project\HTML\src\` 下与页面实现有关的文件
- `D:\Project\HTML\server\`（自建文件）

### 4.2 允许修改

- `D:\Project\HTML\src\index.html`（前端接入与去手写/印刷区分）
- 新增 `D:\Project\HTML\server\` 下实现所需文件（含 `package.json`、`.env.example` 等，不含真实 Key）

### 4.3 禁止修改

- `AGENTS.md`
- `roles\` 下角色规则
- 本契约
- 真实 API 密钥、系统配置和与本任务无关的文件
- 不引入真实 Key、不上传真实 Key 到仓库

### 4.4 后端任务（`server/`）

- 初始化 `package.json`（依赖 `express`、`docx`；如需 CORS 可自加 `cors`，但后端已静态托管 `src/`，通常无需 CORS）。
- 实现 5 个 API 端点，按第 3 节契约。
- `.docx` 生成用 `docx` 库：输出复习单（含单词、释义、默写空行、造句/翻译练习区、标题/日期/姓名/得分占位）。
- 读取环境变量；提供 `.env.example` 说明配置项，不包含真实 Key。
- 统一错误处理：未配置 Key 返回 503；DeepSeek 调用失败返回 502 和可读信息；入参非法返回 400；所有错误不泄露 Key。
- 静态托管 `src/`，并在启动日志打印访问地址与 Key 是否已配置。

### 4.5 前端任务（`src/index.html`）

- 移除上传弹窗的「手写图片/印刷图片」来源切换（`.source-switch`）及相关 JS：`currentSource`、`sourceLabel`、`updateSourceButtons`、`handleSourceChange`、历史/记录中的来源字段等；上传文案同步去掉手写/印刷字样。
- 图片上传：真实读取为 base64 并预览，调用 `POST /api/ocr`。
- 单词复习：识别词 → `POST /api/words` 生成词卡；拼写/造句/翻译答案检查改走 `POST /api/grade`。
- 作业批改：图片识别 → `POST /api/grade`；手动输入也改走 `POST /api/grade`。
- 生成 Word：调用 `POST /api/word-doc` 触发 `.docx` 下载。
- 保留现有布局；补齐 loading / 空态 / 错误态；API 不可用时回退到预置演示数据并明确标注「演示」。

### 4.6 Coder 验收与提交

- 修改前必须读取实际目标文件。
- 后端至少执行语法检查（`node --check`）并实际启动服务验证 `GET /api/health`；无 Key 时验证返回 503。前端至少做 HTML/JS 静态结构检查。
- 提交内容必须包括：实际修改文件、每个文件修改内容、运行命令及真实结果、未解决问题、潜在风险、建议 Checker 重点检查项。
- Coder 只能将状态标记为 `submitted`，不得宣布最终完成。

## 5. Checker 任务

### 5.1 允许读取

- 本契约及项目级 `AGENTS.md`
- `roles\checker.md`
- Coder 实际修改的文件和版本差异
- Coder 报告中列出的验证结果

### 5.2 禁止修改

- 默认不得修改 Coder 的代码、契约或项目规则；发现问题只提交返工要求，不直接修复。

### 5.3 独立检查要求

- 查看真实文件内容和差异，不能只相信 Coder 的文字总结。
- 独立运行可复现的命令（启动服务、`GET /api/health`、未配置 Key 时各接口返回 503 等）。
- 检查是否越过允许文件范围、是否引入不必要依赖、是否泄露密钥、是否遗漏关键交互。
- 检查：布局仍保留左侧功能栏与两个主面板；上传不再区分手写/印刷；图片读取→OCR、单词生成、批改、Word 下载链路是否真实调用后端；错误/回退态是否清晰。

### 5.4 Checker 输出

结论只能是：`ACCEPTED`、`REJECTED` 或 `NEEDS_USER`。

报告必须包括：原始目标、实际检查文件、实际差异、运行命令及结果、发现的问题、范围检查、最终结论；若拒绝，还要列出明确返工要求。

## 6. 主 Agent 职责

- 审查 Coder 提交与 Checker 报告，不把 Coder 的「已完成」直接视为最终完成。
- Checker 为 `REJECTED` 时退回 Coder；为 `NEEDS_USER` 时向用户提问；证据充分且检查通过后才向用户汇报。
- 最终汇报必须包含用户目标、完成任务、实际修改文件、验证结果、Checker 结论、剩余风险和下一步建议。

## 7. 明确不做

- 不实现登录、用户系统、云端同步或其他左侧功能的具体业务。
- 不把 DeepSeek Key 写入前端或提交到仓库。
- 不伪造 API 调用、识别结果、批改结果、下载文件或测试结果。
- 生产级部署、鉴权、限流、鉴黄/安全审查等不在本阶段范围。

## 8. 历史与用户已确认事项

1. 阶段一（可点击/可展示的演示原型）已由 Checker 独立验收为 `ACCEPTED`，布局与演示交互保留。
2. 用户已确认本阶段目标：接入真实 AI（DeepSeek），Key 放服务端（新增后端/代理），前端只调代理。
3. 用户已确认排序：保留当前排版；图片上传去掉「手写/印刷」区分。

原始版本与用户补充设计后的更新版均已由 Checker 独立验收为 `ACCEPTED`；本阶段为新增阶段，状态 `running`。