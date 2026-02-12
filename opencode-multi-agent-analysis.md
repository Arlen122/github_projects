# OpenCode 多 Agent 系统分析

本文基于 [OpenCode Agents 文档](https://opencode.ai/docs/agents/) 与代码库（`packages/opencode`）梳理主 agent 与 subagent 的调用关系、上下文与结果回传机制。

---

## 1. 主 Agent 如何调用 Subagent：Task 工具

主 agent 调用 subagent 的**唯一方式**是使用内置的 **Task 工具**（`task`）。文档里没有单独列出 “Task” 或 “delegate”，是因为它作为普通工具之一暴露给模型，主 agent 在需要“让专门 agent 干活”时会**主动选择调用 Task 工具**。

### 1.1 Task 工具在哪定义

- **实现**：`packages/opencode/src/tool/task.ts`（`TaskTool`）
- **注册**：`packages/opencode/src/tool/registry.ts` 中与 `BashTool`、`ReadTool` 等一起注册，主 agent（如 Build）默认拥有该工具。

### 1.2 Task 工具的“说明书”如何生成

Task 工具对模型的描述是动态生成的（见 `task.ts` 和 `task.txt`）：

1. 拉取所有 **非 primary** 的 agent（即所有 subagent）。
2. 若当前有 caller agent，则按 **Task 权限**（`permission.task`）过滤掉被 deny 的 subagent。
3. 用这些 subagent 的 `name` 和 `description` 拼成列表，替换 `task.txt` 里的 `{agents}`，形成最终工具描述。

因此模型看到的是：“你可以用 Task 工具，并指定 `subagent_type` 为下面之一：general: ...; explore: ...; …”。

### 1.3 Task 工具的参数（模型如何调用）

```ts
// task.ts 中 z.object
{
  description: string,   // 3–5 词任务简述
  prompt: string,       // 交给 subagent 的完整任务说明
  subagent_type: string, // 要调用的 subagent 名称，如 "general" / "explore"
  session_id?: string,  // 可选：要复用的已有 Task 子 session，用于延续上下文
  command?: string,     // 可选：触发该任务的命令名（如 slash command）
}
```

- **主 agent 调用方式**：在某一轮中发出 tool call，例如  
  `task(description="搜索用法", prompt="在 src/ 下查找 foo 的调用", subagent_type="explore")`。
- **带上下文延续**：若传入上次返回的 `session_id`，则在该子 session 上继续，实现“多次调用同一 subagent 并继承上下文”（见下节）。

### 1.4 权限与可见性

- **Task 权限**（`permission.task`）控制主 agent **能否**调用某个 subagent，以及是直接执行还是需要用户确认（allow / deny / ask）。  
  文档中的 “Task permissions” 对应代码里的 `PermissionNext.evaluate("task", a.name, caller.permission)` 与配置中的 `permission.task`。
- 若某 subagent 被配置为 `hidden: true`，它不会出现在 @ 的自动补全里，但若权限允许，主 agent 仍可通过 Task 工具按名称调用（见文档 “Hidden” 说明）。

---

## 2. 手动 @ 调用与 Task 的关系

用户通过 **@ 提及某个 subagent** 时，流程是“当前会话中下一轮由该 subagent 负责回复”，而不是直接创建子 session。

### 2.1 用户 @ subagent 时发生了什么

1. **用户消息**中会带一个 `type: "agent"` 的 part，`name` 为被 @ 的 subagent（如 `general`）。
2. **创建用户消息**时（`prompt.ts` 中 `createUserMessage`）：
   - 若请求里带了 `agent`（即 @ 选中的 agent），则用户消息的 `agent` 字段会被设为该 subagent。
   - 同时若存在 agent part，会注入一段 **合成系统提示**：  
     “Use the above message and context to generate a prompt and call the task tool with subagent: &lt;name&gt;”。
3. **下一轮回复**时，`lastUser.agent` 就是这个 subagent，因此**回复者**是该 subagent（在**当前同一 session** 里）。
4. 该 subagent 会看到“用户让我干活”+ 上面那段“用 task 工具调用自己”的提示，因此典型行为是：**再调用一次 Task 工具**，指定 `subagent_type` 为自己，把用户内容放进 `prompt`。这样才会真正创建一个**子 session**，在子 session 里执行任务。

所以：

- **手动 @**：在父 session 里“切换”到该 subagent 来回复；该 subagent 通常再通过 **Task 工具**把实际工作放到子 session。
- **主 agent 自主调用**：主 agent 直接调用 Task 工具，指定 `subagent_type`，不经过 @。

两种路径最终都通过 **Task 工具** 创建/复用子 session 并跑 subagent。

---

## 3. Subagent 的上下文是否独立

**是，完全独立。**  
每次 Task 工具执行时，都会基于一个**子 session**（child session）运行 subagent，该 session 与主 session 隔离。

### 3.1 子 Session 的创建与父子关系

- 在 `task.ts` 的 `execute` 中：
  - 若未传 `session_id`，会 `Session.create({ parentID: ctx.sessionID, ... })`，即**当前主 session 的 ID** 作为 `parentID`。
  - 若传了 `session_id` 且能找到已有 session，则**复用**该 session，不再新建。
- 子 session 有独立：
  - `id`、消息列表、对话历史
  - 权限（可对子 session 单独设置 `permission`，例如禁止 todo、禁止再调 task 等）
  - 标题：默认带 “(@&lt;agent.name&gt; subagent)”。

因此：**每次 Task 调用 = 在（新或已有的）子 session 里跑 subagent**，和主 session 的上下文是分开的。

### 3.2 多次调用同一 Subagent 能否继承上一次的上下文

**可以。** 通过 Task 的 **`session_id`** 参数。

- Task 工具返回的 `output` 里会带一段元数据（见 `task.ts`）：
  ```text
  <task_metadata>
  session_id: <session.id>
  </task_metadata>
  ```
- 主 agent 若在后续某次调用中把这段 `session_id` 再传给 Task 工具（即 `session_id: <上次返回的 id>`），则不会新建 session，而是在**同一个子 session** 上继续，从而继承该 subagent 之前的对话与状态。

文档与 `task.txt` 里也写明：  
“Each agent invocation is stateless unless you provide a session_id.”

---

## 4. Subagent 能否“交互”

- **在子 session 内部**：可以。子 session 有自己完整的消息历史，subagent 可以在该 session 里多轮对话、多次用工具，和普通会话一样。
- **与主 session / 用户**：没有直接的“对话式交互”。主 session 只能通过 **Task 工具的一次执行** 把 `prompt` 丢进去，等子 session 跑完，拿到**一次**返回值（见下节）。若要让 subagent 再根据主 agent 的反馈干活，需要主 agent 再次调用 Task，并视情况传入 `session_id` 以延续上下文。

所以：subagent 的“交互”发生在子 session 内；和主 agent 之间是“一次任务 → 一次结果”的调用关系，除非主 agent 多次发起 Task 并传 `session_id` 做多轮协作。

---

## 5. 任务结果如何回到主 Agent

**通过 Task 工具的返回值。**  
子 session 跑完后，`task.ts` 的 `execute` 会：

1. 从子 session 的最后一条 assistant 消息里取**最后一段 text** 作为主要可读结果。
2. 汇总该 session 内 assistant 的 tool 调用信息（id、tool、status、title 等）作为 `metadata.summary`。
3. 将 `output` 设为：上述 text + 带 `session_id` 的 `<task_metadata>...</task_metadata>`。
4. `return { title, metadata, output }`。

这些会作为**主 session 中该轮 tool call 的 result** 写回主 session 的 assistant 消息（ToolPart 的 state.output / metadata）。因此：

- **主 agent 的下一轮**能看到这次 Task 的完整 `output`（含子 session 的文本结果和 `session_id`）。
- 主 agent 可以据此总结、再决策、或把 `session_id` 用于下一次 Task 调用以延续子 session 上下文。

文档中也说明：  
“The result returned by the agent is not visible to the user. To show the user the result, you should send a text message back to the user with a concise summary of the result.”  
即：结果先回到主 agent 的上下文，由主 agent 再决定是否用一段文字回复给用户。

---

## 6. 会话导航：父 / 子 Session 在 UI 上的关系

- **TUI**：`session_child_cycle`（默认 `<leader>+Right`）、`session_child_cycle_reverse`（`<leader>+Left`）、`session_parent`（`<leader>+Up`）用于在**当前会话的父 session 与各子 session** 之间切换查看。
- **路由**：`Session.children(parentID)`、`Session.get(sessionID)` 等用于按 `parentID` 列出子 session、跳转到子 session 或回到父 session。
- Task 工具在 UI 上会展示子 session 的入口（如“view subagents”），便于用户点进对应子 session 查看 subagent 的完整对话。

---

## 7. 小结表

| 问题 | 结论 |
|------|------|
| 主 agent 通过什么调用 subagent？ | **Task 工具**（`task`），参数包括 `subagent_type`、`prompt`、可选的 `session_id` 等。 |
| 有没有单独的 “delegate/task” 文档入口？ | 没有独立章节，Task 作为普通工具在 [Agents](https://opencode.ai/docs/agents/) 的 Tools/Permissions 与 “Use cases” 中体现；代码里就是 `ToolRegistry` 中的 `TaskTool`。 |
| Subagent 上下文是否独立？ | **是**，每次 Task 对应一个（新或复用的）**子 session**，与主 session 隔离。 |
| 多次调用同一 subagent 能继承上下文吗？ | **能**，通过 Task 的 **`session_id`** 复用同一子 session。 |
| Subagent 能否交互？ | 在**子 session 内**可多轮交互；与主 agent 之间是**单次任务 → 单次结果**，除非主 agent 多次调用 Task 并传 `session_id`。 |
| 任务结果会回主 agent 吗？ | **会**，作为 Task 工具的 **return 值**（含 `output` 与 `metadata`）写回主 session 的 assistant 消息，主 agent 下一轮可见并可再处理。 |

---

## 8. 代码索引（便于对照）

- Task 工具定义与执行：`packages/opencode/src/tool/task.ts`
- Task 工具描述模板：`packages/opencode/src/tool/task.txt`
- 主循环中“待执行的 subtask”处理（创建 tool part、调用 TaskTool.execute）：`packages/opencode/src/session/prompt.ts`（约 319–460 行，以及 command 中 `isSubtask` / `type: "subtask"`）
- 用户消息中 agent part 与“用 task 调用 subagent”的注入：`packages/opencode/src/session/prompt.ts`（约 1150–1175 行、`createUserMessage`）
- Session 父子关系与 children：`packages/opencode/src/session/index.ts`（`parentID`、`Session.children`）
- Task 权限与禁用逻辑：`packages/opencode/test/permission-task.test.ts`、`packages/opencode/src/permission/arity.ts` / `PermissionNext`
- TUI 中 session 切换与 Task 展示：`packages/opencode/src/cli/cmd/tui/routes/session/index.tsx`（如 `moveChild`、Task 块展示）

---

*分析基于当前 OpenCode 代码库与 [Agents 文档](https://opencode.ai/docs/agents/) 整理，若有配置或版本差异请以实际仓库与文档为准。*
