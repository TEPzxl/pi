> 译注：本文译自[英文原文](cli.md)，中文译文可能滞后；代码示例保留原样。

<a id="cli-and-modes-reference"></a>

<a id="command-line"></a>

# 命令行

本页介绍 Pi 内置的命令行命令和选项。运行 `pi --help`，或在命令后附加 `--help`，可查看所安装版本的确切接口。顶层帮助还会包含已加载扩展注册的选项。

```sh
pi [options] [--] [@files...] [messages...]
pi install <source> [options]
pi remove <source> [options]
pi uninstall <source> [options]
pi update [target] [options]
pi list
pi config [options]
pi auth <check|print-api-key|print-bearer-token> [options]
pi mcp <list|login|logout> [options]
```

<a id="modes"></a>

<a id="invocation-and-output"></a>

## 调用与输出

```sh
pi
pi --print "Summarize this repository"
git diff | pi --print "Review this change"
pi --mode json "Inspect this repository" > events.jsonl
```

当 stdin 和 stdout 都连接到终端时，Pi 会打开终端 UI，除非使用 `--print`、`--mode json` 或 `--mode rpc` 选择了其他接口。只要其中一个流被重定向，且未选择 JSON 或 RPC 模式，Pi 就会使用 print 模式。关于如何在交互式、print、JSON、RPC 和 SDK 集成之间选择，请参阅 [CLI 集成](cli-integration.md)。

| 输入 | 行为 |
|---|---|
| `message` | 提供初始提示词 |
| `@path` | 在首条提示词中包含文本文件或图片 |
| 通过管道传入的 stdin | 将其内容置于首条提示词之前 |
| `--` | 停止解析选项，使提示词可以以 `-` 开头 |

Pi 相对于当前工作目录解析 `@path`。工作目录还决定项目配置、资源发现和会话分组。

`--print` 控制 Pi 是否运行一次后退出。`--mode` 选择输出接口。当 stdin 和 stdout 都连接到终端时，`--mode text` 不会强制单次执行；需要这种行为时，请使用 `--print`。

| 选项 | 行为 |
|---|---|
| `-p`、`--print` | 执行提供的提示词，将 assistant 的最终文本写入 stdout，然后退出 |
| `--mode text` | 选择文本输出；当 stdin 和 stdout 都连接到终端时，仍会打开终端 UI |
| `--mode json` | 执行提供的提示词，将 JSONL 事件写入 stdout，然后退出 |
| `--mode rpc` | 从 stdin 读取 JSONL 命令，将响应和事件写入 stdout，直到关闭 |
| `--export <input> [output]` | 将会话文件导出为 HTML 后退出；省略 `output` 时自动推导目标路径 |

RPC 模式不接受 `@file` 参数。JSON 和 RPC 模式将 stdout 专用于协议记录。请参阅 [JSON 事件流](json.md)和 [RPC 协议](rpc.md)。

<a id="model-options"></a>

<a id="models"></a>

## 模型

```sh
pi --model sonnet:high
```

模型选择请参阅[选择模型](models.md)，凭据请参阅 [Provider](providers.md)。

- `--provider <name>`<br>
  将 `--model` 的查找范围限制在一个 Provider 内。必须同时指定 `--model`。
- `--model <pattern>`<br>
  按精确 ID 或模糊匹配的 ID/名称选择模型。接受 `provider/id` 以及可选的 `:<thinking>` 后缀。
- `--api-key <key>`<br>
  使用不持久化的 API key 覆盖值。必须通过 `--model` 或 `--models` 选择模型。
- `--thinking <level>`<br>
  设置为 `off`、`minimal`、`low`、`medium`、`high`、`xhigh` 或 `max`。它会覆盖 `--model` 后缀，并受限于模型的能力范围。
- `--models <patterns>`<br>
  设置用于启动和循环切换的范围，以逗号分隔。接受精确 ID、模糊匹配、不区分大小写的 glob，以及可选的 `:<thinking>` 后缀。
- `--list-models [search]`<br>
  列出可用模型，可选用模糊搜索进行过滤，然后退出。

<a id="session-options"></a>

<a id="sessions"></a>

## 会话

```sh
pi --continue
```

关于恢复、派生、命名和存储会话，请参阅[会话与上下文](sessions.md)。

- `-c`、`--continue`<br>
  继续当前项目最近的会话。
- `-r`、`--resume`<br>
  打开会话选择器。
- `--session <path|id>`<br>
  按文件路径、精确 ID 或部分 ID 打开会话。Pi 会先搜索当前项目；若匹配到其他项目的会话，则会提供派生该会话的选项。
- `--session-id <id>`<br>
  按精确的项目会话 ID 打开会话，不存在时创建。ID 允许包含字母、数字、`.`、`_` 和 `-`。
- `--fork <path|id>`<br>
  将现有会话派生为当前项目的新会话。
- `--session-dir <dir>`<br>
  覆盖存储和查找目录。优先级高于 `PI_CODING_AGENT_SESSION_DIR` 和 `sessionDir` 设置。
- `--no-session`<br>
  使用不持久化的内存会话。
- `-n`、`--name <name>`<br>
  设置会话显示名称。

约束：

- 会话 ID 必须以字母或数字开头和结尾。
- `--fork` 不能与 `--session`、`--continue`、`--resume` 或 `--no-session` 组合使用。
- `--session-id` 不能与 `--session`、`--continue` 或 `--resume` 组合使用。可与 `--fork` 组合以指定新 ID。

<a id="tool-options"></a>

<a id="tools"></a>

## 工具

```sh
pi --tools read,grep,find,ls --print "Review this project"
```

默认工具选择的配置请参阅[设置](settings.md#tools)。

- `-t`、`--tools <list>`<br>
  用逗号分隔的内置、扩展或自定义工具允许列表替换默认选择。条目可以是工具名称或模式，其中 `*` 匹配任意字符。除非有条目以 `mcp__` 开头，否则会保留 MCP 工具（请参阅 [MCP 工具](#mcp-tools)）。仅包含 `+name` 和 `-name` 条目的列表不是允许列表，而是用于修改默认选择。
- `-xt`、`--exclude-tools <list>`<br>
  在应用所有其他选择选项之后，禁用逗号分隔的工具名称或模式，包括 MCP 工具。
- `-nbt`、`--no-builtin-tools`<br>
  禁用默认内置工具，同时保留扩展和自定义工具。
- `-nt`、`--no-tools`<br>
  启动时禁用所有内置、扩展、自定义和 MCP 工具。

默认启用的工具是 `read`、`bash`、`edit` 和 `write`，除非通过 `defaultTools` 修改。使用普通名称的 `--tools` 会替换整个选择，因此必须列出所有需要的工具。与 `defaultTools` 一样，它也接受仅由 `+name` 和 `-name` 条目组成的列表，用于在默认选择中添加或移除工具：`pi --tools +codemode,-write` 会保留其他默认工具、启用 `codemode` 并禁用 `write`。这些条目接受精确工具名称，不接受 `*` 模式；要按模式禁用工具，请使用 `--exclude-tools`。普通名称不能与 `+name`/`-name` 条目混用。`/reload` 会启用新加入 `defaultTools` 的工具，但通过 `-name` 移除的工具仍保持移除状态。

<a id="mcp-tools"></a>

`--tools` 选择向模型声明的工具。它不会移除 MCP 工具，后者的可访问范围由其 [exposure](mcp.zh.md#control-tool-exposure) 决定：`pi --tools read,codemode` 会使每个 MCP 工具仍可从 codemode 脚本调用。没有被任何条目点名或匹配到的 MCP 工具，无论其 exposure 如何，都不会被直接声明；只有在列表中包含 `tool_search` 时，才能由它加载。只要有条目以 `mcp__` 开头，`--tools` 也会过滤 MCP 工具，因此以下命令只保留 `radius` 服务器的工具：

```sh
pi --tools read,bash,codemode,'mcp__radius__*'
```

MCP 资源工具（`list_mcp_resources`、`list_mcp_resource_templates`、`read_mcp_resource`）也算作 MCP 工具。要移除 MCP 工具，请使用 `--exclude-tools 'mcp__*'` 或 [`--no-mcp`](#resource-options)。

| 内置工具 | 用途 |
|---|---|
| `read` | 读取文本文件和受支持的图片 |
| `bash` | 运行 shell 命令 |
| `powershell` | 在 Windows 上运行 PowerShell 命令 |
| `edit` | 对现有文件应用精确文本替换 |
| `write` | 创建或覆盖文件 |
| `grep` | 搜索文件内容 |
| `find` | 使用 glob 模式查找路径 |
| `ls` | 列出目录内容 |

内置扩展还添加了两个工具。它们默认关闭；当 MCP 服务器需要时，MCP 扩展会将其开启（请参阅 [MCP](mcp.zh.md#control-tool-exposure)）。要自行启用，请在 `--tools` 或 `defaultTools` 中列出它们。

| 内置扩展 | 用途 |
|---|---|
| `codemode` | 运行调用其他工具的 JavaScript，例如通过 `Promise.allSettled` 并行调用；只有脚本的输出会传给模型 |
| `tool_search` | 搜索尚未向模型声明的工具（exposure 为 `codemode` 和 `deferred` 的工具，例如 MCP 工具），并为下一次调用声明匹配结果 |

<a id="enable-codemode"></a>

### 启用 codemode

要为每个会话启用 `codemode`，请将其加入 `~/.pi/agent/settings.json` 或项目的 `.pi/settings.json` 中的默认工具：

```json
{
  "defaultTools": ["+codemode"]
}
```

这会保留 `read`、`bash`、`edit` 和 `write`，并添加 `codemode`。要仅在一次调用中添加，请使用 `--tools`：

```sh
pi --tools +codemode
```

即使不使用 MCP，Codemode 也很有用：脚本可以并行执行多个工具调用，在大型输出传给模型之前进行过滤，通过 `models.classify()` 调用 TypeSafe 的 Jev 等分类模型（请参阅[分类模型](models.md#use-classifier-models)），还可以通过 `models.generateImages()` 生成图片（请参阅[图像模型](models.md#use-image-models)）。

<a id="how-codemode-works"></a>

### codemode 的工作原理

脚本在 QuickJS 沙箱中运行，通过 `tools.<name>(args)` 访问其他工具。[Codemode](codemode.md) 介绍了脚本 API、工具的列出和查找方式、`store()` 和 `models` 全局对象以及相关限制。

<a id="tool-search"></a>

### 工具搜索

`tool_search` 默认关闭；可通过 `"defaultTools": ["+tool_search"]` 或 `--tools` 启用。它针对尚未声明的工具使用与 `searchTools()` 相同的排序方式，并为下一次模型调用声明匹配结果。加载的工具会像其他工具变更一样记录在会话中，因此会在该分支上保持已声明状态。

<a id="resource-options"></a>

<a id="resources"></a>

## 资源

```sh
pi --extension ./review.ts
```

常规目录和项目信任请参阅[配置](configuration.md)，配置的路径请参阅[设置](settings.md#resources)，包来源请参阅 [Pi 包](packages.md)。

- `-e`、`--extension <path>`<br>
  加载扩展文件或目录，或 `builtin:mcp` 等内置扩展，可重复指定。
- `-ne`、`--no-extensions`<br>
  禁用发现的、已配置的和内置的扩展。显式指定的 `-e` 路径仍会加载，因此 `pi -ne -e builtin:mcp` 只保留内置 MCP 支持。
- `--no-mcp`<br>
  在本次运行中禁用内置 MCP 支持：不连接服务器，不提供 MCP 工具或 `/mcp`。它不影响替代内置 MCP 支持的扩展。
- `--skill <path>`<br>
  加载 skill 文件或目录，可重复指定。
- `-ns`、`--no-skills`<br>
  禁用发现的和已配置的 skills。显式指定的 `--skill` 路径仍会加载。
- `--prompt-template <path>`<br>
  加载提示词模板文件或目录，可重复指定。
- `-np`、`--no-prompt-templates`<br>
  禁用发现的和已配置的模板。显式指定的 `--prompt-template` 路径仍会加载。
- `--theme <path>`<br>
  加载主题文件或目录，可重复指定。
- `--use-theme <name[/name]>`<br>
  选择本次运行的初始交互式主题。
- `--no-themes`<br>
  禁用发现的和已配置的主题。显式指定的 `--theme` 路径仍会加载。
- `-nc`、`--no-context-files`<br>
  禁用 `AGENTS.md` 和 `CLAUDE.md` 发现。

资源路径只对当前进程生效。相对路径基于当前工作目录解析。

<a id="prompt-and-display-options"></a>

<a id="prompts-and-process"></a>

## 提示词与进程

```sh
pi --append-system-prompt ./instructions.md
```

已保存的配置请参阅[配置](configuration.md)，项目信任请参阅[安全](security.md#understand-project-trust)，进程控制请参阅[环境变量](environment-variables.md)。

- `--system-prompt <text|path>`<br>
  用文本或现有文件的内容替换默认 system prompt。
- `--append-system-prompt <text|path>`<br>
  将文本或现有文件追加到 system prompt，可重复指定。
- `--tui-mode <mode>`<br>
  使用 `fullscreen`（默认）或 `regular` 终端模式。
- `--verbose`<br>
  显示详细的交互式启动信息，覆盖 `quietStartup`。
- `-a`、`--approve`<br>
  在此进程中信任项目本地配置和资源。
- `-na`、`--no-approve`<br>
  在此进程中忽略受信任检查限制的项目本地配置和资源。
- `--offline`<br>
  禁用自动网络活动，包括模型目录刷新。等同于 `PI_OFFLINE=1`。
- `-h`、`--help`<br>
  显示帮助（包括已加载扩展注册的标志），然后退出。
- `-v`、`--version`<br>
  显示 Pi 版本，然后退出。

扩展可以注册额外的长选项。未知短选项会被拒绝。

<a id="package-commands"></a>

## 包命令

```sh
pi install npm:@scope/package
```

来源格式、过滤、安装和项目作用域请参阅 [Pi 包](packages.md)。

<a id="common-tasks"></a>

### 常见任务

| 任务 | 命令 |
|---|---|
| 安装包 | `pi install <source>` |
| 列出已配置的包 | `pi list` |
| 移除包及其设置条目 | `pi remove <source>` |
| 配置要加载的包资源 | `pi config` |

为 `install`、`remove`、`uninstall` 或 `config` 添加 `--local` 或 `-l`，即可使用项目设置而非全局设置。

<a id="update-pi-or-packages"></a>

### 更新 Pi 或包

不指定目标运行 `pi update` 时，会更新 Pi 本身。

| 任务 | 命令 |
|---|---|
| 更新 Pi | `pi update` |
| 更新所有已安装的包 | `pi update --extensions` |
| 更新一个已安装的包 | `pi update <source>` |
| 刷新模型目录 | `pi update --models` |
| 更新 Pi 和所有已安装的包 | `pi update --all` |

所选更新包含 Pi 时，可添加 `--force` 重新安装 Pi。

如果 Pi 由 Nix 等其他包管理器提供，`pi update` 无法更新 Pi。请使用对应的包管理器更新 Pi，例如 `nix profile upgrade pi`。包和模型目录更新仍可正常使用。

<a id="aliases-and-command-options"></a>

### 别名与命令选项

- `pi uninstall <source>` 是 `pi remove <source>` 的别名。
- `pi update --self`、`pi update self` 和 `pi update pi` 是 `pi update` 的别名。
- `pi update --extension <source>` 是 `pi update <source>` 的别名。
- `-a`、`--approve` 在单次命令中信任项目本地文件。`-na`、`--no-approve` 忽略受信任检查限制的项目本地文件。
- 在命令后附加 `-h` 或 `--help`，可查看其确切用法和选项约束。

<a id="credential-commands"></a>

## 凭据命令

```sh
pi auth check --provider openai --json
```

身份验证命令需要 `--provider <provider>` 或 `--model <model>`。支持的方法请参阅 [Provider](providers.md)。

| 命令 | 说明 |
|---|---|
| `pi auth check` | 输出 `ready`、`not_ready` 或 `invalid`；分别以状态码 `0`、`1` 或 `2` 退出 |
| `pi auth print-api-key` | 输出解析得到的 API key |
| `pi auth print-bearer-token` | 输出解析得到的 OAuth bearer token |

| 选项 | 适用命令 | 说明 |
|---|---|---|
| `--provider <provider>` | 全部 | 解析某个 Provider 的凭据 |
| `--model <model>` | 全部 | 根据模型解析凭据；可与 `--provider` 组合使用 |
| `--json` | `auth check` | 将结构化结果以 JSON 写出 |
| `--credentials` | `auth check` | 就绪时输出解析得到的凭据 |
| `--no-refresh` | `auth check` | 不刷新已过期的 OAuth 凭据；默认会刷新 |
| `--min-expiry <duration>` | `print-bearer-token` | 使用 `ms`、`s`、`m` 或 `h` 指定 token 所需的剩余有效期，例如 `30m` |

输出凭据的命令会将秘密信息写入 stdout。

<a id="mcp-commands"></a>

## MCP 命令

这些命令可以在会话之外使用，因此 agent 可以通过 `bash` 运行它们。请参阅 [MCP 服务器](mcp.zh.md)。

| 命令 | 说明 |
|---|---|
| `pi mcp add <server> [options] -- <command> [args...]` | 在 `mcp.json` 中添加或替换 stdio 服务器；`--env KEY=VALUE`（可重复指定）和 `--cwd <dir>` 设置其环境和工作目录。命令之后的参数会传递给该命令 |
| `pi mcp add <server> [options] --url <url>` | 添加或替换 streamable HTTP 服务器；`--header KEY=VALUE`（可重复指定）、`--bearer-token-env-var <NAME>`（发送 `Authorization: Bearer ${NAME}`）、`--oauth-client-id`、`--oauth-client-secret`、`--oauth-callback-port` 和 `--oauth-client-name` 配置身份验证 |
| `pi mcp remove <server>` | 从 `mcp.json` 中移除服务器；保留已存储的 OAuth 凭据 |
| `pi mcp list [--json]` | 连接每个已启用的服务器，并输出其状态、工具和错误；当配置条目无效或某个已启用的服务器未连接时，以状态码 `1` 退出 |
| `pi mcp login <server> [--timeout <seconds>]` | 登录 OAuth 服务器：打开授权页面并等待浏览器；在终端中也可粘贴重定向 URL。`--timeout` 限制整个登录过程，包括向授权服务器发送请求的时间（默认 300 秒） |
| `pi mcp logout <server>` | 删除服务器已存储的 OAuth 凭据 |

`add` 和 `remove` 会修改 `~/.pi/agent/mcp.json`；使用 `--local`（`-l`）时，则修改当前目录下的 `.pi/mcp.json`。`add` 还接受 `--exposure <mode>`（请参阅 [Exposure](mcp.zh.md#control-tool-exposure)）和 `--description <text>`，且不会建立连接；请运行 `pi mcp list` 检查服务器。

仅对已受信任的项目读取项目 `.pi/mcp.json` 文件。
