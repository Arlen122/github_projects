# Oh-My-OpenCode 多 Agent 系统分析

本文基于 **oh-my-opencode** 代码库，分析其如何通过 OpenCode 的 **Plugin / Hook 机制** 实现自定义多 agent 系统，并回答与 OpenCode 分析文档中相同的一组问题：主 agent 如何调用 subagent、上下文是否独立、是否可继承、是否可交互、结果如何回传等。

---

## 1. Oh-My-OpenCode 与 OpenCode 的集成方式

### 1.1 作为 OpenCode 插件

oh-my-opencode 是一个 **OpenCode 插件**（Plugin）。入口为 `src/index.ts`，导出一个符合 `@opencode-ai/plugin` 的 **Plugin** 函数：

```ts
const OhMyOpenCodePlugin: Plugin = async (ctx) => { ... }
export default OhMyOpenCodePlugin
```

OpenCode 从配置的 `plugin` 列表（如 `oh-my-opencode@latest`）加载模块，对所有导出的函数执行 `await fn(input)`，其中 `input` 为 `PluginInput`（含 `client`、`directory`、`project`、`worktree` 等）。插件返回一个 **Hooks** 对象，OpenCode 在相应生命周期调用这些 hook。

### 1.2 使用的 Hook / 扩展点

oh-my-opencode 使用了 OpenCode 提供的以下扩展点（见 `index.ts` 的 return）：

| 扩展点 | 用途 |
|--------|------|
| **tool** | 注册自定义工具（见下节）。与 OpenCode 内置工具（如 `task`、`read`、`bash`）一起出现在模型的工具列表中。 |
| **chat.message** | 用户消息提交后、模型回复前：可修改 message、variant、以及触发 keyword-detector、ralph-loop、start-work 等逻辑。 |
| **event** | 订阅 OpenCode 事件（如 `session.created`、`session.deleted`、`message.updated`、`session.error`），用于 tmux 集成、主 session 追踪、恢复、通知等。 |
| **tool.execute.before** | 任一工具执行前：可修改 `output.args`（例如为原生 `task` 注入 `delegate_task: false`）、实现 subagent-question-blocker、delegate-task-retry 等。 |
| **tool.execute.after** | 工具执行后：用于 truncator、context-window-monitor、delegate-task-retry、task-resume-info 等。 |
| **experimental.chat.messages.transform** | 发给模型的消息列表转换（如 context-injector、thinking-block-validator）。 |
| **experimental.session.compacting** | 会话压缩时注入上下文（compaction-context-injector）。 |
| **config** | 提供配置处理（config-handler），与 OpenCode 配置合并。 |

**结论**：多 agent 的“调用”不是通过 OpenCode 内置的 Task 工具 alone，而是通过插件注册的 **新工具**（`delegate_task`、`call_omo_agent`、`background_task` 等）以及在这些工具内部通过 **OpenCode 的 `ctx.client`**（session.create、session.prompt、session.messages）创建子 session、发送 prompt、轮询结果。Hook 用于在工具执行前后做策略调整（如对原生 `task` 禁用 `delegate_task`）、以及与会话/事件联动。

---

## 2. 主 Agent 如何调用 Subagent：三种工具

oh-my-opencode 提供了 **三种** 与多 agent 相关的工具，主 agent（或用户）通过 **调用这些工具** 来“调用” subagent。

### 2.1 delegate_task（核心：分类 / 直接指定 agent）

- **定义**：`src/tools/delegate-task/tools.ts`（`createDelegateTask`），在插件中注册为 `delegate_task`。
- **参数**（节选）：
  - `load_skills`: 必填，要注入的 skill 名称数组（可为 `[]`）。
  - `description`、`prompt`：任务简述与完整提示。
  - **二选一**：`category`（预定义分类，如 `general`、`quick`）或 `subagent_type`（直接指定 agent 名，如 `oracle`、`explore`）。
  - `run_in_background`：`true` = 异步（返回 task_id，不阻塞）；`false` = 同步（等待子 session 完成后再返回结果）。
  - `session_id`：可选，用于**延续已有子 session**（与 OpenCode 原生 Task 的 `session_id` 语义一致）。
- **行为概要**：
  - 若有 `session_id`：根据 `run_in_background` 走 **executeSyncContinuation** 或 **executeBackgroundContinuation**（向该 session 再发一条 prompt，轮询或仅登记后台任务）。
  - 若用 `category`：解析出使用的 agent（如 Sisyphus-Junior）和模型/技能配置，再根据 `run_in_background` 走 **executeSyncTask** / **executeBackgroundTask** 或 **executeUnstableAgentTask**（不稳定模型会强制后台并轮询）。
  - 若用 `subagent_type`：解析出可调用的 subagent（通过 `client.app.agents()` 过滤非 primary），再同样走 sync/background。
- **子 session 创建**：通过 `client.session.create({ body: { parentID: parentContext.sessionID, title: `... (@${agentToUse} subagent)`, permission: [...] }, query: { directory } })` 创建，与 OpenCode 原生 Task 的父子 session 关系一致。
- **结果回传**：
  - **同步**：轮询子 session 的 `session.status()` 与 `session.messages()`，直到会话空闲且消息数稳定，取最后一条 assistant 消息的 text/reasoning 拼接为输出，并在末尾附 `<task_metadata>session_id: ...</task_metadata>`，作为工具返回值回到主 agent。
  - **后台**：立即返回 task_id 和 session_id，主 agent 之后可用 `background_output(task_id=...)` 取结果。

**主 agent 通过“调用 `delegate_task` 工具”来调用 subagent**；没有单独的 delegate API，就是工具调用。

### 2.2 call_omo_agent（仅 explore / librarian）

- **定义**：`src/tools/call-omo-agent/tools.ts`（`createCallOmoAgent`），注册为 `call_omo_agent`。
- **限制**：`ALLOWED_AGENTS = ["explore", "librarian"]`，仅允许这两种 agent。
- **参数**：`description`、`prompt`、`subagent_type`（必须为 explore 或 librarian）、`run_in_background`（必填）、可选 `session_id`。
- **行为**：
  - **后台**：通过 `BackgroundManager.launch(...)` 启动后台任务，立即返回 task_id 等说明。
  - **同步**：若无 `session_id` 则 `client.session.create({ parentID, title, permission })` 创建子 session，再 `client.session.prompt({ path: { id: sessionID }, body: { agent, tools: { task: false, delegate_task: false }, parts } })` 发送 prompt；然后轮询 `session.status()` 与 `session.messages()` 判定完成，从消息中抽取 text/reasoning/tool_result 拼成输出，并附 `<task_metadata>session_id: ...</task_metadata>`。
  - 若有 `session_id`：在同一子 session 上继续发 prompt（同步），同样轮询后返回结果。
- **与 OpenCode 原生 task 的关系**：在 `tool.execute.before` 中，当 `input.tool === "task"` 且 `subagent_type` 为 explore 或 librarian 时，会**禁用** `call_omo_agent`（避免子 session 里再调 call_omo_agent 形成多余嵌套）。见 `index.ts` 451–462 行。

**主 agent 通过“调用 `call_omo_agent` 工具”来专门调用 explore 或 librarian**；同样没有独立 API，只有工具。

### 2.3 background_task / background_output / background_cancel

- **定义**：`src/tools/background-task/tools.ts`。
  - `background_task`：接收 `description`、`prompt`、`agent`（任意已注册 agent），通过 `BackgroundManager.launch(...)` 将任务入队，**不等待**完成，立即返回 task_id 和“用 background_output 查结果”的说明。
  - `background_output`：根据 task_id 取任务状态或完整结果（可 block 等待完成、可 full_session 等）。
  - `background_cancel`：按 task_id 或 all 取消运行中/排队中的后台任务。
- **子 session**：由 `BackgroundManager` 在内部按并发配置创建子 session（`client.session.create`）、发送 prompt（`client.session.prompt`），与 delegate_task 的 sync 路径类似，但完全异步，结果通过事件/通知和 `background_output` 获取。
- **用途**：并行跑多个 agent（如“让 GPT 调试的同时让 Claude 试另一种方案”），主 agent 继续做别的事，需要时再 `background_output` 拿结果。

**主 agent 通过“调用 `background_task`”来异步调用任意 agent**，通过 `background_output` 取结果。

---

## 3. Subagent 的上下文是否独立

**是，完全独立。**  
与 OpenCode 原生 Task 一致，每次“派发”都会基于 **子 session**：

- **delegate_task（sync）**：`executeSyncTask` 中 `client.session.create({ body: { parentID: parentContext.sessionID, ... } })`，子 session 拥有独立 id、消息历史、权限（如 `question: deny`）。
- **call_omo_agent（sync）**：同样 `client.session.create({ parentID: toolContext.sessionID, ... })`，子 session 独立。
- **background_task / delegate_task( run_in_background=true )**：由 `BackgroundManager` 在内部创建子 session，同样带 `parentID`，与主 session 隔离。

因此：**每个 subagent 任务对应一个独立的子 session**，与主 session 及其他子 session 的上下文隔离。

---

## 4. 多次调用同一 Subagent 能否继承上下文

**可以。** 通过 **session_id** 延续同一子 session：

- **delegate_task**：若传入 `session_id`（来自上次返回的 `<task_metadata>session_id: ...</task_metadata>`），则不会新建 session：
  - **同步**：`executeSyncContinuation` 向该 session 发 `client.session.prompt({ path: { id: args.session_id }, body: { agent: resumeAgent, tools: {...}, parts: [{ type: "text", text: args.prompt }] } })`，在该 session 的已有历史上追加一轮，轮询后返回新结果。
  - **后台**：`executeBackgroundContinuation` 调用 `manager.resume({ sessionId: args.session_id, prompt, ... })`，在同一个 session 上继续后台执行。
- **call_omo_agent**：若传入 `session_id`，则 `executeSync` 中直接使用该 session，再发 prompt，同样是在同一子 session 的上下文中继续。

文档与工具描述中均强调：用 `session_id` 可 “continue with full context preserved”、“saves tokens, maintains continuity”。

---

## 5. Subagent 能否“交互”

- **在子 session 内部**：可以。子 session 有完整消息历史，subagent 可在该 session 内多轮推理、多次调用工具（read、grep、glob 等），与普通会话一致。
- **与主 session / 主 agent**：没有持续的“对话流”。主 agent 通过**单次工具调用**传入 `prompt`（及可选的 `session_id`），子 session 跑完后：
  - **同步**：工具返回值即子 session 的产出（文本 + `<task_metadata>session_id: ...`），主 agent 在下一轮看到该工具结果。
  - **后台**：主 agent 收到 task_id，之后用 `background_output(task_id)` 拉取结果；系统还可通过 notification hook 在完成时通知。

若需“多轮协作”，主 agent 需要**多次**调用 `delegate_task` 或 `call_omo_agent`，并传入**同一个 session_id**，从而在同一子 session 上延续对话。

---

## 6. 任务结果如何回到主 Agent

- **同步路径**（delegate_task run_in_background=false、call_omo_agent run_in_background=false）：
  - 工具内部轮询子 session 直到“空闲 + 消息稳定”，然后取最后一条（或若干条）assistant 消息的 text/reasoning（以及部分 tool_result），拼成字符串，并追加 `<task_metadata>session_id: ...</task_metadata>`。
  - 该字符串作为**工具的 return 值**返回给 OpenCode，OpenCode 将其写入主 session 中该轮 tool call 的 **ToolPart state.output**，主 agent 下一轮即可看到该工具结果。
- **后台路径**（delegate_task run_in_background=true、call_omo_agent run_in_background=true、background_task）：
  - 工具立即返回，内容为 task_id、session_id 以及“用 background_output 查结果”的说明。
  - 主 agent 在后续任意时刻调用 **background_output(task_id=...)**，取回该任务的输出（同样来自子 session 的消息聚合），该输出再次作为工具结果进入主 session 上下文。

因此：**结果要么直接作为同步工具的返回值回到主 agent，要么通过 background_output 工具再次以工具结果形式回到主 agent**。

---

## 7. 与 OpenCode 原生 Task 的关系

- oh-my-opencode **额外注册**了 `delegate_task`、`call_omo_agent`、`background_task` 等工具，与 OpenCode 原生的 **task** 工具并存；主 agent 可以选择用原生 `task` 或用 `delegate_task` / `call_omo_agent` / `background_task`。
- 在 **tool.execute.before** 中，当**原生** `task` 被调用时，oh-my-opencode 会修改其参数：
  - 对**所有**原生 task 调用：注入 `delegate_task: false`，避免子 session 里的 agent 再调 delegate_task。
  - 当原生 task 的 `subagent_type` 为 explore 或 librarian 时，再注入 `call_omo_agent: false`。
  这样既保留原生 task 的用法，又避免在子 session 内重复派发导致混乱。
- **Session 创建与调用方式**：oh-my-opencode 的工具内部使用的是同一套 OpenCode 客户端 API（`client.session.create`、`client.session.prompt`、`client.session.messages`），与原生 Task 工具创建子 session、发 prompt、取结果的方式一致，只是**策略不同**（分类、技能注入、同步/后台、轮询与超时等由 oh-my-opencode 自己实现）。

---

## 8. 小结表

| 问题 | 结论 |
|------|------|
| 主 agent 通过什么调用 subagent？ | 通过 **插件注册的工具**：**delegate_task**（分类或直接 subagent_type）、**call_omo_agent**（仅 explore/librarian）、**background_task**（任意 agent、纯异步）。无单独 delegate API，就是工具调用。 |
| 如何与 OpenCode 集成？ | 作为 **OpenCode Plugin**，使用 **tool**、**chat.message**、**event**、**tool.execute.before/after** 等 Hook；在工具内部通过 **ctx.client**（session.create / prompt / messages）创建子 session 并驱动 subagent。 |
| Subagent 上下文是否独立？ | **是**。每次派发都对应一个 **子 session**（parentID 指向主 session），与主 session 及其他子 session 隔离。 |
| 多次调用同一 subagent 能继承上下文吗？ | **能**。通过 **session_id** 参数延续同一子 session（delegate_task 与 call_omo_agent 均支持），在同一 session 上追加 prompt 并轮询或后台执行。 |
| Subagent 能否交互？ | **子 session 内**可多轮；与主 agent 之间是 **单次任务 → 单次结果**（或后台 + background_output），需主 agent 多次调用并传 session_id 才能实现多轮协作。 |
| 任务结果如何回主 agent？ | **同步**：工具返回值（含 `<task_metadata>session_id: ...`）作为该次 tool call 的 result 写回主 session。**后台**：通过 **background_output(task_id)** 再次以工具结果形式取回并进入主 session 上下文。 |

---

## 9. 代码索引（便于对照）

- 插件入口与 Hook 注册：`src/index.ts`
- delegate_task 定义与分类/执行逻辑：`src/tools/delegate-task/tools.ts`、`executor.ts`、`categories.ts`、`prompt-builder.ts`
- call_omo_agent 定义与 sync/background：`src/tools/call-omo-agent/tools.ts`、`constants.ts`
- background 任务与输出/取消：`src/tools/background-task/tools.ts`；`src/features/background-agent/manager.ts`
- 对原生 task 的 tool.execute.before 修改：`src/index.ts`（约 451–462 行）
- Session 创建与 prompt 调用：`executor.ts`（executeSyncTask、executeSyncContinuation）、`call-omo-agent/tools.ts`（executeSync）、BackgroundManager

---

*分析基于 oh-my-opencode 代码库整理；若与最新版本有差异，请以实际仓库为准。*
