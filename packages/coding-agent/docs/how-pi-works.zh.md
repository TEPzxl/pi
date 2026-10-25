> 中文译文：[英文原文](how-pi-works.md)。译文可能滞后于原文，技术细节以英文原文和当前源码为准；代码示例保持原样。

<a id="how-pi-works"></a>

# Pi 的工作原理

Pi 协调模型请求、工具执行、上下文组装和会话存储。会话（session）是 Pi 对一次对话的记录，包含消息、工具调用及其结果、模型变更、上下文压缩和其他事件。

会话中的消息和事件构成一棵树。树中的每条路径都是一个分支。以当前条目结尾的分支是活动分支，它为下一次模型请求提供历史记录。

<a id="agent-loop"></a>

## Agent 循环

提交的消息会添加到活动分支。Pi 根据 system prompt、活动分支、可用工具和模型设置构造模型请求，再通过选定的 Provider 发送。

Provider 以流式方式返回 assistant 响应，其中可以包含文本和工具调用。Pi 记录响应，执行每个工具调用，并记录结果。这就完成了一个 turn。如果工具结果或排队的消息需要再次请求模型，Pi 就开始下一个 turn；否则，本次运行（run）结束。

Steering 消息在当前 assistant turn 结束后进入流程。Follow-up 消息在 Agent 完成待处理工作后进入流程。中止操作会停止当前运行，并将排队的消息返回编辑器。

<a id="context"></a>

## 上下文

活动分支提供对话历史。Pi 将会话条目转换为模型兼容的 user、assistant 和 tool-result 消息。

Pi 根据基础指令和发现的上下文文件构建 system prompt。请求还携带工具定义和 Skill 描述。

完整的 Skill 指令按需加载。Extension 可以添加指令或转换上下文。

Prompt template 会在编辑器输入成为用户消息之前将其展开。选中的文件、图片、粘贴的文本和 shell 输出都可以成为消息内容。

<a id="sessions"></a>

## 会话

持久化会话保存为 JSONL 文件。树中的每个条目都有一个 ID，并引用其父条目。当前条目标识活动分支。

从较早的条目继续对话，会在同一文件中创建另一个分支。Fork 和 clone 则会将选定的历史记录复制到新的会话文件。

模型上下文根据活动分支重建。上下文压缩会插入一个摘要条目，在后续模型请求中替代较早的消息。原始条目仍保留在会话树中。

<a id="interfaces"></a>

## 交互接口

Interactive 模式在终端中渲染会话和 Agent 事件。Print 模式运行一个 prompt 并输出最终响应。JSON 模式将 Agent 事件输出为 JSONL。

RPC 模式从 stdin 接收 JSONL 命令，并将响应和事件写入 stdout。TypeScript SDK 在进程内创建和控制 Agent 会话。

所有接口都使用相同的 Agent 和会话机制。

<a id="extensions-and-resources"></a>

## 扩展与资源

Extension 是加载到 Pi 进程中的 TypeScript 模块。它们的工厂函数可以注册工具、命令、快捷键、Provider、事件处理器、渲染器和终端 UI。

Skill 提供按需加载的指令及配套文件。Prompt template 提供可复用的消息文本。Theme 提供终端颜色配置。Pi package 通过 npm 或 git 分发这些资源。

<a id="trust-and-permissions"></a>

## 信任与权限

Pi 在加载项目设置和资源之前，先确定项目信任状态。完成信任决策和项目资源加载后，Pi 再加载上下文文件。启用的工具使用 Pi 进程的操作系统权限。Extension 也在该进程中执行。
