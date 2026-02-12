# Claude Code 多 Agent 系统分析

本文基于 [Claude Code 官方文档](https://code.claude.com/docs)（[Sub-agents](https://code.claude.com/docs/en/sub-agents)、[Agent teams](https://code.claude.com/docs/en/agent-teams)）及社区分析（[Claude Code Agent Teams 机制研究](https://www.cnblogs.com/pDJJq/p/19592455/research-on-claude-code-agent-teams-mechanism-17xegk)），梳理 **Subagent** 与 **Agent Teams** 的调用关系、上下文与通信机制，并重点分析两者的差异。

---

## 1. 概述：Subagent 与 Agent Teams 的定位

Claude Code 在文档中将 **Subagents** 与 **Agent Teams** 放在同一层级（与 Skills、Plugins 等并列），均为「Build with Claude Code」下的核心能力。

| 概念 | 一句话 | 典型用途 |
|------|--------|----------|
| **Subagent** | 单会话内、由主 agent 通过 Task 工具派发的**专用子 agent**，拥有独立 context window，结果**汇总回主会话**。 | 探索代码库、做计划前调研、隔离高吞吐操作、并行研究后汇总。 |
| **Agent Teams** | **多会话**：一个 Lead 会话 + 多个 Teammate 会话，通过**共享任务列表**和 **Mailbox（SendMessage）** 协作；Teammate 之间可**直接通信**。 | 多角度并行评审、竞争假设调试、跨层协作、需要「讨论与挑战彼此结论」的复杂任务。 |

官方表述：  
> Subagents work within a single session; agent teams coordinate across separate sessions.  
> 若需要多个 agent **并行且互相通信**，用 Agent Teams；若只需「派活 → 拿结果」，用 Subagents。

---

## 2. Subagent 机制

### 2.1 主 Agent 如何调用 Subagent：Task 工具

- **唯一入口**：主 agent 通过内置 **Task 工具** 调用 subagent，指定 `subagent_type`（或等价参数）、`description`、`prompt` 等。
- **参数**（节选）：`description`、`prompt`、`subagent_type`、可选 `session_id`（延续同一子会话）。无 Agent Teams 时，Task 不涉及 `team_name`、`name`（spawn 出的 agent 名）、`mode`。
- **权限**：主 agent 对某 subagent 的调用由 **Task 权限**（如 `Task(Explore)`、`Task(my-custom-agent)`）及 settings 中的 `deny` / `disallowedTools` 控制；可用 `Task(agent_type)` 限制主 agent 只能 spawn 指定类型的 subagent。

### 2.2 Subagent 的上下文与结果回传

- **上下文**：每个 Task 调用对应一个**子会话**（或复用 `session_id` 的已有子会话），拥有独立的 context window、system prompt（subagent 自己的配置）、工具集与权限。与主会话**完全隔离**。
- **继承上下文**：通过 Task 返回的 `<task_metadata>session_id: ...</task_metadata>` 在后续 Task 调用中传入同一 `session_id`，即可在同一 subagent 会话上延续。
- **结果回传**：子会话跑完后，**仅将结果**（如最后一段 text + metadata）作为 **Task 工具的返回值** 写回主会话；主 agent 在下一轮看到该 tool result。Subagent 的完整对话与中间过程**不会**自动流入主会话，从而节省 context、保护主会话焦点。

### 2.3 内置 Subagent 与自定义配置

- **内置**：Explore（只读、Haiku）、Plan（只读、继承主模型）、General-purpose（全工具、继承主模型），以及 Bash、statusline-setup、Claude Code Guide 等辅助 agent。
- **自定义**：通过 Markdown + YAML frontmatter 定义（`name`、`description`、`tools` / `disallowedTools`、`model`、`permissionMode`、`skills`、`memory`、`hooks` 等），可放在项目 `.claude/agents/`、用户 `~/.claude/agents/`、或通过 CLI `--agents`、插件提供。
- **前台/后台**：Subagent 可跑在**前台**（阻塞主会话直到完成）或**后台**（并发执行，事先征得权限，结果后续可见或通过 resume 处理）。

---

## 3. Agent Teams 机制

Agent Teams 为**实验功能**，需设置 `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` 启用。

### 3.1 核心组件（与 Subagent 的架构差异）

| 组件 | 角色 |
|------|------|
| **Team lead** | 创建团队、spawn Teammate、分配/更新任务、通过 SendMessage 协调，可运行在 **delegate 模式**（仅做协调、不写代码）。 |
| **Teammates** | **独立的 Claude Code 实例**（各自独立会话），执行被分配或自己认领的任务，通过 **SendMessage** 与 Lead 及其他 Teammate 通信。 |
| **Task list** | 共享任务列表（pending / in progress / completed），支持依赖（blockedBy）；Lead 与 Teammate 均可查看、认领、更新。 |
| **Mailbox** | 基于 **SendMessage** 的消息系统：私信、广播、关闭请求/响应、计划审批等。 |

与 Subagent 的关键区别：**Teammate 是独立进程/会话**，不共享主会话的 context；**通信靠显式 SendMessage**，而不是「Task 一次调用 → 一次返回值」。

### 3.2 Lead 侧：Task 与 Team 相关工具的变化（参考博客）

启用 Agent Teams 后，Lead 的 System Prompt 中 Task 与任务相关工具会扩展：

- **Task（spawn Teammate）**  
  - 新增参数：`name`（spawn 出的 agent 名）、`team_name`（团队名，省略则用当前团队上下文）、`mode`（如 `plan` 要求先审批计划）。  
  - 用于创建并启动 **Teammate 会话**，而非单会话内的 subagent 子会话。

- **TaskCreate**  
  - 描述中强调任务「可能分配给 Teammate」；新任务状态为 `open`、无 `owner`，需用 **TaskUpdate** 的 `owner` 参数分配。

- **TaskList**  
  - 增加「分配任务给 Teammate 前先看列表」的用法；输出侧增加与 **assignTask** 的配合说明，以及 **Teammate Workflow**：完成当前任务后查 TaskList → 找 pending、无 owner、无阻塞的任务 → 用 **claimTask** 认领或等 Lead 分配。

- **TeamCreate / TeamDelete**  
  - 创建/删除团队；团队配置与任务目录分别落在 `~/.claude/teams/{team-name}/` 与 `~/.claude/tasks/{team-name}/`。TeamDelete 前须先关闭所有 Teammate。

- **SendMessage**  
  - 类型：`message`（私信）、`broadcast`（广播，成本高）、`shutdown_request` / `shutdown_response`、`plan_approval_response`。  
  - 所有 Teammate 间及与 Lead 的协调都通过 SendMessage 显式传递；**仅写文字不会对其他人可见**，必须用工具。

### 3.3 Teammate 侧：System Prompt 与工具

- **System Prompt**：在原有基础上增加 **Agent Teammate Communication** 章节，强调：用 SendMessage（type `message` / 谨慎使用 `broadcast`）与团队沟通；用户主要与 Lead 交互；工作由任务系统与消息协调。
- **工具**：Teammate 的工具集相对 Lead 有裁剪（无完整 Team 管理能力），侧重任务执行与 **SendMessage**；任务状态更新通过 TaskUpdate 等与共享 Task list 同步。
- **Personality/角色**：由 Lead 在 spawn 时通过 **prompt** 注入（如「你负责 UX 分析」「你扮演 devil’s advocate」）；Teammate 仍会加载项目中的 **CLAUDE.md** 等上下文。

### 3.4 任务分配与通信模式

- **分配方式**：**Lead assigns**（Lead 用 TaskUpdate 指定 `owner`）或 **Self-claim**（Teammate 完成当前任务后从 TaskList 中认领下一个可用的 pending 任务，常用 `claimTask`）。  
- **通信**：Teammate 与 Lead、Teammate 与 Teammate 之间通过 **SendMessage** 传递发现、结论、关闭请求、计划审批等；消息自动送达，无需 Lead 轮询。  
- **Delegate 模式**：Lead 可切换为仅做协调（spawn、消息、任务、关闭），不直接写代码，避免 Lead 抢活。

---

## 4. Subagent 与 Agent Teams 的差异分析（重点）

下面从**会话与上下文**、**通信方式**、**协调方式**、**适用场景**与**成本**五方面对比，并给出一张小结表。

### 4.1 会话与上下文模型

| 维度 | Subagent | Agent Teams |
|------|----------|-------------|
| **会话边界** | **单会话内**：主会话 + 多个子会话（每个 Task 对应一个子 session，或复用 session_id）。 | **多会话**：一个 Lead 会话 + 多个**独立 Claude Code 实例**（每个 Teammate 一个会话）。 |
| **上下文隔离** | 子会话有独立 context window、system prompt、工具集；与主会话隔离。 | 每个 Teammate 有**完全独立**的 context window；**不共享** Lead 的对话历史；仅通过 spawn prompt、CLAUDE.md、任务列表与消息交换信息。 |
| **主会话可见性** | 主 agent 通过 **Task 返回值** 看到 subagent 的**汇总结果**；subagent 的完整推理与中间过程不进入主会话。 | Lead **看不到** Teammate 的完整执行过程，仅通过 **SendMessage** 与任务状态更新获得信息；用户可与任一 Teammate 直接对话（如 Shift+Up/Down 选会话）。 |

因此：Subagent = 「主会话 + 若干子会话，结果汇总回主」；Agent Teams = 「多会话并列，靠任务表 + 消息协作」。

### 4.2 通信方式

| 维度 | Subagent | Agent Teams |
|------|----------|-------------|
| **主 ↔ 子/Teammate** | **单向、调用/返回**：主 agent 调用 Task(prompt, subagent_type, …) → 子会话执行 → **一次** return（output + session_id）。若需多轮，主 agent 再次调用 Task 并传 session_id。 | **双向、显式消息**：Lead 通过 **SendMessage**（message / broadcast / shutdown_request / plan_approval_response）与 Teammate 通信；Teammate 同样用 SendMessage 向 Lead 或其它 Teammate 汇报、讨论、请求关闭或计划审批。 |
| **子/Teammate 之间** | **无**。Subagent 之间不直接通信；所有协调经主 agent（主 agent 可多次调不同 Task，再综合结果）。 | **有**。Teammate 之间可直接 **SendMessage**（私信或广播），用于讨论、挑战彼此结论等。 |
| **信息粒度** | 主 agent 收到的是 Task 工具返回的**摘要/结果**，可控制子会话「只把关键信息」带回。 | 信息完全由**消息内容**决定；若 Teammate 在回复里写得很详细但 SendMessage 只发概括，会出现「内容损失」（博客中提到的不足）。 |

结论：Subagent 是「调用 → 一次结果」的**函数式**协作；Agent Teams 是「多主体 + 消息总线」的**分布式协作**。

### 4.3 协调方式

| 维度 | Subagent | Agent Teams |
|------|----------|-------------|
| **谁在协调** | **主 agent**：决定何时派给哪个 subagent、何时复用 session_id、如何综合多次 Task 结果。 | **Lead + 共享任务列表**：Lead 创建任务、分配或让 Teammate 认领；任务依赖（blockedBy）与状态（pending / in progress / completed）由系统与各 agent 共同维护；Teammate 可**自组织**（claim 下一个任务）。 |
| **任务抽象** | 无统一「任务列表」；主 agent 用自然语言或内部规划拆解工作，通过多次 Task 调用实现。 | **显式 Task list**：TaskCreate、TaskUpdate、TaskList、claimTask、assignTask 等；适合「多人在同一批任务上协作」的流程。 |

### 4.4 适用场景与成本

| 维度 | Subagent | Agent Teams |
|------|----------|-------------|
| **最佳用途** | 聚焦型子任务、只需「结果」回主会话：探索代码库、计划前调研、跑测试/大输出隔离、并行研究后汇总。 | 需要**多角色讨论、互相挑战、并行探索**：多角度代码评审、竞争假设调试、跨前端/后端/测试的并行开发、需共识或辩论的决策。 |
| **Token 成本** | **较低**：仅主会话 + 若干子会话的返回摘要进入主会话；子会话可选用更小模型（如 Haiku）。 | **较高**：每个 Teammate 都是完整实例，context 不共享；消息与任务状态多轮传递，总 token 随人数与轮次增长。 |
| **协调开销** | 主 agent 负责所有派发与汇总，逻辑简单。 | 需管理团队生命周期、任务依赖、消息类型与关闭/审批流程；广播慎用（成本高）。 |

### 4.5 小结对比表

| 问题 | Subagent | Agent Teams |
|------|----------|-------------|
| 主 agent 通过什么调用子/Teammate？ | **Task 工具**（description, prompt, subagent_type, 可选 session_id）。 | **Task 工具** 用于 **spawn Teammate**（含 name, team_name, mode）；任务通过 **TaskCreate / TaskUpdate / TaskList** 与 **assignTask / claimTask** 分配与认领。 |
| 上下文是否独立？ | **是**：每个 Task 对应（新或复用的）**子会话**，与主会话隔离。 | **是**：每个 Teammate 为**独立会话/实例**，与 Lead 及其他 Teammate 隔离。 |
| 多次调用能否继承上下文？ | **能**：通过 Task 返回的 **session_id** 在下次 Task 中传入，复用同一子会话。 | Teammate 会话天然持续；任务与消息在**共享 Task list + Mailbox** 上延续，无「同一 subagent session_id」概念。 |
| 子/Teammate 能否交互？ | **子会话内**可多轮；与主 agent 之间为**单次任务 → 单次结果**，多轮需主 agent 多次 Task + session_id。 | **能**：通过 **SendMessage** 与 Lead、Teammate 之间**直接通信**；可讨论、挑战、请求关闭或计划审批。 |
| 结果如何回主/Lead？ | **Task 工具返回值**（output + session_id）写回主会话，主 agent 下一轮可见。 | **SendMessage** 内容 + **Task 状态更新**（TaskUpdate 等）；Lead 不自动获得 Teammate 的完整执行过程。 |
| 是否多会话？ | **否**：单主会话 + 多个子会话（同进程/会话树）。 | **是**：多会话（Lead + 多个 Teammate 实例）。 |

---

## 5. 与 OpenCode / Oh-My-OpenCode 的对照（简要）

- **OpenCode**：主 agent 通过 **Task 工具** 调用 subagent，子会话独立、可 session_id 延续、结果通过 Task 返回值回主；无「多会话 + 共享任务列表 + 显式消息」的 Agent Teams 形态。  
- **Oh-My-OpenCode**：在 OpenCode 上通过插件增加 **delegate_task**、**call_omo_agent**、**background_task** 等，仍是「单会话树 + 子会话 + 工具返回值/background_output」；与 Claude Code 的 **Agent Teams**（多实例 + SendMessage + Task list）不同。  
- **Claude Code**：同时提供 **Subagent**（单会话内、Task 驱动、结果回主）和 **Agent Teams**（多会话、任务列表 + Mailbox、Teammate 间可通信），两种机制并存、按场景选择。

---

## 6. 小结表（Claude Code 单列）

| 问题 | Subagent | Agent Teams |
|------|----------|-------------|
| 主/Lead 通过什么调用？ | Task 工具（subagent_type, prompt, session_id…） | Task 用于 spawn Teammate；TaskCreate/Update/List + assign/claim 管理任务。 |
| 上下文是否独立？ | 是（子会话） | 是（独立实例/会话） |
| 能否继承/延续上下文？ | 能（session_id） | 通过任务列表与消息延续协作，无 session_id 复用。 |
| 子/Teammate 间能否交互？ | 否（仅回主 agent） | 是（SendMessage：私信、广播、关闭、计划审批） |
| 结果如何回主/Lead？ | Task 返回值 | SendMessage + 任务状态 |
| 典型场景 | 探索、调研、隔离大输出、并行研究汇总 | 多角度评审、竞争假设、跨层协作、需讨论与共识 |

---

## 7. 参考

- [Claude Code — Create custom subagents](https://code.claude.com/docs/en/sub-agents)  
- [Claude Code — Run agent teams](https://code.claude.com/docs/en/agent-teams)  
- [Claude Code Agent Teams 机制研究](https://www.cnblogs.com/pDJJq/p/19592455/research-on-claude-code-agent-teams-mechanism-17xegk)（pDJJq，博客园）  

*分析基于上述文档与社区文章整理；若与最新版本有差异，请以官方文档为准。*
