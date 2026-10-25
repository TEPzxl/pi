> 中文译文：[英文原文](mcp.md)。译文可能滞后于原文，技术细节以英文原文和当前源码为准；所有代码示例保持原样。

<a id="mcp-servers"></a>

# MCP 服务器

Pi 通过 stdio 或 Streamable HTTP 连接 [Model Context Protocol](https://modelcontextprotocol.io) 服务器，并让模型能够使用其工具和资源。

<a id="quick-setup"></a>

## 快速配置

添加一个本地 stdio 服务器，检查连接，然后启动 Pi：

```bash
pi mcp add filesystem -- npx -y @modelcontextprotocol/server-filesystem .
pi mcp list
pi
```

对于远程服务器：

```bash
pi mcp add docs --url https://example.com/mcp --bearer-token-env-var DOCS_TOKEN
pi mcp list
```

这些命令默认添加用户级服务器。添加 `--local` 或 `-l`，则改为写入项目配置：

```bash
pi mcp add -l tools --env API_KEY='${TOOLS_KEY}' -- uvx tools-mcp
```

在交互式会话中使用 `/mcp`，可以检查连接、登录、重新连接、更改暴露方式，或启用和禁用服务器。如果在会话外添加、移除或修改了服务器，请运行 `/reload`。

<a id="configure-servers"></a>

## 配置服务器

Pi 从 `~/.pi/agent/mcp.json` 读取用户级服务器，从 `.pi/mcp.json` 读取项目服务器。只有授予[项目信任](security.md#understand-project-trust)后，才会读取项目配置。项目条目会替换同名的用户级条目。

如果项目条目没有 `command`、`url` 或 `type`，它只覆盖同名用户级服务器的 `enabled`、`exposure` 和 `toolExposure`，其余内容（包括 `env`、`headers` 和 `auth`）保持不变。例如，以下配置只在一个项目中关闭某个用户级服务器：

```json
{
  "mcpServers": {
    "internal-tools": { "enabled": false }
  }
}
```

配置格式与其他 MCP client 一致：

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "."]
    },
    "docs": {
      "url": "https://example.com/mcp",
      "headers": { "Authorization": "Bearer ${DOCS_TOKEN}" },
      "description": "Search and read the product documentation"
    }
  }
}
```

Stdio 服务器使用 `command`、`args`、`env` 和 `cwd`。相对的 `cwd` 值以会话目录为基准解析。`command`、参数或 `cwd` 中开头的 `~/` 表示用户主目录。

HTTP 服务器使用 `url`、`headers` 和 `oauth`（见[使用 OAuth 认证](#authenticate-with-oauth)）。不支持旧版 SSE transport。

两种服务器类型都支持：

- `timeout`：每个请求的超时秒数（默认 60）。进度通知会重置此计时。
- `enabled: false`：保留配置条目，但不连接该服务器。
- `exposure` 和 `toolExposure`：控制模型通过何种方式使用工具（见[控制工具暴露方式](#control-tool-exposure)）。
- `description`：用一句话说明服务器提供的能力。它用于在 system prompt 中列出服务器（见[控制工具暴露方式](#control-tool-exposure)），工具搜索根据它对服务器工具进行排序，codemode 的 `describeNamespace()` 也会返回它。未配置时，连接成功后会使用服务器指令的第一行。

个人服务器和携带凭据的服务器应保存在用户级文件中。项目文件只用于项目所必需的服务器，并且只应在受信任的项目中使用。

<a id="configuration-rules"></a>

### 配置规则

- 服务器名称只能包含字母、数字、`_` 和 `-`。工具命名为 `mcp__<server>__<tool>`，其中除字母、数字和 `_` 以外的字符都会替换为 `_`；如果同一服务器的工具名称经转换后发生冲突，则这些工具都会获得哈希后缀。仅在 `-` 和 `_` 上不同的服务器名称视为同一服务器：第二个会被拒绝，`mcp.json` 中的服务器会覆盖通过扩展注册的服务器。
- `type` 可选。存在 `command` 时选择 stdio，存在 `url` 时选择 Streamable HTTP。如果显式设置，`type` 必须是 `stdio`、`http` 或 `streamable-http`。
- 不接受 `sse`。文档中提供 SSE 端点的服务器通常也提供 Streamable HTTP，常见路径是 `/mcp`，而不是 `/sse`。
- `command` 是一个可执行文件，`args` 包含其参数。它不是一整条 shell 命令字符串。
- `env` 和 `headers` 的值可以引用环境变量，例如 `${GITHUB_TOKEN}`。也可以用 `!command` 执行命令，但命令必须占据整个值，例如 `"Authorization": "!echo Bearer $(gh auth token)"`。
- 无效条目会被报告并跳过，不会阻止其他服务器连接。

`pi mcp add` 和 `pi mcp remove` 可以在 shell 中完成常见修改。其选项见 [MCP 命令](cli.zh.md#mcp-commands)。

<a id="inspect-or-change-a-server"></a>

### 检查或修改服务器

`/mcp` 列出已配置服务器的状态、工具数量、暴露方式和配置来源。需要关注的服务器会排在前面。选择一个服务器后，可以检查其工具和连接详情、重新连接、登录或退出登录、更改暴露方式，或启用和禁用它。

暴露方式和启用状态的变更会保存到定义该服务器的文件中，不会替换无关内容。在受信任的项目中，“Enable in this project”和“Disable in this project”会为用户级服务器添加项目覆盖配置；之后对该服务器的修改会保存到覆盖配置中。禁用的服务器仍会显示在列表里。在交互式 TUI 以外，`/mcp` 会打印服务器状态；`/mcp login <server>`、`/mcp logout <server>` 和 `/mcp reconnect <server>` 则直接执行对应操作。

以下 shell 命令无需会话即可运行：`pi mcp add`、`pi mcp remove`、`pi mcp list`、`pi mcp login` 和 `pi mcp logout`。这些 shell 命令不会加载扩展。

<a id="diagnose-connection-problems"></a>

### 诊断连接问题

运行 `pi mcp list` 会连接每个已启用的服务器，并打印其状态、工具和错误。当有条目无效或有已启用的服务器未连接时，命令以状态码 1 退出。`/mcp` 会显示完整连接错误，以及启动失败的 stdio 服务器的 stderr 末尾内容。

Pi 会在启动后统一报告一次配置错误、连接失败和需要登录的情况。服务器日志通知会按 `<time> [<server>] <level> <logger>: <message>` 格式追加到 `~/.pi/agent/mcp.log`。文件超过 5 MB 后会轮转为 `mcp.log.1`。

会话启动时，Pi 在后台连接每个已启用的服务器。服务器连接成功后，其工具才会出现；`codemode` 的描述不列出这些工具，因此不会随着服务器连接而变化。首次 prompt 只会为带有 `direct` 工具的服务器等待最多 10 秒，因为这些工具必须在请求中声明。其他服务器在被需要时才等待：codemode 脚本会等待它指名的服务器（`mcp__<server>`）；调用 `searchTools()` 或读取 `ALL_TOOLS` 时，则等待全部服务器；`tool_search` 和资源工具也会等待全部服务器。HTTP 网络错误和临时状态码（408、429 和 5xx）会重试两次。连接断开后会显示为未连接，并在下一次调用时重新连接。服务器通知工具列表变更时，新工具会被加入，被撤回的工具将无法再调用。

停止 stdio 服务器时，会先关闭其 stdin，再发送 SIGTERM，最后向其进程组发送 SIGKILL。这也会停止通过 `npx` 或 `uvx` 等包装程序启动的服务器。

<a id="migrate-configuration-from-another-client"></a>

## 从其他客户端迁移配置

将转换后的条目放入 `mcp.json` 的 `mcpServers` 中，然后运行 `pi mcp list` 验证。

| 客户端 | 转换方式 |
|---|---|
| Claude Desktop、Claude Code 或 Cursor | 复制现有的 `mcpServers` 条目。 |
| VS Code | 将条目从顶层 `servers` 对象移入，并将 `${input:...}` 提示替换为 `${NAME}` 环境变量。 |
| Codex | 将 `[mcp_servers.<name>]` 下的 `command`、`args`、`env` 和 `url` 等 TOML 字段转换为 JSON。 |
| OpenCode | 将 `"type": "local"` 转换为 stdio 条目，将 `command` 数组拆为 `command` 和 `args`，将 `environment` 重命名为 `env`，并将 `{env:NAME}` 替换为 `${NAME}`。将 `"type": "remote"` 转换为 URL 条目。 |

<a id="authenticate-with-oauth"></a>

## 使用 OAuth 认证

使用 OAuth 的远程服务器（例如 Sentry）不需要在 `mcp.json` 中配置凭据：

```json
{
  "mcpServers": {
    "sentry": { "url": "https://mcp.sentry.dev/mcp" }
  }
}
```

当服务器拒绝未认证连接时，`/mcp` 会显示需要登录。选择“Sign in”，运行 `/mcp login sentry`，或运行 `pi mcp login sentry`。Pi 会打开授权页面并等待批准。如果浏览器在另一台机器上运行（例如通过 SSH 使用 Pi），请将授权后的重定向 URL 粘贴到登录界面。正在运行的会话会在下一轮使用新凭据。

Pi 向授权服务器注册自身，将 token 保存在 `~/.pi/agent/mcp-auth.json` 中，并在 access token 过期或被服务器拒绝时刷新。如果服务器之后要求额外的 scope，Pi 会再次请求登录。退出登录会删除已存储的凭据。

凭据归属于服务器名称和 URL。相同 URL 但名称不同的服务器（例如每个账号使用一个名称）分别登录；不同 `mcp.json` 文件中名称和 URL 相同的服务器共用一次登录。

OAuth 适用于没有 `Authorization` header 的 HTTP 服务器。对于不支持动态客户端注册的服务器，需要配置已注册的客户端：

```json
{
  "mcpServers": {
    "example": {
      "url": "https://mcp.example.com/mcp",
      "oauth": { "clientId": "my-client", "clientSecret": "${EXAMPLE_SECRET}", "callbackPort": 8765 }
    }
  }
}
```

重定向 URI 必须与注册的 URI 匹配。`callbackPort` 使用 `http://127.0.0.1:<port>/callback`。如需使用其他 URI，设置 `callbackUrl`；它必须使用 HTTP，并位于 `localhost`、`127.0.0.1` 或 `[::1]`。Pi 会按原样发送它。当 `callbackUrl` 没有指定端口时，Pi 使用 `callbackPort` 或空闲端口，并按照 RFC 8252 对 loopback 重定向的许可，将端口加入 URI。`clientSecret` 可选，可以引用环境变量或命令。

如果服务器没有公布所需的 scope，可以将 `scope` 设置为空格分隔的列表。否则，Pi 请求服务器公布的 scope。之后请求的 scope 会追加到配置值中。

Pi 以 `pi` 的名称注册。有些服务器只接受已知客户端的注册。设置 `clientName` 可以发送其他名称：

```json
{
  "mcpServers": {
    "figma": { "url": "https://mcp.figma.com/mcp", "oauth": { "clientName": "Claude Code" } }
  }
}
```

这个名称只在 Pi 注册客户端时发送。要以新名称重新注册，请先退出登录。

有些授权服务器根据客户端的 Client ID Metadata Document URL 接受客户端，而不是先进行注册。将 `clientRegistration` 设为 `cimd`，即可使用 pi.dev 上 Pi 的文档作为身份，而不进行注册：

```json
{
  "mcpServers": {
    "example": { "url": "https://mcp.example.com/mcp", "oauth": { "clientRegistration": "cimd" } }
  }
}
```

client ID 为 `https://pi.dev/oauth/client.json`，重定向 URI 为 `http://127.0.0.1:<port>/callback`。如果授权服务器没有在授权响应中发送 `iss` 参数（RFC 9207），Pi 会改用特定于该 MCP 服务器的文档和重定向路径：`https://pi.dev/oauth/<id>/client.json`，以及 `http://127.0.0.1:<port>/callback/<id>`。授权服务器必须公布对 Client ID Metadata Document 和公共客户端的支持，否则登录失败。`cimd` 不能与 `clientId` 或 `clientName` 组合使用；`callbackUrl` 必须使用 `localhost` 或 `127.0.0.1`，且路径为 `/callback`。

Pi 通过服务器的 protected resource metadata（RFC 9728）发现授权服务器，并检查授权服务器 metadata 中指定的 issuer 是否符合预期（RFC 8414）。有些服务器公布了错误的授权服务器，或根本没有公布，导致登录打开不存在的页面。设置 `authServerMetadataUrl`，即可指定正确授权服务器的 metadata 文档：

```json
{
  "mcpServers": {
    "example": {
      "url": "https://mcp.example.com/mcp",
      "oauth": { "authServerMetadataUrl": "https://example.okta.com/.well-known/openid-configuration" }
    }
  }
}
```

Pi 会使用并信任配置的文档，不再进行发现，因此只应指向你信任的文档。URL 必须使用 HTTPS，除非主机是 `localhost`、`127.0.0.1` 或 `[::1]`。

<a id="control-tool-exposure"></a>

## 控制工具暴露方式

服务器的每个工具都注册为 `mcp__<server>__<tool>`。服务器的 `exposure` 决定模型通过何种方式使用它：

| 暴露方式 | 行为 | 典型用途 |
|---|---|---|
| `codemode`（默认） | 可以从 [`codemode`](cli.zh.md#tools) 脚本中调用，但既不向模型声明，也不在 codemode 描述中列出。脚本通过 `searchTools()`、`describeTool()` 或 `ALL_TOOLS` 查找工具。 | 通用 MCP 服务器，尤其适合脚本需要组合或筛选调用的场景。 |
| `deferred` | 直到 [`tool_search`](cli.zh.md#tools) 加载了匹配工具，才在下一次模型调用中声明。 | 工具很多、需要先发现再直接调用的服务器。 |
| `direct` | 像内置工具一样向模型声明，同时也能从 codemode 调用。 | 规模小、经常使用的工具集。 |
| `hidden` | 已注册但不可调用。 | 应保持不可用的服务器或工具。 |

`codemode-deferred` 可作为 `codemode` 的别名。

带有 `codemode` 或 `deferred` 工具的服务器会列在 system prompt 的 `mcp_servers` 分节中，说明如何使用其工具，并附上配置的 `description`；如果没有配置且已经连接，则使用服务器指令的一行摘要。Pi 在 prompt 开始时，等待带有 `direct` 工具的服务器后更新此分节。如果分节发生变化，例如某个服务器连接成功并提供了摘要，Pi 会将新分节追加到对话中，而不是修改工具声明，从而让较早消息继续命中缓存。`describeNamespace()` 和 `searchTools()` 的 `namespace` 选项接受 `mcp__dev-radius`、`mcp__dev_radius`、`dev-radius` 或 `dev_radius`。

带有 `codemode` 暴露方式的服务器连接成功时，Pi 会激活 `codemode`。对于带有 `deferred` 暴露方式的服务器，Pi 会激活 `tool_search`。如果希望模型无需搜索就能看到某个工具，请通过 `toolExposure` 将它设为 `direct`。

`toolExposure` 可以为单个工具覆盖服务器的暴露方式。键可以是服务器工具的精确名称，也可以是用 `*` 匹配任意字符的模式。精确名称优先于模式；多个模式中以第一个匹配项为准。服务器即使使用 `hidden` 暴露方式，也可以只暴露选定的工具：

```json
{
  "mcpServers": {
    "github": {
      "url": "https://api.githubcopilot.com/mcp/",
      "exposure": "deferred",
      "toolExposure": {
        "search_code": "direct",
        "get_*": "codemode",
        "delete_*": "hidden"
      }
    }
  }
}
```

`pi mcp list` 会标记暴露方式与其服务器不同的工具。`/mcp` 的 Tools 视图也会显示实际生效的暴露方式。

具有 `codemode` 或 `deferred` 暴露方式的工具，可以通过这两种间接机制中的任一种使用：codemode 脚本可以调用它们，`tool_search` 也可以加载它们。Codemode 调用不依赖当前活动工具集，因此在 `/tree`、恢复会话和 fork 之后仍然可用。由 `tool_search` 加载的工具会记录在对话历史中，并在该分支上持续保持声明。

除非 `--tools` 的某个条目以 `mcp__` 开头，否则它不会移除 MCP 工具；`pi --tools read,codemode,'mcp__radius__*'` 只保留 `radius` 的工具。`--exclude-tools` 接受相同模式，`--no-mcp` 则为本次运行禁用 MCP（见[工具](cli.zh.md#mcp-tools)）。

要在没有 MCP 服务器时仍保持 `codemode` 激活，请在[设置](settings.md#tools)中添加 `"defaultTools": ["+codemode"]`。要阻止自动激活 codemode，请在 `mcpServers` 旁设置 `"autoEnableCodemode": false`。项目值覆盖用户级值。如果 `codemode` 和 `tool_search` 都未激活，导致非 direct 工具无法调用，Pi 会发出一次警告。

超过 20 KB 的文本结果会移除中间部分，并在移除处插入 `…N chars truncated…` 标记后发送给模型。完整文本保存到临时文件中，结果会给出文件名。Codemode 脚本收到完整结果，可以先精简再返回给模型。

Codemode 脚本收到完整的 MCP `CallToolResult`，包括 `content`、`structuredContent` 和 `isError`。带有 `isError` 的结果在脚本内仍正常兑现，但直接调用时会报告为错误。`image(result.content[0])` 用于转发图片块。服务器指令不属于任何工具描述；脚本可以通过 `describeNamespace("mcp__<server>")` 读取它，同时获得该服务器的工具名称。

<a id="use-resources"></a>

## 使用资源

当已连接的服务器提供[资源](https://modelcontextprotocol.io/specification/2025-11-25/server/resources)时，Pi 会添加 Codex 和 OpenCode 所使用的资源工具：

- `list_mcp_resources` 以 JSON 形式列出资源：`{ server?, resources: [{ server, uri, name, ... }], nextCursor? }`。指定 `server` 时列出一页，使用 `cursor` 继续读取下一页。未指定 `server` 时，列出所有服务器的全部资源。
- `list_mcp_resource_templates` 列出用于访问服务器未直接列出的资源的 URI template。
- `read_mcp_resource` 根据 `server` 和 `uri` 读取资源。文本以文本形式发送给模型，图片以图片形式发送。其他二进制资源会保存到临时文件中，模型接收文件路径。脚本收到 `{ server, uri, contents }`。

这些工具可以访问所有已启用、非 hidden 且提供资源的服务器。其暴露方式取这些服务器中范围最广的一种：先是 `direct`，再是 `codemode` 或 `deferred`。工具结果中的资源链接会注明 `read_mcp_resource` 和服务器。

Pi 不会渲染 MCP Apps，因此会省略由 `ui://` URI 或 `text/html;profile=mcp-app` 标识的相关资源。资源图标也会省略。

读取和列出资源遇到临时 HTTP 错误（408、429 或 5xx）时会重试一次。工具调用不会重试，因为服务器可能已经执行了操作。

<a id="permissions"></a>

## 权限

每个 MCP 调用都会经过 Pi 的工具处理管线。因此，扩展中的 `tool_call` 和 `tool_result` 处理器（包括权限门控）同样作用于 MCP 工具。从 codemode 脚本发起的调用，会将 codemode 的调用 ID 作为 `parentToolCallId`。

`pi.getAllTools()` 会报告每个服务器声明的 annotation：`readOnlyHint`、`destructiveHint`、`idempotentHint` 和 `openWorldHint`。权限扩展可以据此判断哪些调用需要确认（见[工具暴露方式](extensions.zh.md#tool-exposure)）。资源工具标记为只读。

<a id="extensions-and-sdk"></a>

## 扩展与 SDK

<a id="add-servers-from-extensions"></a>

### 从扩展添加服务器

扩展可以通过 `pi.registerMcpServer(name, config)` 为当前会话添加服务器，配置结构与 `mcpServers` 条目相同（见[扩展中的 MCP 服务器](extensions.zh.md#mcp-servers)）。通过扩展注册的服务器与配置文件中的服务器一样进行连接，并在 `/mcp` 中将该扩展显示为来源。

对启用状态或暴露方式的更改只作用于当前会话。配置文件中的同名服务器优先，`/mcp` 会列出被覆盖的注册项。`pi mcp` shell 命令不会加载扩展，只能看到通过文件配置的服务器。

<a id="replace-the-built-in-mcp-support"></a>

### 替换内置 MCP 支持

如果安装的扩展注册了 `/mcp`（例如 `pi-mcp-adapter`），它就会在会话中替换内置 MCP 支持。此时，Pi 在会话中不再读取 `mcp.json` 或连接其中的服务器，`/mcp` 由该扩展提供。移除扩展即可恢复内置行为。若不安装替代扩展而只想关闭内置 MCP 支持，可以在 `pi config` 的 Built-in 中禁用 `mcp`，或在[设置](settings.md#resources)中配置 `"extensions": ["-builtin:mcp"]`。`--no-mcp` 只为本次运行禁用它。

注册 `codemode` 或 `tool_search` 的扩展也会替换同名内置工具。只有替代工具声明了 [`reachesTools`](extensions.zh.md#tool-exposure)，Pi 才会为 MCP 服务器激活它，并等待其调用所需的服务器。Shell 层的 `pi mcp` 命令始终使用内置实现。

<a id="use-mcp-from-the-sdk"></a>

### 从 SDK 使用 MCP

SDK 会话不会加载内置扩展。需要向 ResourceLoader 添加 MCP 扩展；对于 `codemode` 服务器，还需添加 codemode 扩展；对于 `deferred` 服务器，还需添加 tool-search 扩展。参阅 [Codemode 与 MCP](sdk.md#codemode-mcp)。
