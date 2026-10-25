> 译注：本文译自[英文原文](extensions.md)，中文译文可能滞后；代码示例保留原样。

<a id="extensions"></a>

# 扩展

扩展是为 Pi 添加可执行行为的 TypeScript 模块。当工作流需要工具、命令、事件处理器、模型 Provider、会话状态或终端 UI，而不只是指令时，就应使用扩展。

扩展在 Pi 进程内部运行，拥有相同的操作系统权限。它可以检查提示词、工具调用、文件、凭据和会话历史，因此只应加载来自可信来源的扩展。

典型扩展会添加 agent 工具、保护路径、确认危险命令、响应会话事件、修改上下文、提供命令或显示持久状态。

<a id="quick-start"></a>
<a id="writing-an-extension"></a>
<a id="create-an-extension"></a>

<a id="create-and-load-an-extension"></a>

## 创建并加载扩展

扩展导出一个接收 `ExtensionAPI` 的默认工厂函数。该工厂函数为当前扩展运行时注册能力。

创建 `~/.pi/agent/extensions/hello.ts`：

```typescript
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";

export default function (pi: ExtensionAPI) {
  pi.registerCommand("hello", {
    description: "Show a greeting",
    handler: async (name, ctx) => {
      ctx.ui.notify(`Hello, ${name || "world"}!`, "info");
    },
  });
}
```

启动 Pi 并运行 `/hello`。开发期间，可直接加载文件：

```bash
pi --extension ./hello.ts
```

Pi 使用 `jiti`，因此本地 TypeScript 扩展不需要单独的编译步骤。发布扩展和依赖时，请使用 [Pi 包](packages.md)。

<a id="extension-locations"></a>
<a id="available-imports"></a>
<a id="choose-where-it-loads"></a>

<a id="add-it-to-pi"></a>

## 将扩展加入 Pi

将扩展放入用户或项目的扩展目录。Pi 会加载目录中直接存放的 TypeScript 或 JavaScript 文件，以及包含 `index.ts` 或 `index.js` 入口点的子目录。

小型扩展使用单个文件，多文件实现使用目录。将 npm 依赖放入附近的 `package.json`。常规位置请参阅[配置](configuration.md)，额外路径请参阅[设置](settings.md#resources)。

重新加载会替换扩展运行时，因此 `await ctx.reload()` 之后的代码不得复用旧运行时的状态。只有个人扩展和显式通过命令行加载的扩展，才能参与在项目扩展加载之前运行的 `project_trust` 事件。

<a id="understand-the-lifecycle"></a>

<a id="respect-the-runtime-lifecycle"></a>

## 遵循运行时生命周期

工厂函数可以是同步或异步的。Pi 会等待异步工厂函数完成后再继续启动，因此它可以获取配置或注册启动期间需要的 Provider。

不要在工厂函数中启动进程、socket、watcher 或定时器，因为某些调用会加载扩展但不启动会话。
应在 `session_start` 中，或在需要它们的命令或工具中启动长期运行的资源。
通过幂等的 `session_shutdown` 处理器关闭会话作用域内的资源。

一次运行从输入和 `before_agent_start` 开始，经过模型、消息和工具事件，直到 `agent_end`。
运行启动期间，包括压缩检查和 `before_agent_start`，Pi 都视为忙碌，因此在此期间发送的消息会进入正在启动的运行。
之后仍可能继续自动重试、恢复、压缩或处理排队工作。
<a id="agent_start--agent_end--agent_before_settle--agent_settled"></a>

`agent_before_settle` 是最后一个可执行操作的边界：它可以追加条目并请求续跑一次。
`agent_settled` 是最终事件，仅用于通知；当集成需要知道 Pi 不会再自动继续时，使用它。

<a id="extensionapi-methods"></a>

<a id="choose-an-integration-point"></a>

## 选择集成点

| 能力 | 主要 API |
|---|---|
| 观察或修改生命周期行为 | `pi.on()` |
| 添加可由模型调用的操作 | `pi.registerTool()` |
| 添加 `/` 命令 | `pi.registerCommand()` |
| 添加快捷键或 CLI 标志 | `pi.registerShortcut()` 或 `pi.registerFlag()` |
| 发送 user 消息或自定义消息 | `pi.sendUserMessage()` 或 `pi.sendMessage()` |
| 持久化不属于上下文的会话数据 | `pi.appendEntry()` |
| 更改活动工具、模型或思考级别 | `pi` 上的会话控制方法 |
| 添加模型 Provider | `pi.registerProvider()` |
| 添加 MCP 服务器 | `pi.registerMcpServer()` |
| 为每个请求路由到一个模型 | [`pi.registerVirtualModel()`](virtual-models.md) |
| 添加终端渲染 | 渲染器注册和 `ctx.ui` |
| 与另一个扩展通信 | `pi.events` |

事件、上下文、工具和结果的确切类型，请查阅 [`extensions/types.ts`](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/extensions/types.ts) 中导出的声明。

<a id="follow-the-extension-contracts"></a>

## 遵守扩展契约

<a id="events"></a>
<a id="work-with-events"></a>

<a id="events-and-concurrency"></a>

### 事件与并发

处理器按照扩展加载顺序和注册顺序运行。`pi.on()` 返回一个用于取消该次注册订阅的函数；变更不会影响已经开始的事件分发。
有些事件用于通知；另一些会转换数据、替换结果或取消操作。
应使用各事件声明的结果类型，而不是假定所有返回值都会产生效果。

事件涵盖资源发现、会话、agent 和消息生命周期、Provider、工具以及原始输入。

`before_agent_start` 同时提供当前提示词及其结构化的 `systemPromptOptions`。`event.messageType` 表示是什么启动了本次运行：对于提示词或 `pi.sendUserMessage()`，其值为 `"user"`，并提供 `prompt` 和 `images`；对于 `pi.sendMessage(..., { triggerTurn: true })`，其值为 `"custom"`，并提供启动本次运行的 `message`，而没有 `prompt`。返回的 `message` 会被加入任一种运行；要跳过由自定义消息启动的运行，请检查 `messageType`。优先修改提示词分节、所选工具或指导规则，以便 Pi 追加对话记录增量。返回 `systemPrompt` 或设置 `forceSystemPrompt` 会替换本次运行的整个提示词，但对话记录仍继续记录结构化分节。Provider 收到的开头 system prompt 是被强制指定的文本。

`message_end` 可以在保留角色的同时替换已定稿的消息。`tool_call` 可以修改输入或阻止执行。`tool_result` 处理器会依次组合，每个处理器都能看到之前的变更。

<a id="provider_stream_event"></a>

`provider_stream_event` 在 Pi 规范化每个已解析的 Provider 流事件之前触发。该事件标识 Provider、API 和模型；`event.data` 是 Pi 能获取到的最早结构化值，不一定是原始 HTTP 字节或 SSE 帧。应将其视为只读，因为修改可能影响规范化。此事件仅用于通知，不会持久化。

处理器按流顺序被等待执行，因此缓慢的处理器会延迟流的消费。处理器错误会被报告，但不会改变 Provider 响应。[`debug-provider.ts`](../examples/extensions/debug-provider.ts) 提供了一个可选择启用的查看器，按 assistant 消息对原始事件分组。

<a id="context_with_system"></a>

`context` 转换对话消息，不包含承载提示词和工具的 system 消息；Pi 会在之后恢复该状态。仅当请求局部的转换必须掌控完整对话记录时，才使用 `context_with_system`，并保持索引零处为一条 system 消息。

`turn_end` 和 `agent_before_settle` 是可执行操作的边界。其处理器可以链式衔接拟追加的 `custom`、`custom_message`、`context_edit` 或 `compaction` 条目，并返回 `continue: true` 以再发起一次模型请求。请为继续运行设置条件保护，因为无条件继续可能导致循环。完整的验证和顺序契约请查阅导出的事件声明。

<a id="cache_warming_decision"></a>

`cache_warming_decision` 可以通过 `{ action: "warm" }` 或 `{ action: "stop" }` 覆盖空闲时的提示词缓存刷新。以最后一个返回 action 的处理器为准。

同一条 assistant 消息中的工具调用可以并行运行。
当另一个工具事件运行时，不要假定同批的另一个调用或结果已经存在。
对于由活动轮次拥有的嵌套工作，请使用 `ctx.signal`；命令和空闲会话事件通常没有操作信号。

返回 `undefined` 的 `user_bash` 处理器会将命令传递给下一个处理器；如果没有处理器处理它，则交由本地执行。返回 `operations` 或 `result` 会停止传播。处理器失败会阻止命令，而不是回退到本地执行。

<a id="custom-tools"></a>
<a id="register-tools"></a>

<a id="tools"></a>

### 工具

自定义工具定义名称、面向模型的说明、TypeBox 参数 schema 和 `execute()` 函数。
其结果必须包含面向模型的 `content`，以及用于渲染或状态重建的 `details` 字段。
没有结构化详情时，使用 `details: undefined`。如果工具发起嵌套模型调用，请在结果中包含这些调用的 `usage`，以保持会话总计准确。

在 `execute()` 中抛出异常会产生失败的工具结果。
返回对象不会将其标记为错误。
仅当 agent 应在该批次中每个已完成工具都同意终止后跳过自动后续处理时，才返回 `terminate: true`。

工具共享可变的内存状态时，应顺序执行。
修改文件的工具应使用 `withFileMutationQueue()` 包裹完整的读取—修改—写入操作。
截断面向模型的大型结果，并告知模型在哪里读取完整输出。

当结果为数据时，声明 `outputSchema` 并返回匹配的 `structuredContent`。模型仍会收到 `content`；codemode 脚本等程序化调用方收到的则是 `structuredContent`，而不是文本。没有 `outputSchema` 的工具会将其文本内容传给脚本。要报告仍携带数据的失败，请返回带有 `isError: true` 的结果，而不是抛出异常：模型会看到错误，脚本仍会收到 `structuredContent`。

工具可以通过 `ctx.executeTool(name, args, { signal, onUpdate })` 运行其他工具。嵌套调用与模型发起的调用一样，会经过参数验证以及 `tool_call` 和 `tool_result` 处理器，并发出 `tool_execution_start`、`tool_execution_update` 和 `tool_execution_end`；所有这些事件都携带 `parentToolCallId`，其 `toolCallId` 由 pi 按 `<parent id>/<n>` 分配。这些 ID 不会作为工具调用或工具结果出现在对话记录中。嵌套调用不会添加对话记录条目：其结果只传给调用它的工具，由该工具自行报告，例如通过 `onUpdate` 和 `details` 报告。会话在调用工具的结果消息上，以 `nestedCalls` 保存这些调用的有界记录（名称、参数、状态、耗时、错误；从不保存结果）。该记录用于压缩时的文件列表，并会显示在 HTML 导出中。单次调用超过 8 KiB 或每个工具结果合计超过 32 KiB 的参数会被省略，最多保留 256 次调用，而 `complete: false` 表示记录中丢失了部分信息。各个嵌套深度的结果 `usage` 都会计入调用工具的结果 `usage`，因此工具只报告自身用量，而不报告其所调用工具的用量。`ctx.tools` 列出 `ctx.executeTool()` 可以调用的工具。对 `content` 进行脱敏的 `tool_result` 处理器也应替换 `structuredContent`；仅替换 `content` 会使其被丢弃。

请参阅 [`hello.ts`](../examples/extensions/hello.ts)、[`todo.ts`](../examples/extensions/todo.ts)、[`dynamic-tools.ts`](../examples/extensions/dynamic-tools.ts) 和 [`truncated-tool.ts`](../examples/extensions/truncated-tool.ts)。

<a id="tool-exposure"></a>

### 工具暴露方式

`exposure` 控制模型如何访问工具。“可调用”指可以由其他工具通过 `ctx.executeTool()`（`ctx.tools`）调用，`codemode` 工具的脚本就是这样做的：

- `direct`（默认）：活动时向模型声明，且活动时可调用。
- `model-only`：活动时向模型声明，但永远不可调用。用于编排其他工具或询问用户的工具。
- `codemode`：只要已注册就可调用，并由 `codemode` 工具列出。除非显式激活，否则不向模型声明。
- `deferred`：类似 `codemode`，但 codemode 工具不会列出它；`tool_search` 可以找到并激活它。
- `hidden`：已注册但不可访问。由于工具无法取消注册，要撤回工具，可用 `exposure: "hidden"` 重新注册它。

`namespace: { name, aliases, description, instructions }` 将相关工具分组，MCP 服务器也是如此。Codemode 工具会在一个标题下列出某个 namespace，并附上其 `description`。`instructions` 保存较长的使用指导；它不会被列出，codemode 脚本通过 `describeNamespace(name)` 读取它。`aliases` 是脚本可用于查找 namespace 的其他名称，例如 `mcp__docs` 对应的服务器名称 `docs`；名称本身及其脚本标识符形式（`my-docs`、`my_docs`）都可以匹配。

注册 `direct` 或 `model-only` 工具会激活它；其他 exposure 的工具不会在注册时激活。活动集合（`pi.getActiveTools()`、`pi.setActiveTools()`）就是向模型声明的工具集合。`pi.getAllTools()` 报告每个工具的 `exposure`、`namespace`、`annotations` 和 `reachesTools`。

`annotations` 是关于工具行为的提示，其含义与 MCP 工具注解相同：`readOnlyHint`、`destructiveHint`、`idempotentHint` 和 `openWorldHint`。MCP 工具携带服务器声明的提示。缺失的提示采用 MCP 默认值：工具不是只读的，可能具有破坏性，也可能访问开放世界。这些提示未经验证，但权限扩展可以据此决定哪些调用需要确认。以下代码会对 Codex 需要请求批准的调用进行确认：

```typescript
pi.on("tool_call", async (event, ctx) => {
  const hints = pi.getAllTools().find((tool) => tool.name === event.toolName)?.annotations;
  const needsApproval =
    hints?.destructiveHint === true ||
    (!hints?.readOnlyHint && ((hints?.destructiveHint ?? true) || (hints?.openWorldHint ?? true)));
  if (needsApproval && !(await ctx.ui.confirm("Allow tool call?", event.toolName))) {
    return { block: true, reason: `${event.toolName} was not approved` };
  }
});
```

编排其他工具的工具可以通过 `prepareLoadout(loadout)` 调整其活动期间模型所见的内容。活动工具发生变化时，该方法就会运行，并接收已声明的工具、可调用的工具，以及每个已注册工具的 exposure、namespace 和提示词指导规则。它返回已声明工具（包括自身）的替代 `descriptions`，以及 `hiddenDeclarations`：这些活动工具的声明会从请求中省略，但工具仍保持活动且可调用。默认 system prompt 会从工具列表和规则中省略隐藏工具；文件读取工具被隐藏时，skills 提示也不会点名任何工具，因此编排工具应自行呈现这些工具的指导规则。`codemode` 仅使用此 hook、`exposure`、`reachesTools` 和 `ctx.executeTool()`，因此其他工具可以在不同名称下实现相同行为。

会访问其他工具的工具通过 `reachesTools(input)` 声明一次调用可能访问哪些工具：返回 `"all"`，或返回输入中可能指代它们的词，其中每个词指代所有名称包含于该词中的工具或 namespace。对于搜索或列出工具的脚本，`codemode` 返回 `"all"`；否则返回脚本中的词；`tool_search` 始终返回 `"all"`。延迟注册工具的扩展会在 `tool_call` 处理器中使用它，以在调用运行前完成所需工具的注册，MCP 扩展对仍在连接的服务器就是这样处理的。MCP 扩展也仅在同名工具声明了 `reachesTools` 时，才会为 MCP 工具激活 `codemode` 或 `tool_search`。

<a id="activate-tools-dynamically"></a>

### 动态激活工具

先注册所有工具，使可选工具保持非活动状态，再从加载器工具中使用 `pi.setActiveTools()` 选择所需的活动工具。名称必须已经注册；未知名称会被忽略。

Pi 会在对话记录的首条 system 消息中记录初始提示词和工具集合，然后在下一次模型请求之前追加工具和提示词变更。无法表示该转换的 Provider 会收到完整的对话记录检查点，这可能使缓存前缀失效。

<a id="tool-rendering"></a>

### 工具渲染

工具的 `renderCall` 和 `renderResult` 在交互式对话记录和 HTML 导出中绘制其调用。`pi.registerToolRenderer((toolName, next) => renderers)` 为任意工具的调用选择渲染器，包括尚未注册的工具，例如恢复会话时服务器尚未连接的 MCP 工具。`next()` 返回其余解析器（按扩展加载顺序），再到已注册工具原本会使用的渲染器，因此 `next() ?? mine` 只会在缺失时补充。

<a id="mcp-servers"></a>

### MCP 服务器

`pi.registerMcpServer(name, config)` 为当前会话添加 MCP 服务器。`config` 的结构与 [`mcp.json`](mcp.zh.md) 中的 `mcpServers` 条目相同：stdio 服务器使用 `command`、`args`、`env` 和 `cwd`，HTTP 服务器使用 `url`、`headers` 和 `oauth`，此外还有 `exposure`、`toolExposure`、`description`、`enabled` 和 `timeout`。

```typescript
pi.registerMcpServer("jira", { url: "https://mcp.example.com/jira", exposure: "codemode" });
pi.unregisterMcpServer("jira");
```

扩展加载期间注册的服务器会在会话启动时与 `mcp.json` 中的服务器一起连接；之后注册的服务器会立即连接，而 `pi.unregisterMcpServer()` 会关闭连接并使该服务器的工具不可访问。注册信息不会保存：每次加载时都需要重新注册，例如根据扩展自身的设置注册。`mcp.json` 中的同名服务器优先，`/mcp` 会显示这种覆盖。再次注册相同名称会替换该扩展先前的注册；如果名称由另一个扩展注册、名称无效或配置无效，则会抛出异常。

内置 MCP 支持负责连接已注册的服务器。当另一个扩展替代了它，且没有任何扩展连接这些服务器时（请参阅 [MCP](mcp.zh.md#replace-the-built-in-mcp-support)），每项注册都会作为扩展错误报告。其他 MCP 扩展也可以连接已注册的服务器：在 `session_start` 时通过 `pi.getMcpServers()` 读取，并处理 `mcp_servers_change` 事件以应对后续变更。

<a id="extensioncontext"></a>
<a id="extensioncommandcontext"></a>
<a id="use-extension-context"></a>

<a id="context-and-session-changes"></a>

### 上下文与会话变更

`ExtensionContext` 提供工作目录、模式、UI、会话管理器、模型运行时、中止信号、上下文用量，以及压缩和关闭的控制接口。
对于与 Provider 无关的嵌套模型调用，请使用 `ctx.modelRegistry.streamSimple()`。

命令处理器收到 `ExtensionCommandContext`，它额外提供等待空闲、重新加载、树导航和会话替换操作。
这些操作仅限命令使用，因为从生命周期处理器调用它们可能使运行时死锁。

会话替换会使旧上下文失效。切换之前只捕获普通数据；之后，对于绑定会话的工作，使用传给 `withSession` 的新上下文。

<a id="state-management"></a>
<a id="persist-state"></a>

<a id="state"></a>

### 状态

根据状态如何参与对话选择存储方式：

| 状态 | 存储方式 |
|---|---|
| 跟随活动分支的工具状态 | 工具结果的 `details` |
| 不进入模型上下文的持久数据 | `pi.appendEntry()` |
| 需要存储并发送给模型的自定义内容 | `pi.sendMessage()` |
| 单个会话之外的数据 | 外部存储 |

在 `session_start` 期间，通过 `ctx.sessionManager.getBranch()` 重建与分支相关的状态。
不要从文件中的每个条目重建，因为被放弃的分支代表不同的历史路径。
当自定义存储内容需要出现在对话记录中时，注册条目或消息渲染器。

<a id="custom-ui"></a>
<a id="mode-behavior"></a>
<a id="interact-with-the-user"></a>
<a id="account-for-each-mode"></a>

<a id="ui-and-modes"></a>

### UI 与模式

`ctx.ui` 提供对话框、通知、状态文本、widget、标题、编辑器访问和自定义组件。
仅在交互需要独立的渲染和输入时，才使用 `ctx.ui.custom()`。
关于组件、焦点、overlay、主题和性能的指导，请参阅[终端 UI](tui.md)。

扩展会在交互式、RPC、JSON 和 print 模式下加载。
交互式模式提供完整的终端 UI。
RPC 可以通过 [RPC 扩展 UI 协议](rpc-extension-ui.md)转发受支持的对话框和通知，但不支持自定义终端组件；JSON 和 print 模式没有 UI。
使用 `ctx.mode === "tui"` 保护仅限终端的行为，并使用 `ctx.hasUI` 判断交互式和 RPC 客户端支持的交互是否可用。

保持工具和事件行为独立于渲染，以便非交互式模式仍可正常工作。

<a id="error-handling"></a>
<a id="handle-errors-and-shutdown"></a>

<a id="errors-and-cleanup"></a>

### 错误与清理

Pi 会报告处理器错误，并在可能时继续执行。作为故障安全措施，`tool_call` 处理器失败会阻止工具执行；工具执行失败会变成提供给模型的错误结果。

即使正常操作已经尝试清理，也应在 `session_shutdown` 中释放资源。
保持清理操作幂等，因为取消、重新加载、会话替换和进程退出可能汇聚到同一条清理路径。
使用 `ctx.shutdown()` 请求有序关闭进程。

<a id="examples-reference"></a>
<a id="use-examples-as-the-implementation-reference"></a>

<a id="examples-and-reference"></a>

## 示例与参考

经过检查的[扩展示例](../examples/extensions/)涵盖工具、生命周期事件、命令、标志、快捷键、状态、渲染、Provider、OAuth、远程执行和终端组件。
从符合你的集成点的最小示例开始。

模型服务集成请参阅[自定义 Provider](custom-provider.md)，自定义组件请参阅[终端 UI](tui.md)，要将扩展与其他资源一起安装或分发，请参阅 [Pi 包](packages.md)。
