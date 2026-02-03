## OpenCode 扩展机制深度报告（以 Oh-My-OpenCode 为案例）

> **目标**：用 `oh-my-opencode`（本仓库）作为“真实重度插件”样本，反向验证 OpenCode 的插件/Hook 扩展机制如何实现、可扩展性边界在哪里、是否足以支撑二次开发（自建模型服务、MCP/skills 市场、自定义上下文机制等）。
>
> **对照基准**：
> - OpenCode 插件文档：`https://opencode.ai/docs/plugins/`
> - OpenCode 源码（本地）：`/Users/cc/yre/opencode-dev`
> - Oh-My-OpenCode 源码（本项目）：`/Users/cc/yre/oh-my-opencode-dev`

---

## 1. 结论摘要（先给结论）

- **OpenCode 的插件机制是“固定 hook 名称 + 全局事件总线(event) + 可注入 tool”的组合**：插件能在多个关键点拦截/改写输入输出，并且能添加大量新工具，足以做出 `oh-my-opencode` 这种“重度增强插件”。
- **但插件不能“新增新的 hook 埋点/生命周期”**：hook 名称是 SDK `Hooks` 接口中固定字段，核心只在若干处硬编码调用 `Plugin.trigger("<hook-name>")`。要扩展新的埋点，需要改 OpenCode core（新增 trigger 调用点 + 扩展 SDK 类型）。
- **插件能订阅“所有 Bus 事件”（event hook）**，因此在“反应式增强/监控/自动化”方向很灵活；但 **新增 Bus 事件类型仍属于 core 改造**。
- **自建模型服务对接**：若你的服务能伪装成 OpenAI/Anthropic 等现有 Provider 兼容协议，可能可以走配置层；若需要“新增 provider 类型/特殊 token 计费/特殊输入输出变换”，**大概率要改 Provider 层（core）**。
- **AI 市场（MCP + skills + tool盒子）**：插件非常适合做“加载器/路由器/聚合器”。`oh-my-opencode` 已经用插件把三层 MCP（内置 + Claude Code 兼容 + skill 内嵌）和技能系统打通，是一个强有力的正面样例。
- **自定义上下文机制**：OpenCode 原生提供 `experimental.chat.messages.transform`、`experimental.chat.system.transform`、`experimental.session.compacting` 三个关键扩展点；`oh-my-opencode` 大量使用（注入/裁剪/预压缩/规则注入），说明该机制在“上下文工程”上相当可用。

---

## 2. OpenCode 插件/Hook 机制：核心实现到底长什么样？

### 2.1 插件加载与初始化（OpenCode core）

**核心代码**：`/Users/cc/yre/opencode-dev/packages/opencode/src/plugin/index.ts`

- OpenCode 在运行时构造一个 `PluginInput`：
  - `client`（SDK client，可调用 `session.*`、`tui.showToast` 等）
  - `project / directory / worktree / serverUrl / $ (Bun shell)`
- 插件来源三类：
  - **Internal plugins**：core 直接 import（例如 auth 插件）
  - **npm 插件**：根据配置里的 `plugin: [...]` 列表，用 `BunProc.install(pkg, version)` 安装并 `import()` 载入
  - **file:// 插件**：以本地路径 `import()` 载入
- 每个插件模块可能同时导出 default + named export；OpenCode 会做去重（同一个函数引用只初始化一次）。

### 2.2 Hook 触发机制（固定 hook 名称 + 顺序执行）

**核心代码**：`/Users/cc/yre/opencode-dev/packages/opencode/src/plugin/index.ts`

- OpenCode 提供统一入口 `Plugin.trigger(name, input, output)`：
  - 依次遍历所有已加载插件 hooks
  - 如果某个插件实现了该 hook 字段（例如 `"tool.execute.before"`），就 `await` 执行
  - **顺序是“插件加载顺序”**，且是串行执行（一个 hook 卡住会卡住整体）

### 2.3 hook 名称列表来自哪里？（SDK 定义固定字段）

**核心代码**：`/Users/cc/yre/opencode-dev/packages/plugin/src/index.ts`

`Hooks` 接口明确列出所有可用 hook 名称（节选）：
- `event`
- `tool`
- `config`
- `"chat.message"` / `"chat.params"` / `"chat.headers"`
- `"tool.execute.before"` / `"tool.execute.after"`
- `"experimental.chat.messages.transform"` / `"experimental.chat.system.transform"`
- `"experimental.session.compacting"`
- `"permission.ask"`
- `"command.execute.before"`
- `"experimental.text.complete"`

**含义**：插件不能定义一个新的 hook 名称（比如 `"session.foo.bar"`）并期待 core 会触发它；除非你去改 core 的调用点。

### 2.4 OpenCode 在哪些地方触发 hook？（硬编码调用点）

**最关键的触发点**（决定“扩展边界”）：

- **消息发送/入库**：`/Users/cc/yre/opencode-dev/packages/opencode/src/session/prompt.ts`
  - 创建 user message 与 parts 之后触发：
    - `Plugin.trigger("chat.message", input, {message, parts})`
  - LLM 前对 messages 做最终变换：
    - `Plugin.trigger("experimental.chat.messages.transform", {}, { messages })`
- **工具执行前后**：`/Users/cc/yre/opencode-dev/packages/opencode/src/session/prompt.ts`
  - 在工具执行前：
    - `Plugin.trigger("tool.execute.before", {tool, sessionID, callID}, { args })`
  - 在工具执行后：
    - `Plugin.trigger("tool.execute.after", {tool, sessionID, callID}, result)`
  - 该逻辑适用于 ToolRegistry tools，也适用于 MCP tools（OpenCode 会包一层 wrapper 统一触发 hook）。
- **会话压缩（compaction）**：`/Users/cc/yre/opencode-dev/packages/opencode/src/session/compaction.ts`
  - compaction 前触发：
    - `Plugin.trigger("experimental.session.compacting", {sessionID}, {context:[], prompt?:string})`
  - 插件可以：
    - 往 `context[]` 里追加（拼到默认 prompt 后面）
    - 或直接设置 `prompt`（完全替换默认 compaction prompt）

### 2.5 插件 event hook：订阅所有总线事件（Bus）

**核心代码**：`/Users/cc/yre/opencode-dev/packages/opencode/src/plugin/index.ts`

OpenCode 会订阅 Bus 的所有事件，并对每个插件执行：

- `hook["event"]?.({ event })`

**含义**：只要 core 在 Bus 上发布了事件（如 `session.created`、`session.idle`、`message.updated`…），插件都能观察并做出反应。

但注意：
- 插件能“观察/响应”事件
- 插件不能“扩展 core 的事件类型定义”本身；要新增一个核心事件，还是需要改 core 在合适的位置 `Bus.publish(...)`

### 2.6 Tool 注入机制（插件 tool = 直接注入 ToolRegistry）

**核心代码**：`/Users/cc/yre/opencode-dev/packages/opencode/src/tool/registry.ts`

ToolRegistry 会聚合三类 tool：

1) OpenCode 内置工具（bash/read/write/edit/task/todo/skill 等）
2) `Config.directories()` 下 `{tool,tools}/*.{js,ts}` 的本地脚本工具（按文件名/导出拼出 tool id）
3) **插件注入工具**：遍历每个 plugin 的 `plugin.tool` 字段，把 `ToolDefinition` 转成 Tool.Info

**含义**：插件的“新增功能”最强路径之一就是：**新增工具**。在 `oh-my-opencode` 中，自定义工具是主战场之一（delegate_task、skill、skill_mcp、slashcommand、interactive_bash、background_*、look_at 等）。

---

## 3. Oh-My-OpenCode 是如何“利用 hook 能力”实现新功能的？

### 3.1 入口总览：它把 OpenCode 当成一个可编排平台

**入口文件**：`/Users/cc/yre/oh-my-opencode-dev/src/index.ts`

`OhMyOpenCodePlugin` 返回的 hooks 对象同时实现了：

- `tool: { ... }`：注入 20+ 自定义工具（以及一些内置 wrapper）
- `"chat.message"`：用户消息发出时的拦截与改写（关键词检测、Claude Code 兼容、auto slash command、start-work 等）
- `"tool.execute.before"`：在工具执行前改写参数/附加约束（如禁用子代理问问题、截断 label、注入规则、Prometheus 只读、Atlas 编排约束等）
- `"tool.execute.after"`：工具执行后增强输出（截断输出、错误恢复、提示、编排回收）
- `event`：监听 session/message/tool 等总线事件，驱动后台任务与持续工作流
- `"experimental.chat.messages.transform"`：对即将送入模型的 messages 做变换（上下文注入、thinking 块校验等）
- `"experimental.session.compacting"`：压缩前注入自定义上下文（保持工作状态/计划）

> 注意：该文件末尾明确写了警告：**不要从 `src/index.ts` export 普通函数**，否则 OpenCode 会把它们当插件实例调用。

### 3.2 34 个 hooks：从“埋点”到“系统能力”的映射

Hook 矩阵（精简版，完整矩阵建议直接看 `src/hooks/AGENTS.md` + 本报告附录表）：

- **编排/工作流层（Orchestration）**
  - `atlas`：强制“编排者不直接改代码”，把实现工作推给子代理；对 `delegate_task` 注入“单任务”系统指令；在任务完成后注入“必须验证”的提醒，并附加 git diff 摘要。
  - `todo-continuation-enforcer`：在 `session.idle` 时检查待办，自动推动继续（避免半途停工）。
  - `ralph-loop`：自迭代循环（带 completion promise），驱动连续多轮执行。

- **上下文工程层（Context Engineering）**
  - `tool-output-truncator`：截断工具输出，避免上下文爆炸。
  - `preemptive-compaction`：到阈值前预压缩。
  - `compaction-context-injector`：压缩时注入结构化上下文，提升“续写质量”。
  - `rules-injector` / `directory-agents-injector` / `directory-readme-injector`：按条件把规则/说明注入到对话上下文。
  - `thinking-block-validator`：在 `experimental.chat.messages.transform` 验证/修复 thinking 块结构，避免 provider/模型因格式问题报错。

- **鲁棒性与自愈层（Resilience）**
  - `session-recovery`：针对常见崩溃类型做自动恢复。
  - `anthropic-context-window-limit-recovery`：针对 Anthropic 上下文限制错误做恢复。
  - `edit-error-recovery`、`delegate-task-retry`：工具失败后注入可操作的恢复路径。
  - `unstable-agent-babysitter`：监控不稳定代理/后台任务的超时与异常。

- **交互体验与“习惯塑形”层（UX + Guardrails）**
  - `keyword-detector`：切换 ultrawork/search/analyze 等模式并注入相应提示。
  - `agent-usage-reminder` / `category-skill-reminder`：提醒使用合适的专用能力（agent/skill）。
  - `non-interactive-env` / `interactive-bash-session`：适配非交互环境与 tmux 交互式 bash。
  - `question-label-truncator`、`subagent-question-blocker`：减少 UI 噪音与“子代理反问人类”的失控行为。
  - `stop-continuation-guard`：/stop-continuation 之后全链路停工（包含 todo 强推、loop、boulder 状态等）。

### 3.3 Oh-My-OpenCode “新增了哪些能力模块”（不仅是 hooks）

这部分可以理解为：**插件把 OpenCode 的扩展点“堆叠成一套产品级系统”**。

#### A) Agents（多模型、多角色分工）

位置：`/Users/cc/yre/oh-my-opencode-dev/src/agents/`

新增/强化了 11 个 agent（Sisyphus/Atlas/Prometheus/Hephaestus/Oracle/Librarian/Explore/Multimodal-Looker/Metis/Momus/Sisyphus-Junior），并配套：
- fallback 链
- tool restriction（例如 librarian/explore 默认禁止 write/edit/task/delegate_task）
- 专门的“编排哲学”（见 `docs/guide/understanding-orchestration-system.md`）

> 这不是改 OpenCode core，而是借助 OpenCode 自带的“agent 概念/选择机制”，在插件侧建立团队分工。

#### B) Tools（工具盒：delegate_task / skill / mcp / lsp / ast-grep 等）

位置：`/Users/cc/yre/oh-my-opencode-dev/src/tools/`

核心新增：
- `delegate_task`：任务委派路由器（按 category/agent 选择模型、并行、后台、重试）
- `skill` / `skill_mcp`：技能执行 + 技能携带 MCP 的生命周期管理
- `slashcommand`：把命令模板（builtin commands + 自定义命令）变成工具
- `interactive_bash`：tmux 交互式 bash
- `background_output` / `background_cancel`：后台任务观测/控制
- `look_at`：多模态文件/图片解析工具
- LSP/AST-grep/grep/glob 等工具增强（更系统化）

在 OpenCode 层面，这些都通过 `plugin.tool` 注入 `ToolRegistry`，从而出现在模型可调用工具集合中。

#### C) Skills + MCP（三层技能与 MCP 系统）

位置：
- skills：`/Users/cc/yre/oh-my-opencode-dev/src/features/builtin-skills/` + loader
- MCP：`/Users/cc/yre/oh-my-opencode-dev/src/mcp/`

`oh-my-opencode` 把 “技能（prompt 模板/约束）” 与 “MCP（外部工具连接）”绑定，形成三层结构：

1) **内置 MCP**：`websearch/context7/grep_app`
2) **Claude Code 兼容 MCP**：读取 `.mcp.json` 并做 `${VAR}` 展开
3) **skill 内嵌 MCP**：某个 skill 自带 MCP 配置，并由 `skill-mcp-manager` 在会话生命周期里懒加载/回收

这相当于在插件侧实现了一个“AI 市场/工具盒”雏形：技能描述“怎么做”，MCP 提供“能做什么”，再由 `delegate_task`/分类系统决定“谁来做”。

#### D) Background Agent + Tmux（并行化与可观察性）

位置：`/Users/cc/yre/oh-my-opencode-dev/src/features/background-agent/`、`tmux-subagent/`

通过监听 OpenCode 的 session 事件并用插件工具启动/轮询子会话，形成：
- 并发执行（按 provider/model 限流）
- 任务状态机（launch → poll → complete）
- 失败恢复与通知
- tmux 可视化会话管理

---

## 4. 这个插件“重构了原版的什么部分”？

这里的“重构”不是指修改 OpenCode core 代码，而是指：**在不改 core 的情况下，通过 hook 与工具注入，把 OpenCode 的默认“单代理对话式开发”重构成“编排系统 + 多代理团队 + 可验证工作流”**。

可以把它理解为三类“对原版行为的重写”：

### 4.1 重写“工作模式”（从单回合对话 → 连续 boulder 推进）

OpenCode 原版：用户一句话 → agent 产出 → 结束  
Oh-My-OpenCode：通过 `session.idle`、`session.error`、todo/loop 等事件，把会话变成可持续推进的流水线（直到任务完成或显式 stop）。

### 4.2 重写“工具使用方式”（从自由工具调用 → 受约束的编排协议）

典型例子：`atlas` hook 在 `tool.execute.before/after` 强制编排者不直接写代码，必须 delegate；并在 delegate 返回后强制“你必须自己验证”。

### 4.3 重写“上下文管理策略”（从被动 compaction → 主动裁剪/注入/预压缩）

OpenCode 原版主要依赖自动 compaction；Oh-My-OpenCode 通过：
- 截断工具输出（减少污染）
- 预压缩（提前做）
- compaction 时注入结构化上下文（提高续写质量）
- rules/README/AGENTS 的条件注入（提高领域对齐）

这本质是在 plugin 层把“提示词工程 + 上下文工程”产品化了。

---

## 5. OpenCode 的扩展机制到底有多灵活？（以 Oh-My-OpenCode 验证）

### 5.1 强项：能做“很像重写产品”的增强

从 `oh-my-opencode` 可以确认：只靠插件，就能实现：
- 新工具体系（tool 注入）
- 新的工作流编排（通过 event + tool.before/after + chat.message）
- 较深的上下文工程（messages transform + compaction hook）
- 与外部生态兼容（Claude Code hooks、skill loader、mcp loader）

结论：**对“工具/流程/上下文”三件事，插件机制非常强。**

### 5.2 边界：不能新增 hook 埋点（只能用既有 trigger 点）

证据链：
- hook 名称固定：`/Users/cc/yre/opencode-dev/packages/plugin/src/index.ts` 的 `Hooks` 接口字段
- hook 触发点固定：core 里硬编码 `Plugin.trigger("...")` 的调用点（`session/prompt.ts`、`session/compaction.ts` 等）

因此：
- `oh-my-opencode` **没有“扩展 OpenCode 新埋点”**（它只是实现了既有 hook 字段）
- 它能做到这么多，是因为 OpenCode 预留的埋点已经足够覆盖“消息/工具/压缩/系统提示/权限/命令/事件总线”等关键点

### 5.3 插件是否“自己扩展了新埋点”？——在插件内部可以，但不会变成 OpenCode 的新 hook

`oh-my-opencode` 在插件内部确实扩展了自己的“内部事件/状态机”（例如 background task 生命周期、boulder-state、tmux session 状态等），但这属于插件自身实现，不会让 OpenCode 的其它插件也获得一个新 hook 名称。

---

## 6. 面向你的二次开发目标：三类需求逐项评估

### 6.1 对接自建模型服务（Provider/模型层）

**插件能做的：**
- 如果你的模型服务能兼容 OpenAI/Anthropic 等现有 provider 协议（或通过 OpenCode 已有 provider 接入方式配置），可能无需 fork。
- 插件可以做 auth、请求前后改写（有限，主要在 chat.params/chat.headers/system transform 等范围内）。

**插件做不到/通常需要改 core 的：**
- 新增一个 Provider 类型（注册、鉴权、token 预算、tool schema 变换、流式协议适配）
  - 参考：`/Users/cc/yre/opencode-dev/packages/opencode/src/provider/provider.ts`（provider 管理）

**建议**：
- 最优路径：把内部模型服务做成“兼容现有 provider 的网关”（减少 fork 成本）
- 次优路径：fork core 的 Provider 层，并尽量保持改动最小、可上游化

### 6.2 AI 市场（MCP 工具盒 + skills）

**插件非常适合**：
- tool 注入可以做“市场入口工具”（例如 `skill`、`skill_mcp`、`slashcommand`）
- MCP 本身是 OpenCode 支持的一等公民，core 还会把 MCP tools 纳入工具执行路径，并同样触发 plugin hooks
- `oh-my-opencode` 的三层 MCP 系统可以作为你“内部 AI 市场”的直接参考

**建议**：
- 你可以在插件侧实现：market registry（skills metadata）+ mcp lifecycle manager + 权限/配额策略 + 路由器（delegate_task 类）
- 若要做“市场 UI/搜索/推荐”，更多是客户端/上层产品，不一定要动 core

### 6.3 自定义上下文机制（更强的 Context Engine）

OpenCode 给了三个非常关键的插拔点：
- `experimental.chat.messages.transform`：最后一步改写 messages（适合注入/裁剪/重排）
- `experimental.chat.system.transform`：改写系统 prompt（适合策略/安全/规范注入）
- `experimental.session.compacting`：改 compaction prompt/上下文（适合续写质量与状态保留）

`oh-my-opencode` 已经证明：在不改 core 的情况下，可以实现相当复杂的上下文工程（规则注入、输出截断、预压缩、压缩上下文注入、错误恢复等）。

**建议**：
- 如果你的“新上下文机制”主要是：选择性注入、去重、压缩、持久化、结构化记忆 → 先用插件实现
- 如果你的机制需要：改变 core 的消息存储结构、token 计量策略、跨会话检索索引（强依赖底层数据结构）→ 可能需要 fork core 的 session/message 层

---

## 7. 你接下来如何基于 OpenCode 做二次开发（建议路线图）

### 7.1 “先插件后 fork”的优先策略

1) 先用插件做：
   - 自定义工具/技能市场入口
   - 工作流编排（类似 atlas + background manager）
   - 上下文工程（transform + compaction）
2) 只在确认“确实缺埋点/缺 provider”时再 fork core：
   - 新增 hook 埋点：在 core 的关键流程加 `Plugin.trigger("new-hook")`
   - 新增 Provider：扩展 `provider` 层

### 7.2 如果你真的要“新增埋点/Hook”

你需要改动 OpenCode core（最小改动原则）：
- `packages/plugin/src/index.ts`：扩展 `Hooks` 接口（新增字段）
- `packages/opencode/src/...`：在合适流程中插入 `Plugin.trigger("你的hook", input, output)`
- （可选）补充文档与测试，确保插件开发者可用

---

## 附录 A：Oh-My-OpenCode Hook 矩阵（完整版建议在源码中生成/同步）

> 该矩阵已在 `src/hooks/AGENTS.md` 维护；本报告生成时也做了全量盘点。你如需“自动同步到表格/导出 CSV”，可以基于 hooks 目录做一个小脚本从 AST 提取订阅的 hook 字段。

---

## 参考链接（外部）

- OpenCode 插件文档：`https://opencode.ai/docs/plugins/`
- OpenCode 文档首页：`https://opencode.ai/docs/`
- OpenCode GitHub：`https://github.com/anomalyco/opencode`

