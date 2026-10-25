> 中文译文：[英文原文](index.md)。译文可能滞后于原文，技术细节以英文原文和当前源码为准；代码示例保持原样。下文已翻译的页面链接到 `.zh.md`，其余链接仍指向英文文档。

<a id="pi"></a>

# Pi

Pi 是一个在终端中工作的可扩展 AI Agent。给它一个目标和工作目录，它就可以检查文件、运行命令、编辑内容，并处理多步骤任务。

你可以用 Pi 进行软件开发、整理研究笔记、写作、处理数据文件或开展个人项目。既可以直接使用 Pi，也可以通过 prompt 让它适应你的工作流，或者利用 SDK 构建由 Pi 驱动的其他应用。

<a id="start-using-pi"></a>

## 开始使用 Pi

第一次使用 Pi？按照[快速入门](quickstart.md)安装 Pi、连接模型，并完成第一个任务。

如果已经安装 Pi，可以根据目标选择：

- [交互式使用 Pi](usage.md)：添加文件、运行命令、指导正在进行的工作，以及导出结果。
- [选择模型](models.md)：连接订阅、API key、本地模型或兼容端点。
- [继续会话或创建分支](sessions.md)：恢复工作，或在不丢失历史记录的情况下探索另一种方案。
- [配置 Pi](configuration.md)：设置个人偏好、工作目录、指令和可复用资源。
- [理解 Pi 的工作原理](how-pi-works.zh.md)：了解工具、上下文、会话和 Agent 循环。

<a id="customize-pi"></a>

## 定制 Pi

Pi 可以复用 prompt、加载专用指令、添加可执行集成、更改终端界面、连接模型服务，并将这些资源打包分发。
使用[快速入门中的定制方式选择指南](quickstart.md#choose-how-to-customize-pi)，选择满足需求的最小机制。

<a id="automate-or-embed-pi"></a>

## 自动化或嵌入 Pi

- 使用 [Print 模式](cli.zh.md#invocation-and-output)处理一次性任务和脚本任务。
- 使用 [JSON 事件流模式](json.md)接收一次运行中的结构化事件。
- 使用 [RPC 模式](rpc.md)控制独立的 Pi 进程。
- 使用 [TypeScript SDK](sdk.md)在应用内部运行 Pi。

<a id="find-reference-and-setup-information"></a>

## 查阅参考与配置资料

通过参考页面查阅 [CLI 选项](cli.zh.md)、[设置](settings.md)、[Provider](providers.md)、[快捷键绑定](keybindings.md)和[环境变量](environment-variables.md)。

平台相关帮助请参阅[终端配置](terminal-setup.md)、[Windows](windows.md)、[tmux](tmux.md)、[Android 上的 Termux](termux.md)，或[容器化](containerization.md)。

<a id="work-safely"></a>

## 安全使用

Pi 的工具和扩展使用 Pi 进程的权限运行。项目信任控制 Pi 加载哪些项目资源，但不会对工具调用进行沙箱隔离。在使用不可信文件、仓库、扩展或无人值守自动化之前，请阅读[安全说明](security.md)。
