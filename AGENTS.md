# AGENTS.md

## 任务完成要求

- 所有与用户的对话、PLAN 文档均使用中文。
- 除非用户在当前对话中明确要求，否则不要运行 `bun fmt`、`bun lint` 或 `bun typecheck`。
- 【目前不需要执行】在认为任务完成之前，`bun fmt`、`bun lint` 和 `bun typecheck` 必须全部通过。
- 将 `bun fmt`、`bun lint` 和 `bun typecheck` 视为重量级的工作区检查：每个任务尽可能合并为一次最终验证，避免在迭代过程中反复运行整套检查。
- 如果用户在一次完整的验证通过后不久提出小的跟进需求，除非用户明确要求再次完整验证，否则优先不进行重跑或只做最小的合理重检。
- 【目前都用这个】如果用户要求只关注代码，不要自动运行 `bun fmt`、`bun lint` 或 `bun typecheck`。在该模式下，先完成代码修改，只有用户明确要求时才运行验证。
- 永远不要运行 `bun test`。请始终使用 `bun run test`（运行 Vitest）。

## 项目简介

Synara 是一个用于使用 Codex 和 Claude 等编码代理的极简 Web GUI。

该仓库尚处于非常早期的开发阶段。我们鼓励提出能够提升长期可维护性的大规模改动。

## 核心优先事项

1. 性能优先。
2. 可靠性优先。
3. 在高负载以及故障期间（会话重启、重新连接、部分流）保持行为可预测。

如果需要权衡，请选择正确性和稳健性，而非短期便利。

## Model Selection

Rankings, higher = better. Cost reflects what I actually pay (OpenAI is near-free for me due to a deal), not list price. Intelligence is how hard a problem you can hand the model unsupervised. Taste covers UI/UX, code quality, API design, and copy.

| model       | cost | intelligence | taste |
| ----------- | ---- | ------------ | ----- |
| gpt-5.6-sol | 9    | 8            | 5     |
| sonnet-5    | 5    | 5            | 7     |
| opus-4.8    | 4    | 7            | 8     |
| fable-5     | 2    | 9            | 9     |

How to apply:

- These are defaults, not limits. You have standing permission to override them: if a cheaper model's output doesn't meet the bar, rerun or redo the work with a smarter model without asking. Judge the output, not the price tag. Escalating costs less than shipping mediocre work.
- Cost is a tie-breaker only; when axes conflict for anything that ships, intelligence > taste > cost.
- Don't let cost prevent you from using the right model for the job. Instead, take advantage of cheaper options to get more information and try things before moving the work to a more expensive option.
- Bulk/mechanical work (clear-spec implementation, data analysis, migrations): gpt-5.6-sol — it's effectively free.
- Anything user-facing (UI, copy, API design) needs taste ≥ 7.
- Reviews of plans/implementations: fable-5 or opus-4.8, optionally gpt-5.6-sol as an extra independent perspective.
- Never use Haiku.
- Mechanics: gpt-5.6-sol is only reachable through the Codex CLI — `codex exec` / `codex review` (my `~/.codex/config.toml` defaults to gpt-5.6-sol). Use the codex-implementation, codex-review, and codex-computer-use skills; for work they don't cover (investigation, data analysis), run `codex exec -s read-only` directly with a self-contained prompt.
- Claude models (sonnet-5, opus-4.8, fable-5) run via the Agent/Workflow model parameter.

Using gpt-5.5 inside workflows and subagents (the model parameter only takes Claude models, so use a wrapper):

- Spawn a thin Claude wrapper agent with `model: 'sonnet', effort: 'low'` whose prompt instructs it to write a self-contained codex prompt, run `codex exec` via Bash, and return the report (use `schema` on the wrapper to get structured output back).
- Always label these agents with a `gpt-5.6-sol:` prefix, e.g. `{label: 'gpt-5.6-sol:review-auth'}` — the workflow UI shows the wrapper's Claude model, so the label is the only indication the real worker is gpt-5.6-sol.
- Codex runs can exceed Bash's 10-minute timeout: pass an explicit timeout, or run in the background and poll for the report file.
- Parallel gpt-5.6-sol implementation agents must use `isolation: 'worktree'` so codex edits don't collide in the shared checkout.
- Workflow token budgets only count Claude tokens; codex work is free and invisible to `budget.spent()`.

## Long-running Codex Work

gpt-5.6-sol is exceptionally capable on long-running tasks. Give it substantial, multi-step work when it is the right model for the job; do not split work up merely because it is large.

- The quality of the result depends on the prompt. Provide a detailed, self-contained brief: goal, relevant context, constraints, files or systems in scope, expected deliverables, and how to verify completion.
- State important decisions and non-negotiable requirements explicitly. Do not assume the model will infer project-specific conventions or the desired tradeoffs from a short prompt.
- For long tasks, ask it to inspect the current state first, execute the work end to end, and report the changes, verification, and any remaining risks.
- If the work can safely run in parallel, keep each task's ownership and worktree boundaries explicit so agents do not overlap.

## 对话记录性能护栏

- 将对话记录自动滚动视为实时输出功能，而非通用的“正在工作”功能。缓冲、重新连接、待审批和仅工具活动，都不能像助手文本正在主动流式输出那样连接。
- 在连接滚动跟随逻辑时，只计算真实的对话记录消息。工具/工作行不得触发相同的“新内容到达”自动吸底路径。
- 对于常见情况，优先使用更简单的 fork 风格对话记录路径。中小型对话记录应避免虚拟化开销，除非有明确的实测需求。
- 如果使用虚拟化，切勿将 `rowVirtualizer.measure()` 直接与另一个底部吸附或高度跟随循环耦合。实时输出的高度跟随应保持单向，以避免 measure/scroll 反馈循环。
- 在更改聊天滚动、时间线测量或侧边栏驱动的对话记录更新时，通过针对性的对话记录测试来保留这些行为。

## 可维护性

长期可维护性是核心优先事项。如果你要添加新功能，首先检查是否可以将共享逻辑提取到单独的模块中。多个文件之间的重复逻辑是一种代码异味，应尽量避免。不要害怕修改现有代码。不要为了解决某个问题而只添加局部逻辑走捷径。

## UI Conventions

### Open/close (toggle) animations — single source

Any UI element with an open/close toggle (expand/collapse, show/hide, disclosure) MUST reuse the shared disclosure motion in `apps/web/src/lib/disclosureMotion.ts`. Never write bespoke height/opacity transitions or one-off `@keyframes` for a toggle — use the same logic and the same functions everywhere so every toggle feels identical (220ms `ease-out`, with `motion-reduce` fallbacks).

- Shell + content (used by open/close project, sidebar sections, composer suggestions): `disclosureShellClassName(open)` on the grid shell, `DISCLOSURE_INNER_CLASS` on the inner wrapper, `disclosureContentClassName(open)` on the content — or the ready-made `DisclosureRegion` component (`apps/web/src/components/ui/DisclosureRegion.tsx`).
- Base UI `<Collapsible>` panels: wrap with `CollapsiblePanel` (`apps/web/src/components/ui/collapsible.tsx`), which applies `DISCLOSURE_COLLAPSIBLE_PANEL_CLASS`.
- Rotating chevron affordance: `DisclosureChevron` / `disclosureChevronClassName(open)`.

Reference usage: opening/closing a project and the sidebar sections in `apps/web/src/components/Sidebar.tsx`. If you find a toggle that animates differently, migrate it to this module rather than duplicating logic.

## 包角色

- `apps/server`：Node.js WebSocket 服务器。包装 Codex app-server（通过 stdio 的 JSON-RPC），为 React Web 应用提供服务，并管理提供程序会话。
- `apps/web`：React/Vite UI。负责会话用户体验、对话/事件渲染以及客户端状态。通过 WebSocket 连接到服务器。
- `packages/contracts`：共享的 effect/Schema 模式和 TypeScript 契约，用于提供程序事件、WebSocket 协议以及模型/会话类型。保持此包仅为模式定义——不包含运行时逻辑。
- `packages/shared`：服务器和 Web 共享的运行时工具。使用显式子路径导出（例如 `@t3tools/shared/git`）——没有 barrel index。

## 本地开发实例隔离

- 除非用户明确希望共享端口/状态，否则当另一个 Synara 实例正在运行时，不要启动默认的 `bun run dev`。
- 当与用户自己的 Synara 实例并排运行时，请使用隔离的主目录和非默认端口，例如：`env -u T3CODE_AUTH_TOKEN T3CODE_PORT_OFFSET=3158 T3CODE_NO_BROWSER=1 bun run dev -- --home-dir ./.synara-pr84 --port 58090`。
- 为避免冲突，始终先进行干运行：`env -u T3CODE_AUTH_TOKEN T3CODE_PORT_OFFSET=3158 bun run dev -- --home-dir ./.synara-pr84 --port 58090 --dry-run`。
- 除非 Web 应用也配置为使用该令牌连接，否则为浏览器开发实例取消设置 `T3CODE_AUTH_TOKEN`。如果不小心继承了身份验证，浏览器 WebSocket 可能会被拒绝，UI 将不会显示任何线程，即使 SQLite 中有项目/线程。
- 使用 `lsof -nP -iTCP:<port> -sTCP:LISTEN` 检查服务器和 Web 端口。桌面应用可以绑定 `127.0.0.1:<port>`，而开发服务器绑定 IPv6 `*:<port>`，并且 `localhost` 仍可能命中错误的进程。
- 如果 UI 不显示任何线程，在更改 SQL 之前先验证服务器路径：检查隔离的 `state.sqlite`，然后通过 WebSocket 探测 `orchestration.getSnapshot`。包含项目/线程的健康快照意味着问题是客户端连接/水合，而不是空历史记录。

## Codex App Server（重要）

Synara 目前以 Codex 为先。服务器为每个提供程序会话启动 `codex app-server`（通过 stdio 的 JSON-RPC），然后通过 WebSocket 推送消息将结构化事件流式传输到浏览器。

我们在该代码库中的使用方式：

- 会话启动/恢复和轮次生命周期由 `apps/server/src/codexAppServerManager.ts` 代理。
- 提供程序调度和线程事件日志记录由 `apps/server/src/providerManager.ts` 协调。
- WebSocket 服务器在 `apps/server/src/wsServer.ts` 中路由 NativeApi 方法。
- Web 应用通过 WebSocket 在通道 `orchestration.domainEvent` 上消费编排领域事件（提供程序运行时活动会在服务器端投影为编排事件）。

文档：

- Codex App Server 文档：https://developers.openai.com/codex/sdk/#app-server

## 参考仓库

- 开源 Codex 仓库：https://github.com/openai/codex
- Codex-Monitor（Tauri，功能完整，强参考实现）：https://github.com/Dimillian/CodexMonitor

在设计协议处理、用户体验流程和操作保障时，请将这些作为实现参考。
