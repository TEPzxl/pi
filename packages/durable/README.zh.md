> 译注：本文是[英文原文](README.md)的完整中文译文，中文内容可能滞后于原文；所有代码示例均保持原样。

<a id="earendil-workspi-durable"></a>
# @earendil-works/pi-durable

> **实验性（Experimental）。** API 在不同版本之间可能变更，恕不另行通知。

一个持久化、可恢复的 Agent 运行框架。对话、模型轮次、工具调用以及你自己的状态，都会先提交到存储，再对外展示任何内容。如果进程在轮次中途终止，重新打开存储即可从停止的位置继续工作。

基于 [`@earendil-works/pi-ai`](../ai/README.md) 访问模型，并使用 `@earendil-works/chord` 管理文档状态。

<a id="table-of-contents"></a>
## 目录

- [安装](#installation)
- [快速开始](#quick-start)
- [概念](#concepts)
- [持久化与恢复](#persist-and-resume)
- [扩展](#extensions)
- [工具](#tools)
- [代码模式](#code-mode)
- [系统提示词](#system-prompt)
- [每个对话的 Agent](#per-conversation-agent)
- [设置](#settings)
- [环境](#environment)
- [重新加载](#reload)
- [观察对话](#watching-a-conversation)
- [忙碌中的对话](#busy-conversations)
- [重置与交接](#reset-and-handoff)
- [上下文压缩](#compaction)
- [Agent 事件（实验性）](#agent-events-experimental)
- [钩子](#hooks)
- [更多对话与分叉](#more-conversations-and-forks)
- [中止与子 Agent](#abort-and-subagents)
- [子任务](#child-tasks)
- [任务图](#task-graph)
- [你自己的状态](#your-own-state)
- [用量与费用](#usage-and-cost)
- [存储](#storage)
- [错误](#errors)
- [示例](#examples)
- [设计文档](#design-documents)

<a id="installation"></a>
## 安装

```bash
npm install @earendil-works/pi-durable @earendil-works/pi-ai @earendil-works/chord
```

<a id="quick-start"></a>
## 快速开始

```typescript
import { BACKGROUND_CONTEXT } from "@earendil-works/chord/context";
import { createModels } from "@earendil-works/pi-ai/models";
import { openaiProvider } from "@earendil-works/pi-ai/providers/openai";
import { AssistantEntry, createRegistry, Harness, MemoryStorage } from "@earendil-works/pi-durable";

const context = BACKGROUND_CONTEXT;

const models = createModels();
models.setProvider(openaiProvider()); // reads OPENAI_API_KEY

const harness = await Harness.open(new MemoryStorage(), { models, registry: createRegistry() }, context);
const root = await harness.root(context, { agent: { model: { provider: "openai", modelId: "gpt-6-sol" } } });

const submission = await root.submit({ type: "input", content: "What is the capital of France?" }, context);
const settled = await submission.wait(context);
if (settled.status === "done" && settled.type === "input") {
	const answer = await root.commit((tx) => tx.entry(AssistantEntry, settled.answer), context);
	console.log(answer?.model?.[0]);
}
await harness.close(context);
```

上述代码的执行过程：

- `Harness.open()` 在存储后端之上打开一个 Session。`MemoryStorage` 将所有内容保留在内存中。
- `root()` 返回根对话，首次使用时按给定的 Agent 选项创建它。对话是由不可变条目组成的记录。
- `submit()` 持久化接收你的输入并返回一个 `Submission`。内置生成任务调用模型并追加回答。
- `wait()` 在输入得到回答（`done`）或失败（`unanswered`，附带原因）后兑现。

每个异步调用都接收一个 Chord `Context`，它承载取消信号。`BACKGROUND_CONTEXT` 永不取消。取消一次等待只会取消该次等待，绝不会取消实际工作。

<a id="concepts"></a>
## 概念

- **Harness（Agent 运行框架）**：一个已打开的存储，加上在其上运行 Agent 的机制。所有变更都经过同一条原子提交序列，提交存储完成之前不会展示任何内容。
- **Conversation（对话）**：一份对话记录。`root()` 在首次使用时创建根对话；你可以创建更多对话并进行分叉。`Conversation` 句柄本身不持有状态；应通过 `id` 比较句柄。
- **Entry（条目）**：一条不可变的对话记录，例如用户消息（`pi.user`）、模型响应（`pi.assistant`）、工具结果（`pi.tool-result`）、系统提示词变更（`pi.system`）、重置（`pi.reset`），或你自己的条目类型。模型只能看到最近一次重置起的条目。
- **Commit（提交）**：一次原子写入。`conversation.commit((tx) => ...)` 可以同时追加条目、编辑文档并创建任务；要么全部存储，要么全部不存储。
- **Document（文档）**：与对话记录一同存储、通过提交修改的类型化 JSON 状态。内置文档分别保存每个对话的 Agent 选项（`pi.agent`）、面向提供商的会话身份（`pi.provider`）、正在运行的生成任务和工具（`pi.live`）、排队中的提交（`pi.inbox`）以及开销（`pi.usage`）。
- **Task（任务）**：一个持久化状态机，每一步都会保存检查点，使重启后的进程从最近的检查点继续。每个任务都有一个所有者：其对话，或另一个任务。Agent 运行框架将回答作为内置任务运行：`pi.generation` 调用模型，拥有各个工具调用对应的 `pi.tool` 任务，等待它们完成，然后将运行过程交给下一次生成。
- **Submission（提交项）**：你交给对话的内容，可以是用户输入，也可以是要写入的条目；你可以等待它完成。
- **Turn 与 run（轮次与运行）**：一个轮次是一次模型响应及其工具调用；一次运行是从输入到最终回答所经历的所有轮次。运行进行期间，对话处于忙碌状态。
- **Extension（扩展）**：一组具名的工具、系统提示词段落、钩子、包装器和任务。
- **Registry（注册表）**：当前进程安装的扩展。它可以在 Agent 运行框架运行期间变更；新工作使用新状态。
- **Agent**：对话运行时采用的模型、思考级别、选定扩展、工具、指令和工作目录。这些内容以名称形式存储在每个对话的 `pi.agent` 中，每次使用时根据注册表解析。

一次已得到回答的输入，对应以下条目与任务：

```text
submit(input) → pi.user
  pi.generation → pi.system (only if the prompt or tools changed), pi.assistant (tool calls)
    pi.tool × n → pi.tool-result × n   (owned by the generation, which waits for them)
  pi.generation → pi.assistant (answer) → submission done
```

<a id="persist-and-resume"></a>
## 持久化与恢复

使用 SQLite 或 JSONL 存储，在重启后保留对话：

```typescript
import { openNodeSqliteStorage } from "@earendil-works/pi-durable/storage/sqlite/node";

const harness = await Harness.open(await openNodeSqliteStorage("./session.sqlite"), { models, registry }, context);
const root = await harness.root(context); // the same root as last time
harness.resume(); // continue any run the last process left unfinished
```

因崩溃或关闭而中断的工作会保持待处理状态。`resume()` 启动任务调度器；提交或等待也会启动它。每个对话都在 `pi.provider` 中持久化保存自己的 UUIDv7，并将其作为 `sessionId` 转发给 pi-ai，用于提供商的提示词缓存和会话亲和性。它在重新打开、重试、重置、上下文压缩和模型变更后保持不变；子对话或分叉会获得新的身份。旧版对话会在首次生成或上下文压缩请求之前获得并持久化保存一个身份。

使用同一 `requestId` 重试提交时，会返回已有提交项，而不是重复提交：

```typescript
const submission = await root.submit({ type: "input", content: "Hello", requestId: "greeting-1" }, context);
// After a restart: the same request ID finds the same submission.
const again = await root.submit({ type: "input", content: "Hello", requestId: "greeting-1" }, context);
// again.id === submission.id
```

`harness.submission(id)` 按 ID 重新获取一个提交项，例如在重启后继续等待它。

<a id="extensions"></a>
## 扩展

除内置任务外，Agent 运行框架执行的代码都来自具名扩展，它们安装在由你的进程持有的注册表中：

```typescript
import { createRegistry, defineExtension, defineTool, hook, section, ToolTask } from "@earendil-works/pi-durable";
import { CodingTools } from "@earendil-works/pi-durable/tools";

const Coding = defineExtension({
	name: "coding",
	sections: [section("preamble", () => "You are a concise coding assistant.", { tag: false })],
	hooks: [hook(ToolTask, { beforeTool: (call) => (isDangerous(call) ? { block: "Needs approval" } : undefined) })],
});

const registry = createRegistry();
registry.install(CodingTools);
registry.install(Coding);
```

扩展可以包含 `tools`、`sections`、`hooks`、`wraps`（按名称装饰工具或提示词段落）和 `tasks`。默认情况下，每个对话都会按安装顺序选用全部已安装扩展。注册表中的内容不会被存储；对话存储的是扩展名称。

<a id="tools"></a>
## 工具

`@earendil-works/pi-durable/tools` 提供 `read`、`write`、`edit` 和 `bash`，以及包含这四种工具的 `CodingTools` 扩展。它们仅通过调用的环境访问文件和进程（见[环境](#environment)）。

`read` 将图片文件作为一个图片块返回，调用它的程序会得到一个 `ImageContent`。限制取自模型的 `inputLimits.images.resize`，默认是 2000x2000 像素和 4.5 MB 的 base64 数据。没有图片处理器时，它直接发送符合这些限制的 PNG、JPEG、GIF 和 WebP 文件（尺寸从文件头读取），不对其解码，并拒绝更大的图片及其他格式。配置图片处理器后，它会解码每张图片，按 EXIF 方向转正，将 BMP 转换为 PNG，并缩小到限制范围内。基于 Photon 的处理器运行于 WebAssembly，仅在你导入时加载：

```typescript
import { createNodePhotonImages } from "@earendil-works/pi-durable/images/node"; // or /images/cloudflare, or createPhotonImages(wasm) from /images
import { createCodingTools } from "@earendil-works/pi-durable/tools";

registry.install(createCodingTools({ images: await createNodePhotonImages() }));
```

不支持图片输入的模型会看到占位符而非图片，`read` 会在诊断信息中说明这一点。`/images/node` 在工作线程中处理图片，因此大照片（一到两秒的工作量）不会阻塞事件循环；工作线程在 30 秒内没有图片需要处理后退出，释放 Photon 的 WebAssembly 内存（`createNodePhotonImages({ worker: false, idleMs })`）。在其他环境中，`createPhotonImages()` 和 `/images/cloudflare` 在调用线程中工作，解码会在 WebAssembly 内存中占用图片完整尺寸所需的空间，之后这些内存不会再次缩小；在 Cloudflare Workers 等内存受限的宿主环境中，除非图片始终很小，否则不要配置处理器。

使用 TypeBox schema 定义你自己的工具。`defineTool()` 根据 `parameters` 为 `args` 推导类型，Agent 运行框架会在 `execute()` 之前验证参数。`api.output()` 流式输出运行中的内容；当 `execute()` 没有返回 `output` 时，这些内容会成为结果：

```typescript
import { Type } from "@earendil-works/pi-ai";

const count = defineTool({
	name: "count",
	description: "Count from 1 to n",
	parameters: Type.Object({ n: Type.Number() }),
	execute: async (args, api) => {
		for (let i = 1; i <= args.n; i++) api.output(`${i}\n`);
		return {};
	},
});
registry.install(defineExtension({ name: "count", tools: [count] }));
```

每次调用都作为独立的持久化任务运行。调用意图会在 `execute()` 执行前提交。如果进程在调用中途终止，只有声明了 `replay: "safe"` 的工具才会在重新打开时重新运行；否则模型会收到一个 `interrupted` 错误结果，其中包含截至中断前已提交的输出。`execute()` 抛出异常会向模型提供错误结果。结果还可以返回 `usage`，它会累加到对话的[用量](#usage-and-cost)中。结果也可以返回 `control: { terminate: true }`：当该轮所有结果都提出这一请求时，运行会结束，不再发起另一次模型请求。

结果有三个通道：供模型使用的 `output`、供调用该工具的程序使用的 `structuredOutput`（见[从工具调用工具](#calling-tools-from-tools)），以及供 UI 使用的 `details`。声明了 `structuredOutputSchema` 的工具应返回匹配的 `structuredOutput`，Agent 运行框架会进行验证。程序调用没有声明该 schema 的工具时，会得到受输出限制约束的内容：单个文本项作为字符串，单张图片作为其 `ImageContent`，没有内容则为 `""`，其他情况则为内容列表；错误结果也遵循这一规则。`structuredOutput` 始终是工具自身的值：声明了 schema 的工具在错误结果中保留它返回的值；未声明 schema 的工具，其错误文本放在 `tool_error` 诊断信息中，抛出异常时的错误消息也如此。对话记录只存储 `output` 和 `details`：

```typescript
const lines = defineTool({
	name: "lines",
	description: "Count the lines of a file",
	parameters: Type.Object({ path: Type.String() }),
	structuredOutputSchema: Type.Object({ lines: Type.Number() }),
	execute: async (args, api, context) => {
		const text = await readText(api, args.path, context);
		const count = text.split("\n").length;
		return { output: [{ type: "text", text: `${count} lines` }], structuredOutput: { lines: count } };
	},
});
```

缺失、未声明或无效的 `structuredOutput` 会将嵌套调用的结果变成附带 `invalid_structured_output` 诊断信息的错误结果；对于由模型发起的调用，由于模型永远看不到它，这个值会被丢弃，并通过 `onReport` 报告。错误结果可以省略它，此时程序可从 `tool_error` 诊断信息中获取错误文本；Agent 运行框架自己写入的错误结果（被阻止、无效、中断、中止）不携带此值，其 `diagnostics` 会说明问题。对声明了 schema 的工具，若 `afterTool` 钩子对其 `output` 做了脱敏，也必须对其 `structuredOutput` 做脱敏。内置 `bash` 返回 `{ output, truncated, fullOutputPath?, exitCode }`，其中 `output` 是通过 `api.retainedOutput()` 读取、模型所能看到的保留尾部输出；即使以非零退出码结束，它也会返回仍携带该值的错误结果。

如果两个扩展都被选用，后安装扩展中的同名工具会替换先前的工具，而 `wrapTool()` 会装饰最终生效的工具：

```typescript
const Venv = defineExtension({ name: "venv", tools: [createBashTool({ commandPrefix: "source .venv/bin/activate" })] });
const Timing = defineExtension({
	name: "timing",
	wraps: [wrapTool(createBashTool(), (bash) => ({ ...bash, execute: (args, api, ctx) => timed(() => bash.execute(args, api, ctx)) }))],
});
```

<a id="calling-tools-from-tools"></a>
### 从工具调用工具

工具使用 `api.executeTool()` 调用另一个工具，代码模式脚本或 MCP 桥接也是如此。嵌套调用本身是一个工具任务，归调用它的任务所有：它会接受验证、运行 `ToolTask` 钩子、保留自身的输出限制和重放策略，并将结果返回给调用方，而不是写入对话记录：

```typescript
const testsPass = defineTool({
	name: "tests_pass",
	description: "Run the test suite and say whether it passed",
	parameters: Type.Object({}),
	execute: async (_args, api, context) => {
		const run = await api.executeTool("bash", { command: "npm test" }, context, { progress: false });
		// Absent when bash did not finish, for example on a timeout.
		const exitCode = (run.structuredOutput as { exitCode: number } | undefined)?.exitCode;
		return { output: [{ type: "text", text: exitCode === 0 ? "passed" : "failed" }] };
	},
});
```

- 调用方获得嵌套调用的 `taskId`、`structuredOutput`、`details`、`diagnostics`、`usage` 和 `isError`，而非供模型使用的 `output`。被阻止、无效、失败或中止的嵌套调用会返回 `isError` 结果。一旦 `execute()` 已返回、调用方被中止或 Agent 运行框架关闭，`executeTool()` 本身就会拒绝；取消其 `context` 只会停止等待。
- 钩子可以看到嵌套调用的 `call.parent`，因此防护逻辑可以区别对待来自脚本的调用。
- 调用方运行期间，其嵌套调用会显示在 `docs["pi.live"].nestedTools` 中，包括实际使用的参数，以及完成后的 `summary`（错误、时长、用量）；也会作为带有 `parentToolCallId` 和 `parentTaskId` 的工具事件呈现。`progress: false` 会阻止嵌套调用的运行中输出进入 `pi.live`。在调用方进入完成状态之前，它遗留的仍在运行的嵌套调用会被中止，包括之前某次尝试创建的调用。调用方的结果消息不会列出其嵌套调用；它们保留在各自的任务记录中。
- 每个嵌套调用都有一个键，默认是它在调用方嵌套调用中的顺序位置，除非调用方传入 `{ key }`。可安全重放的调用方在崩溃后重新运行时，如果按相同顺序调用，或传入相同的键，就会重新获得它已经发起的嵌套调用，无论这些调用已经完成还是仍在运行，而不是再次启动它们。如果崩溃在某个嵌套调用提交结果前打断了它，该调用会遵循自身的 `replay` 策略，因此具有外部副作用的可安全重放工具仍需自行保证幂等性。不可安全重放的调用方绝不会重新运行，所以重启后，它未完成的嵌套调用及这些调用所拥有的所有工作都会在再次运行之前被放弃。
- 嵌套调用的 `usage` 记在其自身工具名下；若将其继续传入调用方的结果，会重复计数。
- 嵌套结果不会长期保留：它们仅存放在调用方的任务文档中（`NestedCallDoc`，每个嵌套调用一份，按其键索引，保存 `{ taskId, result? }`），直到调用方进入完成状态。嵌套调用的任务记录只保留一份简短回执。存储的结果与 `executeTool()` 返回的内容完全一致，不包含供模型使用的 `output`，因此图片或 `bash` 尾部输出只会存储一次。
- 调用嵌套深度没有限制；使用默认 `callers` 的工具甚至可以调用自身。在需要限制的场景中约束 `callers`（见下文）。

<a id="who-may-call-a-tool"></a>
### 谁可以调用工具

工具上的 `callers` 指定谁可以调用它：模型、通过 `executeTool()` 调用的其他工具，或两者（默认）。对话的 `modelTools` 可以缩小向模型提供的工具范围，但不会取消其他工具对这些工具的访问权：

```typescript
const codemode = defineTool({ name: "codemode", callers: ["model"], ... }); // scripts cannot start scripts
const search = defineTool({ name: "mcp__github__search", callers: ["tools"], ... }); // never offered to the model

// The model is offered only codemode; its scripts call read, bash, and the MCP tools.
await root.configure({ modelTools: [codemode] }, context);
// The model is offered everything but bash, which scripts may still call.
await root.configure({ modelTools: { remove: [bash] } }, context);
```

`conversation.agent()` 解析这两份列表：向模型提供的 `tools`，以及嵌套调用从中解析工具的 `callable`。是否启用某个工具仍由 `tools` 决定；`modelTools` 不能添加未启用的工具。

声明了 `deferred: true` 的工具在加载前不会提供给模型，但其他工具随时可以调用它。`@earendil-works/pi-durable/tool-search` 中的 `tool_search` 工具（`createToolSearchExtension()`）允许模型搜索延迟加载的工具，并通过 `control.addTools` 加载匹配项，将它们记录到 `modelTools` 中；加载后的工具会在该分支持续提供给模型，上下文压缩后也不例外，且 `codemode` 提示词段落会排除尚未加载的延迟工具。大型工具集（例如 MCP 服务器的工具集）适合使用这一机制。

<a id="tasks-a-restart-abandons"></a>
### 重启后放弃的任务

如果创建者只在内存中等待一个任务，且重启后不会恢复等待，可以在创建该任务时设置 `abandonOnRestart: true`。后续的 Agent 运行框架启动调度时，若发现该任务仍处于活动状态，会在它及其拥有的所有工作再次运行之前，以 `abortReason: "restart"` 将它们一并中止。不可安全重放工具的嵌套调用会自动采用此设置，这类工具通过 `api.createTask()` 创建的子任务也是如此。如果以此方式放弃的任务缺少定义，它会等待扩展安装，以便其中止处理器仍能执行清理。调用方和对话也会一同等待；`harness.inspect()` 会将其显示为被阻塞。此时中止它会使其成为孤立任务，与其他中止操作相同。

<a id="code-mode"></a>
## 代码模式

`@earendil-works/pi-durable/codemode` 添加了 `codemode` 工具：模型编写 JavaScript，在 QuickJS 沙箱（`@earendil-works/pi-codemode`）中通过 `await tools.<name>(args)` 调用对话的工具。每次调用都是嵌套工具调用，只有脚本的输出会传给模型。脚本可以跨重启恢复：它从原位置继续，不会重新执行已完成的调用。

```typescript
import { CodemodeSandbox } from "@earendil-works/pi-codemode";
import { createCodemodeExtension } from "@earendil-works/pi-durable/codemode";

registry.install(createCodemodeExtension({ sandbox: new CodemodeSandbox({ memoryLimitBytes: 256 * 1024 * 1024 }) }));
```

该扩展还会添加一个 `codemode` 提示词段落，在 token 预算内以 TypeScript 声明列出脚本可调用的工具；脚本使用 `searchTools()`、`describeTool()` 和 `describeNamespace()` 查找其他工具。可通过 `namespaces: [{ name, description, prefix }]` 按名称前缀分组，例如 MCP 服务器的 `mcp__github__`。结果的呈现方式与 coding agent 类似：先是头部，再是脚本输出，失败时还包含已经发起的调用；超出 `max_output_tokens` 的文本会被截断，完整文本保存在临时文件中。脚本也能使用 `models.*`：模型目录、`models.classify()` 和 `models.generateImages()`，它们作为内部工具的嵌套调用运行（`models: false` 可关闭它们）。在 Cloudflare 上应改为传入 `RemoteCodemodeSandbox`。脚本运行期间关闭沙箱（例如应用关闭时）不会被视为脚本的最终结果：脚本会在下一个进程的沙箱中恢复。失败的嵌套调用会在脚本中以错误文本拒绝，除非该工具声明了 schema 并在错误结果中返回了自己的值：`tools.bash` 即使以非零退出码结束，也会兑现为 `{ output, exitCode, ... }`，因此脚本需要检查 `exitCode`。`store(key, value)` 和 `load(key)` 按对话保存 JSON 值；分叉看到的是这些值在分叉条目时的状态。

跨重启恢复的原理：沙箱以确定性方式运行脚本（虚拟时钟、带种子的随机数、UTC），工具会先将脚本收到的每个结果写入日志，再交给脚本。重启后，工具重新运行脚本，并按相同顺序交付日志中的结果，因此脚本会以相同参数发起相同调用；这些调用通过键重新关联到原有嵌套调用，随后脚本继续执行。如果重跑时脚本走了与之前不同的路径、沙箱指纹或代码模式版本发生变化，或其 `timeout_ms` 已在此期间到期，调用就会改为以错误结束。重启时正在运行的嵌套调用遵循自身的重放策略。

<a id="system-prompt"></a>
## 系统提示词

系统提示词由所选扩展的提示词段落组成，在每次请求之前按顺序渲染。段落可以访问解析后的 Agent、为该请求构建的环境以及已提交的文档：

```typescript
section("cwd", (input) => input.env?.cwd); // rendered as <cwd>\n...\n</cwd>; undefined omits it
```

对话的 `instructions` 最后渲染为名为 `instructions` 的段落。段落和工具的变更会作为对话记录中相应位置的系统条目存储。只会重新发送发生变化的部分，从而维持提供商提示词缓存的命中。如果某个段落每次都返回不同内容（例如当前时间），就会破坏这一效果。

<a id="per-conversation-agent"></a>
## 每个对话的 Agent

每个对话将其运行选项保存在自己的 `pi.agent` 文档中。`configure()` 通过一次提交修改它；未设置的字段遵循宿主配置：

```typescript
await root.configure(
	{
		model: { provider: "openai", modelId: "gpt-6-sol" },
		thinkingLevel: "high",
		extensions: { remove: [Coding] }, // edits the host default; an array selects exactly these, in order
		tools: [readTool, bashTool], // an array enables exactly these; { remove: [...] } drops some
		modelTools: [readTool], // of those, offer the model only read; tools may still call bash
		instructions: "Only read; never edit files.",
		cwd: "/work/repo",
	},
	context,
);
await root.configure({ tools: null }, context); // null clears a field back to the host default
const agent = await root.agent(context); // resolved: model, extensions, tools, sections, cwd
```

扩展和工具以对象形式传入，按名称存储，因此存储的名称可以比对应代码存在得更久：卸载某个扩展后，选择了它的对话只是暂时不再获得它，直到它再次安装。`createConversation()`、`fork()` 和 `root()` 都通过 `agent` 接收同样的变更对象。由任务拥有的对话（例如子 Agent 的对话），初始时复制其所有者所在对话的 Agent。分叉初始时采用父对话在分叉条目处的 Agent。一次请求的模型、提示词和提供给模型的工具在准备请求时固定；变更从下一次请求起生效。工具调用和钩子使用各自任务阶段解析出的 Agent，而环境在每次使用时根据当前 `cwd` 构建，因此 `cwd` 或扩展的变更可能影响模型已经发起的调用。

<a id="settings"></a>
## 设置

所有对话共享的运行策略通过 `settings` 传入。每次使用时都会读取，且从不存储，因此可以通过 getter 使其动态生效，例如由设置文件提供数据：

```typescript
const harness = await Harness.open(storage, {
	models,
	registry,
	settings: {
		extensions: [CodingTools, Coding], // default selection; absent: every installed extension
		stream: { timeoutMs: 120_000 },
		retry: { maxRetries: 3 },
		compaction: { reserveTokens: 16384 },
		progress: { partialIntervalMs: 100, outputIntervalMs: 100 },
		toolExecution: "parallel",
		contextRetentionMs: 600_000, // idle conversations keep their context in memory this long (checked lazily where timers cannot be unreferenced)
		documentRetentionMs: 600_000, // loaded documents nothing observes stay in memory this long after their last use; observed ones always
		get followUpMode() {
			return userSettings.followUpMode;
		},
	},
}, context);
```

<a id="environment"></a>
## 环境

`env` 为每次工具调用、提示词段落渲染以及 `runtime.env()` 构建执行环境。它接收对话 ID、其 Agent 的 `cwd` 以及对已提交状态的读取能力，因此同一个函数既可以为每个对话提供独立目录，也可以为每个对话提供独立容器：

```typescript
import { NodeExecutionEnv } from "@earendil-works/pi-durable/env/node";

const harness = await Harness.open(storage, {
	models,
	registry,
	env: ({ cwd }) => new NodeExecutionEnv({ cwd: cwd ?? process.cwd() }),
}, context);
```

`env` 抛出的异常会成为该次调用的错误结果。没有环境时，内置工具会以错误结果失败。每次调用使用一个全新的环境对象也没有问题：`edit` 和 `write` 根据环境的 `id` 和路径，对同一文件的修改进行串行化。自定义 `ExecutionEnv` 应设置 `id`，使相同 ID 在相同路径看到相同文件，例如每个容器对应一个 ID。

宿主也可以直接使用环境，例如展示项目文件。`openBinaryReader()` 读取已打开文件的字节范围，`openDirReader()` 对目录进行分页读取，而接收 argv 数组的 `exec()` 不经过 shell 运行程序，并报告每个输出块来自哪个流：

```typescript
const status = { stdout: "", stderr: "" };
await env.exec(["git", "status", "--porcelain=v2", "-z"], {
	onOutput: (text, _context, { stream }) => {
		status[stream] += text;
	},
}, context);
```

中止 Context 可停止一次调用；`cleanup()` 用于关闭整个环境。自定义环境可以使用 `@earendil-works/pi-durable/testing` 中的 `registerEnvConformance()` 检查自身，与下文的存储检查类似。

某些环境还实现了可选的 `listFiles()`：像 `fd` 一样列出某个目录下任意深度的文件和目录，跳过 git 忽略的内容，达到限制时停止并通过 `truncated` 标记。`NodeExecutionEnv` 未实现此方法；pi-env 的 `RemoteExecutionEnv` 和 `LocalExecutionEnv` 实现了它。如果该方法不存在，或以 `not_supported` 失败，调用方会回退为自行遍历：

```typescript
const listing = await env.listFiles?.(".", { exclude: ["node_modules"], maxEntries: 50_000 }, context);
```

<a id="reload"></a>
## 重新加载

安装与已安装扩展同名的扩展，会以单步操作在原位置替换它：

```typescript
registry.install(await loadCodingExtension()); // same name "coding": replaces the installed one
```

`registry.uninstall(extension)` 会移除该名称对应的已安装扩展，无论它对应的是哪个对象。

已经开始的工作保留它取得的代码：运行中的工具调用会使用旧实现完成，每个任务阶段在阶段开始时从注册表解析一次钩子和 Agent。下一个阶段、请求或调用使用新代码。重启后，需要再次安装相同的扩展；扩展 `tasks` 中待处理的任务会在该扩展安装后恢复。

<a id="watching-a-conversation"></a>
## 观察对话

UI 需要的一切都是已提交状态。`viewState()` 以只读 Chord 状态返回对话的结构化视图，并在每次涉及它的提交后更新：

```typescript
const view = await root.viewState(context);
view.subscribe((value) => {
	// value.entries: the active transcript
	// value.docs["pi.live"]: the running generation (streamed partial, retry, deferred) and tool calls (output, details)
	// value.docs["pi.inbox"], value.docs["pi.usage"], value.docs["pi.agent"], value.docs["pi.provider"]
	render(value);
});
// later: view.dispose();
```

`watch()` 提供相同的视图，并附带每次提交的精确 Chord 操作，每次只执行一个回调：

```typescript
const watch = await root.watch(context);
render(watch.value); // the state at attachment
watch.start(async (value, ops) => {
	await send(ops); // for example to a remote client that applies them
});
// later: await watch.stop();
```

较慢的观察器最多保留 100 个尚未交付的帧。超过后，待交付帧会被一个包含最新完整视图的帧替换。晚加入或重新连接的客户端从当前视图开始；不会重放任何内容。

默认情况下，部分回答和工具输出最多每 100 ms 提交一次，因此崩溃最多丢失这个时间窗口内的内容。`settings.progress` 可以修改间隔；使用远程存储的宿主可以降低提交频率，例如 `{ partialIntervalMs: 500, outputIntervalMs: 500 }`。

<a id="busy-conversations"></a>
## 忙碌中的对话

一次运行正在处理输入时，对话处于忙碌状态。向忙碌的对话提交内容，会将提交项排入该对话的收件箱，即视图中的 `docs["pi.inbox"]`：

```typescript
await root.submit({ type: "input", content: "Also run the tests" }, context); // follow-up (default)
await root.submit({ type: "input", content: "Use pnpm, not npm", whenBusy: "steer" }, context);
await root.submit({ type: "input", content: "Only if idle", whenBusy: "reject" }, context); // throws ConversationBusy
await root.submit({ type: "write", entry: { kind: "app.note", data: "user opened a file" } }, context);
```

- **引导消息（Steers）**在当前一轮工具调用之后放入对话，加入正在进行的工作。
- **后续消息（Follow-ups）**在本次运行给出回答时放入对话，并启动下一次运行。
- **写入（Writes）**只追加条目，不向模型提问。仅包含 `kind` 和 `data` 的写入不会改变模型看到的内容，因此会立即追加，即使运行进行到中途也一样，而且绝不排队：聊天消息一经发送就会显示在对话记录中。因此，它可能出现在更早提交但仍在排队的项目之前；但如果前面有排队的重置，它会等待，以免被重置隐藏。带有 `model`、`head` 或 `edits` 的写入会像引导消息一样等待下一个边界。
- `await submission.abort(context)` 撤回排队中的提交项。
- [设置](#settings)中的 `steeringMode: "all"` 和 `followUpMode: "all"` 会一次性放入所有排队项，而不是每轮一个。

如果运行失败，排队项会留在收件箱中，直到下一次输入或排队写入将它们按最早优先的顺序放入对话。

输入作为 `pi.user` 条目放入对话。如果希望以其他形式展示，可以为它提供一个带有自定义类型和数据的 `entry`；模型仍会将 `content` 视为用户消息：

```typescript
await root.submit(
	{
		type: "input",
		content: "ann: can you check the build?",
		entry: { kind: "chat.mention", data: { author: "ann", text: "can you check the build?" } },
	},
	context,
);
// placed as { kind: "chat.mention", data: { author: "ann", ... }, model: [{ role: "user", content: "ann: can you check the build?" }] }
```

它的行为与其他输入相同：启动运行、引导当前运行、作为后续输入或被拒绝，并会得到回答。排队期间，其收件箱项目携带该条目。类型名称不得以 `pi.` 开头。

<a id="reset-and-handoff"></a>
## 重置与交接

`reset()` 开启一个新的上下文。模型不再看到较早的条目，但它们仍保留在存储中：

```typescript
await root.reset(undefined, context);                                  // start from nothing
await root.reset("We were fixing the flaky login test. Continue.", context); // start from a handoff note
```

忙碌时，重置会像写入一样排队。如果在一轮工具调用期间放入重置，当前运行就会结束。工具可以通过 `control: { handoff: "..." }` 请求相同操作。

<a id="compaction"></a>
## 上下文压缩

上下文压缩缩减模型看到的内容：它对较早的条目生成摘要，并追加一个 `pi.compaction` 条目，其中保存摘要，并将其作为首个保留条目的头部。较早的条目仍保留在存储中。

```typescript
const id = await root.compact("Keep the failing test names", context); // manual, with optional instructions
const { outcome } = (await harness.waitForTask(id, context)).state;
if (outcome.status === "completed" && outcome.result.submissionId !== undefined) {
	const placed = await (await harness.submission(outcome.result.submissionId, context))!.wait(context);
	console.log(placed.status); // "done", or "unanswered" with reason "stale"
}
```

生成摘要期间，对话会继续工作。如果对话空闲，摘要会立即放入；否则会在下一个轮次边界放入。Esc（`abort()`）会取消手动上下文压缩。

生成任务也会自行进行上下文压缩，由[设置](#settings)控制：

```typescript
settings: {
	compaction: {
		enabled: true, // automatic compaction; manual compact() always works
		reserveTokens: 16384, // above contextWindow - reserveTokens, the next request waits for a compaction
		keepRecentTokens: 20000, // roughly how much recent context stays verbatim
		backgroundTokens: 32768, // this far below that, a compaction starts in the background; 0 disables it
	},
}
```

当提供商因上下文过长而拒绝请求时，生成任务会压缩上下文并重试一次。如果摘要的截断位置早于当前上下文的起点，在放入时会以 `stale` 状态结束，因此多个摘要同时进行时，截断位置最靠后的那个持续生效。摘要生成的开销计入 `pi.usage`。`CompactionTask` 上的 `beforeCompact` 钩子可以拒绝压缩，或提供自己的摘要。

如果生成任务在阻塞阈值处等待的上下文压缩失败，生成任务也会失败，不会发送请求：它的输入以 `unanswered` 状态结束，原因为 `model_error`，详情为 `Compaction failed: <error>`。如果压缩没有找到可压缩的内容，或被拒绝，请求则会继续进行。

正在运行的上下文压缩会列在 `docs["pi.live"].compactions` 中，包含原因、尝试次数和重试退避信息。最新的失败会保留在 `docs["pi.live"].compactionFailure`（任务、原因、错误）中，直到某次压缩生成摘要。Agent 事件增加了 `compaction_start` 和 `compaction_end`（压缩失败时带有 `error`），快照中也增加了 `compactions` 和 `compactionFailure` 字段。

<a id="agent-events-experimental"></a>
## Agent 事件（实验性）

如果使用方需要 coding-agent 风格的事件（`message_start`、`message_update`、`tool_execution_start` 等），而非结构化状态：

```typescript
import { watchEvents } from "@earendil-works/pi-durable";

const stream = await watchEvents(harness, root.id, context);
initialize(stream.snapshot); // entries, run, in-flight generation, tools, compactions, inbox, agent, usage
stream.start(async (events) => {
	for (const event of events) console.log(JSON.stringify(event));
});
```

事件由提交派生，每次提交对应一批事件，应用于快照之上。消息和工具更新携带增量：文本与思考内容的追加、工具调用参数文本的追加，以及输出的裁剪和追加。当使用方落后超过 100 批时，会改为收到一个新的 `snapshot` 事件。一次运行的完整事件流见 `test/examples/19-json.ts`。

<a id="hooks"></a>
## 钩子

钩子允许扩展在选择了它们的对话中观察或调整内置任务：

```typescript
import { GenerationTask, hook, ToolTask } from "@earendil-works/pi-durable";

const Guard = defineExtension({
	name: "guard",
	hooks: [
		hook(ToolTask, { beforeTool: (call) => (call.name === "bash" ? { block: "bash is disabled here" } : undefined) }),
		hook(GenerationTask, { onYield: (answer) => (needsMoreWork(answer) ? { continue: "Keep going." } : undefined) }),
	],
});
```

- **生成：** `beforeRequest`（替换某次请求的消息）、`afterResponse`、`onYield`（用另一条用户消息继续运行）和 `afterTools`（一轮工具调用全部完成后运行）。
- **工具：** `beforeTool`（阻止调用或重写参数）和 `afterTool`（替换结果）。

若要将钩子限制在部分对话中，仅在这些对话中选用其扩展，例如 `configure({ extensions: { add: [Guard] } })`。

<a id="more-conversations-and-forks"></a>
## 更多对话与分叉

```typescript
const other = await harness.createConversation({ ownership: { kind: "ownerless" } }, context);
const fork = await root.fork(entryId, { ownership: { kind: "ownerless" } }, context);
```

分叉可以看到父对话截至 `entryId` 的条目，并独立继续。它保留父对话在该条目处的 Agent，但会获得新的提供商会话身份。两种方法都接收 `agent` 和 `init`，在创建时的提交中应用。

<a id="abort-and-subagents"></a>
## 中止与子 Agent

`await root.abort(context)` 停止对话：撤回排队输入（排队写入保留），中止当前工作中的所有任务，并在对话空闲后兑现。

对话可以由任务**拥有**。子 Agent 工具在 `api.commit()` 内通过 `ownership: { kind: "task", taskId: api.taskId }` 创建子对话，然后通过 `api.conversation(id)` 驱动它：

```typescript
const Subagent: Extension = defineExtension({
	name: "subagent",
	tools: [
		defineTool({
			name: "subagent",
			description: "Delegate a self-contained task to a subagent and get its answer back.",
			parameters: Type.Object({ task: Type.String() }),
			replay: "safe", // a rerun after a crash finds the same child and submission
			execute: async (args, api, context) => {
				const child = await api.commit(async (tx) => {
					// The ownership index remembers the child, so a rerun reuses it.
					const existing = (await tx.scanConversations({ ownerTaskId: api.taskId }, 1)).items[0];
					if (existing !== undefined) return existing.id;
					// Starts as a copy of this conversation's agent: model, extensions, tools, cwd.
					const created = await tx.createConversation({ ownership: { kind: "task", taskId: api.taskId } });
					// A cheaper model, and no subagents of its own.
					await configure(tx, created.id, { model: haiku, extensions: { remove: [Subagent] } });
					return created.id;
				}, context);
				await api.details({ conversationId: child }, context); // lets a UI attach to the child
				const request = { type: "input", content: args.task, requestId: `subagent:${api.taskId}` } as const;
				const settled = await (await (await api.conversation(child, context))!.submit(request, context)).wait(context);
				return { output: [{ type: "text", text: settled.status }] };
			},
		}),
	],
});
```

被拥有的工作归其所有者管理：

- 中止调用会中止子对话。调用失败也一样：包括 `execute()` 抛出异常，或崩溃中断了不可安全重放的调用。
- 只有子对话空闲后，父对话才会空闲。
- 使用 `{ background: true }` 创建的任务构成一道边界：它拥有的工作不会因父级中止而停止，也不会使父级保持忙碌。`root.abort(context, { background: true })` 则也会中止它。

示例以实际产品代码展示了这两种模式：

- [`22-subagent-foreground.ts`](test/examples/22-subagent-foreground.ts)：上面的工具，返回子对话的回答。UI 通过调用的 `details` 找到子对话，并将子对话事件缩进显示在该调用下方。
- [`23-subagent-background.ts`](test/examples/23-subagent-background.ts)：通过一个 `subagent` 工具管理持久化子 Agent，支持创建、发送消息（引导或后续）、等待、停止和列出。每个子对话由一个后台锚定任务拥有，因此父级的 Esc 和等待空闲操作都不会影响它。每条消息由一个后台报告任务交付，该任务在回答到达后将其作为后续输入发送回父对话；请求 ID 防止重启后重复发送消息或报告。

<a id="child-tasks"></a>
## 子任务

任务可以拥有使用 `ownership: { kind: "task", taskId }` 创建的子任务，并通过提交 `waiting` 状态等待它们：

```typescript
pay: async (task, runtime, context) => {
	await runtime.commit(async (tx) => {
		const payments = [];
		for (const card of task.input.cards) {
			payments.push(await tx.createTask(Payment, { card }, { ownership: { kind: "task", taskId: task.id } }));
		}
		// Resume in `decide` once every payment is done; the first failure aborts the rest.
		return { status: "waiting", checkpoint: { phase: "decide", payments }, on: payments, policy: "failFast" };
	}, context);
},
decide: async (task, runtime, context) => {
	const outcomes = await runtime.outcomes(task.state.checkpoint.payments, context);
	// ...commit the checkout's own outcome
},
```

- **等待：** 任务等待期间不运行任何代码。使用 `allSettled` 时，`on` 中的所有任务都完成后才会恢复；使用 `failFast` 时，第一个失败的子任务还会中止其余子任务。采用 `allSettled` 时，`on` 也可以指定其他任务。
- **完成：** 如果一个任务结束时，它拥有的工作仍在运行，它会进入 `completing`：其结果已经确定，但只有这些工作完成后，它才进入终态，`waitForTask()` 才返回。失败或中止的结果会先中止这些工作。
- **中止：** 中止按自下而上的顺序执行。中止任务时，先中止它拥有的工作，只有这些工作全部完成后，才启动它自身的中止处理器，让每个任务撤销自己的副作用。

[`24-child-tasks.ts`](test/examples/24-child-tasks.ts) 运行一个包含四笔付款的结账流程，展示银行卡被拒绝、取消结账，以及付款过程中重启的情形。

<a id="task-graph"></a>
## 任务图

`harness.taskGraph(context)` 将 Session 中所有活动任务展示为一个 Chord 状态，可用于任务面板或调试。每个节点包含其所有者连边（`owner` 任务；若任务归对话所有，则没有该连边）、状态、是否为 `background` 或已标记中止，以及它拥有的对话。`harness.watchTaskGraph(context)` 像对话的 `watch()` 一样，以观察器形式提供相同的值。

```typescript
const graph = await harness.taskGraph(context);
graph.subscribe((value) => {
	for (const node of Object.values(value.tasks)) {
		const status = node.state.status === "waiting" ? `waiting on ${node.state.on.join(", ")}` : node.state.status;
		console.log(`${node.id} ${node.kind} ${status}`, node.owner ?? `conversation ${node.conversationId}`);
	}
});
```

任务随创建它的提交出现在图中，随使其进入终态的提交离开。状态均为已提交状态：`pending`、`running`、`waiting`（带有 `on` 和 `policy`）和 `completing`（带有已确定结果的状态）。重启后，原本处于 `running` 的任务会显示为 `pending`，直到再次运行。待处理任务是否因缺少定义而被阻塞不属于任务图内容；`harness.inspect()` 会报告这一点。任务图只列出活动任务：一旦子 Agent 的所有者任务进入终态，其对话中的后续任务会成为顶层节点，对话的 `ConversationRecord.owner`（也在视图的 `conversation` 中）则将它链接到父级。[`24-child-tasks.ts`](test/examples/24-child-tasks.ts) 会在付款运行期间打印结账流程的任务树。

<a id="your-own-state"></a>
## 你自己的状态

文档是与条目一起提交的类型化 JSON 对象。定义一个文档，并在提交中编辑它：

```typescript
import { defineDoc } from "@earendil-works/pi-durable";

const Todos = defineDoc<{ items: string[] }>({
	kind: "app.todos",
	version: 1,
	scope: "conversation",
	history: "latest", // or "rewindable" to read old values with snapshotAsOf()
	fork: "initial", // what a fork starts with: "initial", "current", or "asOf"
	initial: () => ({ items: [] }),
});

await root.commit(async (tx) => {
	(await tx.doc(Todos, root.id)).items.push("write docs");
}, context);
console.log(await harness.snapshot(Todos, root.id, context));
```

`harness.watchDoc()` 和 `harness.documentState()` 像上面的视图一样观察单个文档。`HarnessOptions.conversationCreated(tx, conversation)` 会在每次创建或分叉对话的提交中运行，包括工具直接调用的 `tx.createConversation()`，从而让每个对话都获得你的文档；`createConversation()`、`fork()` 和 `root()` 中的 `init` 则在同一次提交中写入每次调用专有的数据。扩展的工具、提示词段落和钩子通过 `api` 或 `input.read` 读取自己的文档，文档不存在时则视为默认值（[`11-extension-state.ts`](test/examples/11-extension-state.ts)）。

<a id="usage-and-cost"></a>
## 用量与费用

每个对话在 `docs["pi.usage"]` 中保存 token 和费用总量：模型响应按 `provider/model` 汇总，报告用量的工具结果按工具名称汇总。失败和中止的尝试也会计入。整个 Session 的用量可这样获取：

```typescript
const usage = await harness.usage(context); // { models: { "openai/gpt-6-sol": Usage }, tools: {...} }
```

<a id="storage"></a>
## 存储

| 后端 | 导入 | 说明 |
|---|---|---|
| 内存 | 包根路径中的 `MemoryStorage` | 不会持久化任何内容。 |
| SQLite | `@earendil-works/pi-durable/storage/sqlite/node` 中的 `openNodeSqliteStorage(file)` | 单个数据库文件。使用 WAL 模式和 `synchronous = NORMAL`：提交可在进程崩溃后保留；断电或宿主故障可能丢失最新提交。 |
| JSONL | `@earendil-works/pi-durable/storage/jsonl/node` 中的 `openNodeJsonlStorage(directory, context)` | 同一目录中的仅追加文件。传入 `{ fsync: true }` 可在每次提交标记写入前刷盘。完整历史也会保留在内存中，因此内存占用随历史增长；长期运行的 Agent 应使用 SQLite。 |
| Cloudflare Durable Object | `@earendil-works/pi-durable/storage/sqlite/cloudflare` 中的 `openDurableObjectSqliteStorage(ctx.storage)` | 一个以 SQLite 为后端的 Durable Object 的 SQLite 存储；提交就是它的存储事务。 |

同一时间只能有一个进程拥有某个存储；没有跨进程锁。可移植的 SQLite 和 JSONL 核心（`/storage/sqlite`、`/storage/jsonl`）无需 Node API 即可运行，例如在 Bun 或 Cloudflare Durable Objects 上，只需提供异步 `SqliteDatabase` 接口层或来自 `@earendil-works/pi-durable/env` 的 `FileSystem`。

SQLite 适配器实现基于 Promise 的 `exec`、`run`、`get`、`all`、`transaction` 和 `close`。`run`、`get` 和 `all` 接收 SQL 文本及位置绑定参数；适配器可以按 SQL 文本缓存预处理语句。事务回调接收一个事务句柄；事务内的所有工作都必须使用它，且该句柄在回调结束后失效。适配器必须将无关操作和其他事务排队，直到当前事务完成，因此在回调内调用 `database` 本身会永远无法结束：

```typescript
await database.transaction(async (transaction) => {
	await transaction.exec("CREATE TABLE example (value TEXT)");
	await transaction.run("INSERT INTO example (value) VALUES (?)", "stored atomically");
});
```

自定义后端可以使用任意兼容 Vitest 或 Jest 的运行器执行共享一致性检查套件：

```typescript
import { registerStorageConformance } from "@earendil-works/pi-durable/testing";
import { describe, expect, it } from "vitest";

registerStorageConformance({ describe, expect, it }, "My Storage", async (use) => {
	const storage = await openMyStorage();
	try {
		await use(storage);
	} finally {
		await closeMyStorage(storage);
	}
});
```

包根入口会加载 TypeBox，因为工具任务使用 pi-ai 的 `validateToolArguments()` 验证参数。未打包时，这会带来约 23 MB 的峰值 RSS（常驻内存集）开销；经过 tree-shaking 的打包产物中约为 4 MB。

<a id="errors"></a>
## 错误

以下三条规则决定错误的影响：

1. **存储错误会终止 Agent 运行框架。** 遇到错误的调用会以该错误拒绝。之后的所有调用和所有待完成的等待都会以 `SessionFailed` 拒绝（其 `cause` 为该错误）；观察器和事件流以 `session_failed` 结束；运行中的任务收到中止信号；`onReport` 收到一次该错误。随后 Agent 运行框架自行关闭。不会重试任何操作；自定义存储应自行重试其临时故障。重新打开即可恢复：提交具有原子性，因此存储保持一致，使用相同 `requestId` 重新提交会找到已接收的提交项。
2. **无效请求只使该次调用失败：** 对于无效游标、未知对话或文档未保留的历史，存储会抛出 `StorageRequestError`。你取消的读取也只会使该次读取失败。
3. **你的代码只使它自身的单元失败：** 任务阶段抛出异常会使该任务故障，工具抛出异常会得到错误结果，钩子抛出异常会被报告并跳过（`beforeTool` 会阻止调用），提交回调抛出异常会回滚该次提交。监听器抛出异常或返回被拒绝的 Promise 时会被报告，其他监听器仍然运行。例外是宿主 `RegistryReader` 的 `snapshot()` 抛出异常：没有它，Agent 运行框架无法调度，因此会像存储错误一样导致整个框架失败。

Agent 运行框架通过 `close()` 或因存储错误关闭后，`harness.closed` 会结束，因此宿主可在失败后按如下方式重启：

```typescript
harness.closed.then((end) => {
	if (end.reason === "failed") reopen(end.error);
});
```

失败会使 Agent 运行框架自行关闭；即使存储未能正常关闭，`closed` 也会在关闭过程结束后结束。关闭会等待运行中的任务代码返回，因此忽略中止信号的代码会延迟 `closed` 的完成。

<a id="examples"></a>
## 示例

可运行的示例位于 [`test/examples`](test/examples)。从本包目录运行其中一个示例：

```bash
node --conditions=source --experimental-strip-types test/examples/14-chat.ts
```

| 示例 | 展示内容 |
|---|---|
| [14-chat](test/examples/14-chat.ts) | 一次提问与回答 |
| [16-real-model](test/examples/16-real-model.ts) | 从 OpenAI 流式获取回答 |
| [17-coding-tools](test/examples/17-coding-tools.ts) | 在 JSONL 存储上执行一个使用工具的轮次 |
| [18-print](test/examples/18-print.ts) | 打印模式：提交提示词，打印回答 |
| [19-json](test/examples/19-json.ts) | JSON 模式：在 SQLite、JSONL 或内存上输出 Agent 事件或原始视图操作 |
| [20-inbox](test/examples/20-inbox.ts) | 忙碌期间的引导消息、后续消息、写入与撤回 |
| [21-late-join](test/examples/21-late-join.ts) | 在运行中途接入视图和事件流 |
| [22-subagent-foreground](test/examples/22-subagent-foreground.ts) | 可安全重放的子 Agent 工具，其子对话归该调用所有，子对话事件显示在调用下方 |
| [23-subagent-background](test/examples/23-subagent-background.ts) | 持久化子 Agent：创建、引导、停止、列出、回报回答，支持安全重启 |
| [24-child-tasks](test/examples/24-child-tasks.ts) | 一个拥有并等待四笔付款的结账流程：failFast、中止、重启 |
| [25-compaction](test/examples/25-compaction.ts) | 对长对话在后台、手动及上下文溢出后进行压缩 |
| [26-coding-agent](test/examples/26-coding-agent.ts) | CodingTools、从设置对象读取的实时设置，以及跟随对话目录的环境 |
| [27-plan-mode](test/examples/27-plan-mode.ts) | 带有自有文档的只读规划模式扩展，通过 `configure()` 切换 |
| [28-reviewer](test/examples/28-reviewer.ts) | 具有独立模型、扩展、工具、目录和审查循环的审查者对话 |
| [29-sandbox-per-conversation](test/examples/29-sandbox-per-conversation.ts) | 每个对话独立的环境，从应用文档中查找 |
| [30-tool-override](test/examples/30-tool-override.ts) | 为部分对话提供同名 bash，以及对最终生效的 bash 计时的包装器 |
| [31-reload-and-restart](test/examples/31-reload-and-restart.ts) | 调用中途重新加载扩展，以及存储的选项在重启后保留 |
| [00](test/examples/00-conversation.ts)–[13](test/examples/13-recovery.ts) | 底层各层：会话、文档、分叉、观察器、Agent 运行框架、Agent 配置、重新加载、扩展状态、任务、恢复 |

调用 OpenAI 的示例需要 `OPENAI_API_KEY`；其他情况下，大多数示例使用模拟提供商。

<a id="design-documents"></a>
## 设计文档

- [`docs/spec.md`](https://github.com/earendil-works/pi/blob/main/packages/durable/docs/spec.md)：规范性说明

基准测试：`npm run bench:storage`、`npm run bench:storage:memory`、`npm run bench:tool-output` 和 `npm run bench:nested-tools`。

<a id="license"></a>
## 许可证

MIT
