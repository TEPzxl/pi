# Pi 源码学习路线

最近记录日期：2026-10-25。

本文用于保存已约定的学习路线，并在新会话或上下文压缩后恢复进度。它是学习指南，不是修改源码、安装依赖、运行测试或调用真实模型的授权。普通开发任务不必按本路线执行。

## 当前进度（Current Progress）

- **当前阶段：R7 已首次运行原有指定 mock 测试，Vitest 报告通过；外层终端退出异常仍在。** 用户已凭印象复述工具调用主线，并完成结果配对、失败位置、下一次模型请求输入的检查设计。复述中把工具查找键记成调用 ID，已对照 prepareToolCall 纠正为 name；文件读取细节由助教补全，不能记为完整源码独立追踪已通过。2026-10-25 运行原 should handle tool calls and results，Vitest 输出 1 passed / 37 skipped；随后外层终端报 TTY 错误，整条命令退出码 2，不能记录为正常退出。未修改源码或测试、未安装依赖或调用真实模型；不是整条路线已全部验收。
- **已有基础：**对 `ai`、`agent`、`coding-agent`、`tui` 四个核心包有初步认识；[how-pi-works.zh.md](packages/coding-agent/docs/how-pi-works.zh.md) 已读完。阅读完成不等于 R0 独立验收通过。
- **已独立核对的理解：**R1–R4 已有问答依据保留，含 single 优先、首次工具声明不变、调用 ID 关联、执行等待当前完整 assistant 响应。R5 新增：①用户复述 Agent 能执行工具，在当前 Pi 应用中工具及环境由 AgentSession 配置；已补充 Agent 不依赖必须存在 AgentSession。②用户为“完成 README 总结后检查 package.json”正确选择 followUp，意图是不干预当前总结；已澄清 steer 也不立即中断工具。③用户正确解释 UI 的 tool_execution_end 分支更新组件、移除待处理追踪；已澄清删除的是 pendingTools 记录，不是移除注册工具或结果组件。④用户正确解释一条 assistant 消息只在 message_end 时调用一次 appendMessage，不逐次保存中间更新。
- **R6 新增独立核对：**①用户解释 id / parentId 连接树，并正确追踪 getBranch 从当前节点沿父链回溯、reverse 后得到根到当前节点的路径，兄弟分支不返回。②正确预测当前分支 low 不受另一分支 high 影响，path 指当前祖先路径。③正确解释 model_change 是设置条目，不因此新增聊天消息；已补充由设置恢复与消息转换两个函数分工。④正确预测另一分支上的 compaction 不作用于当前 m1→m2→m6 路径。⑤正确判断首次写入完成后，新增配置条目仍走 else 追加，不必等待新聊天消息。⑥正确计算新增 e5 的 parentId=e2、重开 leafId=e5。
- **R7 新增问答依据：**①用户主线复述覆盖 function_call → Pi toolCall → execute → 结果消息 → Provider 格式，关键查找字段和消息/内容块名称已按源码校准。②用户提出按调用 ID 找对应结果并检查 content，方向正确；Pi 字段为 toolCallId，助教补明还需限制 role=toolResult 并比较预期文本，不只是有内容。③用户用 executed 是否为空区分准备阶段失败与进入工具后失败，核心正确；精确 [] / [README.md] 与记录必须在入口第一步的前提由助教补充。④最新用户正确提出在 mock 的第二次调用检查上下文是否含上次执行结果；助教补明第二个实参 llmContext.messages，以及 role、toolCallId、content 检查。这些都是静态设计，没有新增或运行对应断言。
- **已讲解但仍待巩固：**R4 的 Provider 查找键、name/id、model.api/input.api、实现选择与认证等纠正不等于全量掌握；工具结果改变会影响第二次模型请求输入曾答错；R7 中已能独立定位到第二次 mock 调用的上下文输入检查，说明回传检查位置已有理解，但尚未实际运行内容变化的反事实，mock 固定 done 仍不能证明输入正确。R5 中用户曾把所有外部操作归给 AgentSession、把 Agent 当成仅生成消息，已用 loop → execute → readFile 纠正；默认真实工具不会因模型是 mock 而自动隔离 I/O，尚未独立验证。ReadToolOptions.operations 是替换点，但 resolveReadPathAsync 仍直接检查本地路径，不能承诺零文件系统访问；可用假 read 工具隔离该次工具执行。UI 不显示与模型上下文缺结果不是同一件事，结论已讲解、尚未独立完整说明。R3 取消收尾等保留待巩固。R6 中用户曾把仅 branch(e2) 改内存误认为重开仍是 e2，已用文件未变、_buildIndex 取最后非会话头条目纠正；后续有新增 e5 的题已答对，但不能把无新增的原题也记为独立答对。首次创建文件的门槛与缓冲写入已讲解，不等于全部异常/退出情形已验证；这些待巩固场景未实测，未调用真实模型。
- **本阶段源码依据：**agent-loop.test.ts:305–399 是原最小工具测试；357–375 的 mock 只按 callIndex 回应，不读取模型输入；384–398 仅检查副作用、事件/错误标记和 usage，没有新增的 content / 第二次输入断言。agent-loop.ts:716 按 toolCall.name 查找，726–727 校验参数，832–839 调用 execute，845–852 将抛错包装为错误结果；594–611 显示工具开始事件可早于查找、失败也有结束事件。274–276 分别写 currentContext.messages 与 newMessages，388–407 将上下文转换后作为第二个参数传入 streamFunction。read.ts:124、155 是实际路径解析与读取；对外协议转换、UI 和存储仍按前阶段已读路径，不重新遍历全部实现。
- **下一步：**原指定测试已按用户授权尝试运行。建议在用户另行确认后，仅增强该测试的结果 content / toolCallId 和第二次 mock 请求输入检查，再运行同一指定测试；当前没有修改测试、安装依赖、构建或真实模型调用的授权。外层 TTY 退出异常可由用户在自己的终端用同一命令复核，不擅自修改 shell 配置。保留 UI/存储整合与旧待巩固项，不继续无限拆字段小题。
- **验证范围与证据边界：**原测试保持未修改；本次 Vitest 通过输出只支持原有副作用、事件、错误标记和 usage 断言。按 ID 校验 content、区分失败阶段、检查第二次 mock 输入均只是已讨论的改进方案，没有加入或运行；不能由原测试通过推断这些检查已验证。即便后续加入 mock 输入断言，也只覆盖模型响应函数的入参，不证明真实 HTTP 或服务端处理。完整源码追踪、测试输出、命令退出状态和 Provider 集成验证分别记录。
- **教学与记录偏好：**先用当前源码的实际数据或请求例子解释，不只讲抽象术语；问题给出具体场景、范围和期待回答，不用“展开一行代码”等不清楚的要求。保留提问和短预习，但澄清问题时不机械叠加新题。按用户最新明确要求，只在大阶段结束或准备压缩、交接时集中更新 roadmap，不逐条消息写文件或为此反复运行工具；同一连续会话也无需每轮重读本文。
- **测试环境与退出异常：**旧 Vitest CLI 缺失阻塞已解除：本次预检确认 CLI、typebox、pi-ai workspace link 存在，Node 为 v24.14.0；实际启动 Vitest v4.1.11，指定文件 1 passed，测试 1 passed / 37 skipped（共 38）。随后 `/bin/sh: 2: Cannot set tty process group (No such process)` 导致外层命令退出码 2；Git 状态命令也出现相同尾部错误。仅确认测试通过输出，未确认整个命令正常退出，未排查或修改终端配置。本会话未安装依赖。
- **沟通方式：**默认中文，技术标识保留英文；先回答当前问题，不因一个追问连续下钻多个抽象层。用户已指出提问和预习长期缺失，当前恢复轻量互动；此前暂停是助教基于过载反馈的临时调整，不是用户要求永久取消。再次过载时临时减量，并说明恢复节点，不能无限沿用暂停。可用辅助对话查局部内容，仅主对话维护进度。

### 进度记录

只记录有依据的进度。下表日期是本次落盘日期，不推定此前教学发生的日期。

| 日期 | 阶段 | 已确认 | 待验证 / 阻塞 | 下一步 |
| --- | --- | --- | --- | --- |
| 2026-10-24 | R1 进行中、未验收；R0 已有基础 | 四包有初步认识；`how-pi-works` 已读；能概括测试骨架 | `hello` 值链、事件结果区别、`afterToolCall` 尚待核对；Vitest 入口缺失，安装及成功运行未确认 | 只核对 `hello` 从 mock 参数进入 `executed` 的过程，通过后再学 `afterToolCall` |
| 2026-10-24 | R1：`hello` 值链已核对，阶段仍未验收 | 用户已解释 mock 参数 `hello` 经 `params.value` 被推入 `executed` | 数组副作用与工具返回值的区别已澄清、待后续核对；事件区别与 `afterToolCall` 待核对；测试运行状态不变 | 追踪原始 usage、回调读取及最终 usage 补丁，解释两处 usage 断言为何不同 |
| 2026-10-24 | R1：回调执行时机与两行作用已有基本理解 | 用户说明工具调用结束后执行回调，先记录观察到的用量，再返回修改 | 已澄清返回的是 usage 补丁而非完整结果；缺少补丁时的行为仍待用户推演；测试运行状态不变 | 只做“保留记录、移除返回修改项”的推演，然后学习事件与消息 |
| 2026-10-24 | R1：无 usage 补丁的推演已核对，阶段仍未验收 | 用户正确预测最终保持 `toolUsage`，并指出没有应用新 usage | 用户追问局部更新为何生效；已查证调用方保留原结果并应用补丁，不是语言自动行为；事件与消息待核对，测试运行状态不变 | 先回答当前疑问，无追问后区分工具开始、结束事件 |
| 2026-10-24 | R1：事件存在与错误标记检查已有场景核对 | 用户指出 `isError === true` 时最后一个错误标记断言失败；未把事件存在当作检查未出错 | 最终消息与过程事件的数据区别、结果文本断言缺口和两次模拟响应理由仍待核对；未运行测试 | 阅读消息结果，只推演“文本错误但其他已检查字段不变”能否被测试发现 |
| 2026-10-24 | R1：工具结果文本的断言缺口已核对，阶段仍未验收 | 用户说明按 role 找到结果后只比较 usage，故仅文本错误不会被当前断言发现 | 已澄清 usage 属于消息而不是 role；两次模型响应理由及消息顺序仍待解释；未运行测试 | 只看 mock 两个分支，解释工具输出后为什么还有 assistant 回复 |
| 2026-10-24 | R1：两次模拟响应的主线理由已核对，待收尾 | 用户解释工具输出不等于最终回复，需要模型处理工具结果后给用户回复 | 已补充不一定只是总结、也不固定两次；消息角色顺序与事件归类待串联；测试运行未确认 | 列出本场景消息顺序并区分事件，核对后再进入 R2 |
| 2026-10-24 | R1：事件归类正确，消息角色需纠正 | 用户把工具开始、结束归类为 event | 将工具结果误写为 assistant 并漏掉最终回复；助教另查明此前角色范围漏了自动插入的 system，已纠正教学依据；未运行测试 | 巩固消息角色；默认附短预习，允许辅助对话只读答疑，不共同写进度 |
| 2026-10-24 | R2 入口预习已核对；R1 角色收尾仍待完成 | 用户正确指出 `agentLoop` 委派给 `runAgentLoop`，返回 stream | 返回类型精确为 `EventStream<AgentEvent, AgentMessage[]>`；两个读取接口已讲解、未独立核对；测试运行未确认 | 只核对工具结果消息的 role；预习 `runAgentLoop()` 102–126 行 |
| 2026-10-24 | 定位到角色误解的类型根因；R2 入口预习继续 | 用户已定位 `runLoop` 和返回 `newMessages`，主动追问数组原地修改；正确把测试 stream 与入口 stream 联系起来 | 将 `AgentMessage` 误当 `AssistantMessage`，并混淆读取结果与流收尾；已查证联合类型、工具消息构造和两处 push；未运行测试 | 先补联合类型与数组元素基础，不再重复猜 role；仅小范围预习两个类型别名 |
| 2026-10-24 | R1 基础静态阅读问答收尾，进入 R2 | 用户正确解释 `UserMessage` / `ToolResultMessage` → `Message` → `AgentMessage` 的允许关系，类型阻塞已澄清 | R2 全部循环尚未掌握；数组与上下文区别待核对；测试实际运行仍未确认 | 先区分 `newMessages` 与 `currentContext.messages`，预习工具调用分支的判断条件 |
| 2026-10-24 | R2：本次消息返回与正常执行条件已有回答 | 用户说明不重返旧历史，并识别 toolCall 与非 length 的实际执行路径 | 已补充 length 会生成失败结果而非直接忽略调用；查找、校验、执行准备待核对；未运行测试 | 追踪按名称找到 echo 工具，预习真正调用 execute 时参数的来源 |
| 2026-10-24 | R2：工具查找失败与参数执行链已核对 | 用户指出名称不匹配返回错误且不执行，并追踪 validatedArgs 经 preparation / prepared 传给 execute 第二参 | 原始返回值、finalized、事件与消息构造的关系以及循环继续条件待核对；未运行测试 | 追踪结果包装，预习 hasMoreToolCalls 与内层 while 的关系 |
| 2026-10-24 | R2：内层继续条件已有回答 | 用户指出 hasMoreToolCalls 与其他待处理消息两个条件；题设无排队时前者起作用 | R1 两次响应对应的 true/false 变化、自然结束路径及 turn/run 已讲解待巩固；未运行测试 | 预习 streamAssistantResponse 请求前的消息转换链 |
| 2026-10-24 | R2：请求消息转换主线基本追到 | 用户逆向追出 llmContext、规范化、模型消息转换和 context.messages | 数组/单条消息、可选转换与 signal 的数据/控制区分已澄清待巩固；未运行测试 | 用正向链明确各步输入输出，预习 partial 与最终消息的替换逻辑 |
| 2026-10-24 | R2：流式占位的直觉部分正确 | 用户识别 start 插入 partial、内容逐步增长 | 将本层更新猜成拼接两条消息；需区分 delta、当前消息状态与数组元素替换；未运行测试 | 通过短文本示意澄清，回看 R1 done-only mock 对 addedPartial 的影响 |
| 2026-10-24 | R2：无 start 时的追加分支已核对 | 用户指出 mock 无 start，最终追加消息，并主动提问同名 type 的层次 | 模型事件 type、消息 role、内容块 type 与内外 stream 需要区分；未运行测试 | 展开字段路径，预习首轮 done 携带普通文本时是否执行工具 |
| 2026-10-24 | R2 基础路径收尾，R3 入口预习 | 用户正确指出纯文本使 toolCalls=[]，分支不成立，不执行 echo，并定位到具体内容块检查 | 已补齐自然停止的源码推导；R2 全部变体并未验收，测试实际运行仍未确认 | 对比 Agent.prompt 与 agentLoop 的返回类型及委派方法 |
| 2026-10-24 | R3：入口返回值与委派已核对 | 用户指出 prompt 返回 Promise<void> 并交给 runPromptMessages | 运行完成、state.messages 与 subscribe 的分工已查证，尚未独立解释；未运行测试 | 追踪事件更新状态与通知监听器的顺序 |
| 2026-10-24 | R3：找到 push 并判断先更新状态 | 用户定位 message_end 的 push 并推断先保存，结论正确 | 当时未查明具体疑问，助教误转去讲基础回调语法；未运行测试 | 应先定位真实 subscribe 调用方，不能按未核实的语法缺口推进 |
| 2026-10-24 | R3：用户澄清缺调用上下文，修正教学方向 | 用户明确知道基本语法，实际没找到 subscribe 调用链；指出新方法尚未引入就谈注册 | 已查证 AgentSession 构造函数与测试中的真实调用点；完整连接尚待反馈；未运行测试 | 先说明方法用途与实际调用者，分开注册路径和运行通知路径，不讲基础语法或新章节 |
| 2026-10-24 | R3：用户要求继续，转入运行生命周期 | subscribe 实际调用路径已补充；用户允许继续，不等同全部订阅机制掌握 | runWithLifecycle/finishRun 与异步订阅者的完成边界刚开始讲解；未运行测试 | 先看开始/收尾的真实调用链，再预习等待异步订阅者的现有 mock 测试 |
| 2026-10-24 | R3：异步订阅者等待链与状态判断正确 | 用户沿多层 await 推导 prompt 未完成、isStreaming 仍 true | 需纠正已送达事件与分发尚未完成的区别；正常路径为 320 行而非错误分支 254；未运行测试 | 澄清分发边界，预习实际测试中 agent.abort 到 signal 接收方的链路 |
| 2026-10-24 | R3：取消信号的创建与订阅者支路已找到 | 用户找到 controller 实例化、signal 传 executor，以及 processEvents 把 signal 传 listener，并关联 subscribe | executor 实参身份遗忘、模型 options.signal 支路尚缺；已查 Node.js 24 官方契约与本地传参；未运行测试 | 对照两条同一 signal 支路，区分订阅者观察与 mock 主动响应取消 |
| 2026-10-24 | R3：用户指出取消说明缺少流完成实现 | 用户准确观察测试只有 push(error)，没有显式 end | 已查证子类的 done/error 完成策略、super.push 设置 done/结果及消费者返回；说明尚待用户核对，未运行测试 | 只补具体流实现，并复看 error 为什么仍被交付，不切入完整 ai/Provider 学习 |
| 2026-10-24 | R3：用户反馈复杂且晕，暂停下钻 | 没有新增掌握证据；已有参数追踪和 await 链的理解仍保留 | 取消与底层流实现连续跨层导致当前内容过密；底层细节只算已查证、待回看；未运行测试 | 不布置预习或题目，只用取消请求→工作方响应→Agent 收尾三步复盘 |
| 2026-10-24 | R3：用户自报理解取消前两步，追问第三步 | 仅确认用户对简化三步流程的前两步表示理解 | 第三步 await 返回后 finally/finishRun 的衔接待说明；不等于全取消机制掌握；未运行测试 | 只解释工作返回与状态复位，不添加预习或底层概念 |
| 2026-10-24 | R3：用户要求继续，单独引入 steer | 第三步收尾已讲解，用户允许下一小段；无新独立掌握证据 | 已查证最短外部调用、入队与正常循环消费点；steer 本身尚未核对理解；未运行测试 | 只讲运行中补消息的三步，不展开 followUp 或队列实现，不布置题目 |
| 2026-10-24 | R3：用户要求继续，比较 followUp 消费时机 | steer 正常路径已讲解，用户允许下一小段；无新独立掌握证据 | 已查证 followUp 外部测试调用、入队与内层退出后取出的位置；未运行测试 | 只区分调整当前工作与排后续任务，强调同一 run 内接续，不布置新题 |
| 2026-10-25 | R3：收束 Agent 职责与已有调用链 | 用户要求继续；本轮重新核对入口、事件处理、状态和收尾源码，无新增独立掌握证据 | 职责串联仅记已讲解；取消收尾、两种队列和阶段整体理解仍待反馈；未运行测试 | 等待用户反馈，保持 R3 进行中，不布置预习测问或自动跳 R4 |
| 2026-10-25 | R3：用户选择先用 echo 复习 | 用户明确先复习完整流程，未选择进入 R4；已核对原测试及对应循环、消息和工具结果实现 | 仅记录正常路径已讲解，无新增独立掌握证据；原测试未创建 Agent，Agent 状态路径为源码映射；未运行测试 | 等待具体反馈，不加取消或队列分支，不布置测问和预习 |
| 2026-10-25 | R4 入口：echo 复习后继续，介绍 streamFn 替换边界 | 用户要求继续；已核对调用点、StreamFn 类型、测试 mock 及 SDK 的真实接法 | 本段仅记已讲解，R3 未验收且待巩固；R4 尚未独立解释；未运行测试或请求模型 | 继续时只区分 model、provider、context；不扩展底层流或网络协议，不布置题目 |
| 2026-10-25 | R4：区分 model、provider 与 context | 用户要求继续；已核对测试模型描述及 ai 中相关类型，按 echo 两次请求作对照 | 仅记已讲解，尚无新增独立掌握证据；mock 不执行真实推理；未运行测试或请求模型 | 继续时沿一条路径看 provider 标识与接入实现的关联，不展开鉴权或网络，不布置题目 |
| 2026-10-25 | R4：Provider 标识查表与方法调用 | 用户要求继续；已核对 recomposeProvider 的存入调用、ModelsImpl 的 Map 读写及 prepareRequest → provider.streamSimple | 仅记已讲解，无新增独立掌握证据；原 echo mock 不经过真实接入路径；未运行测试或请求模型 | 继续时只选内置 OpenAI Provider 的对象来源与委派，不展开其他 provider、鉴权或 HTTP |
| 2026-10-25 | R4：用户要求恢复提问与预习，修正节奏 | 用户明确指出只有讲解、缺少问题与预习；已核实 openai.ts 短预习范围 | 本轮只恢复互动并提出理解问题，尚未作答；没有新增掌握或测试通过证据 | 先核对仅改 model.id 是否影响 Provider 查找，再讨论 openaiProvider；一题一小段，不连续堆新概念 |
| 2026-10-25 | R4：纠正两种 id 的混淆，预习改为行为追踪 | 用户定位到 api 工厂调用；明确反馈找字段的预习过浅 | 用户将 Provider 查找键误记为 model.id，已按源码纠正、待验证；未运行测试或请求模型 | 以 createProvider 的 apiFor / dispatch / streamSimple 追踪参数传递，一份任务兼作理解检查，不再出字段辨认题 |
| 2026-10-25 | R4：澄清 single / byApi 不是认证分支 | 用户提出两条路径对应 OAuth / API key 的猜想，并反馈不清楚题目期待什么；已核对配置判断、ProviderStreams 类型与 OpenAI 配置 | 猜想不正确，纠正仅为已讲解、待验证；卡点选项不是待答题，整链任务先搁置；未运行测试或请求模型 | 只用一个具体场景区分 API 实现配置与认证，答完再回方法委派，不增加独立预习 |
| 2026-10-25 | R4：多模型不等于多份 API 实现 | 用户提出同一 Provider 下多个模型共用 key 的例子，并以 name 描述模型选择；已核对请求中使用 model.id 及界面使用 name 的代码 | name/id、模型选择/协议实现选择已讲解，尚未独立验证；示意模型名不代表真实目录；未运行测试或请求模型 | 只推演保持 single 配置、修改 model.api 是否自动切换实现；题目与阅读合并，不展开方法整链 |
| 2026-10-25 | R4 主线收束，按用户要求集中记录 | 用户独立解释 single 优先、调用结果 ID 错配、首次工具声明不变、执行等待当前完整 assistant 响应；已用实际 Responses/Completions 调用及工具往返格式讲解 | 第二次请求包含变更后的工具结果曾答错，纠正后仍待独立核对；其他讲解不当成全量掌握；未运行测试或真实模型 | 预习 sdk.ts 的 Agent 与 AgentSession 组装，准备 R5；今后只在大阶段或压缩/交接时更新，优先实际例子与明确问题 |
| 2026-10-25 | R5 主线收束，集中记录 | 用户正确解释 Agent 执行/Session 配置工具、followUp 场景选择、UI 工具结束显示与追踪、单条完整消息只触发一次 appendMessage | 曾将全部 I/O 归 Session、认为 mock 模型隔离工具 I/O，均已纠正待巩固；operations 仍不覆盖路径存在性检查；无测试或真实模型运行证据 | R6 从 appendMessage 包装完整消息开始，再看 id / parentId；保留旧待巩固项，不以阶段收束代替全部验收 |
| 2026-10-25 | R6 主线收束，集中记录 | 用户正确推演祖先路径、分支设置、model_change 不新增消息、分支外压缩不生效、首次写入后的配置追加、新增 e5 的父节点及恢复位置 | 仅改内存 branch 的重启原题曾答错，已纠正待巩固；首次创建/内存缓冲/异常恢复不当成全部独立验证；无测试或真实模型运行证据 | R7 从 read 调用到结果进入下次请求的局部综合追踪开始，3–5 步、明确执行者和数据；之后再补 UI/存储和断言，不自动获得实施或安装授权 |
| 2026-10-25 | R7 首轮静态练习收束，集中记录 | 用户概括工具主线，并提出按调用 ID/内容验证结果、以入口记录区分失败阶段、在第二次 mock 上下文检查工具结果 | 按 ID 查工具的记忆错误已纠正为按 name；真实 I/O 等细节由助教补全；新断言未加入、原测试也未运行，不标整体验收通过 | 等待确认最小实际验证：先查依赖再尝试指定 mock 测试；缺依赖另确认安装，后续新增检查另确认实施范围，不调用真实 Provider |
| 2026-10-25 | R7 首次实际验证，集中记录 | 用户授权环境检查及原指定 mock 测试；依赖入口存在；Vitest v4.1.11 输出指定文件 1 passed、测试 1 passed / 37 skipped | 外层终端尾部报 TTY 错误，整条命令退出码 2；未确认正常退出；content / ID / 第二次请求输入等新断言仍未加入或运行 | 向用户报告原有断言证据与覆盖局限；增强测试另确认范围，不自行安装、构建或修改终端配置 |

## 恢复入口（Resume）

开始或恢复 Pi 源码学习会话，包括上下文压缩之后：

1. 开始或恢复会话时先读“当前进度”和本节；同一连续会话无需每条消息重读。复用已有结论，不从 R0 重讲或按旧日志退回已解决的问题。
2. 当前 R7 已进行首次实际验证：原指定 mock 测试的 Vitest 输出为 1 passed / 37 skipped，但外层终端 TTY 错误使整条命令退出 2，不能记为正常退出或整条路线已完全验收。用户已凭印象复述工具主线，按 name 查工具等错误已对照源码校准；实际 read 文件操作由助教补全，不等于完整源码独立追踪通过。不要回到 R1 重新采访、重复检查已可用的 CLI 或无限拆字段题。
3. R7 已独立提出三类检查：按调用 ID 找 toolResult 并比较 content；用工具函数入口的 executed 记录区分未进入与进入后失败；在第二次 mock 调用检查上下文是否含上次工具结果。字段 toolCallId、第二个实参 llmContext.messages、精确期望值及入口记录前提已由助教补明。原测试没有被修改，这些新检查全部尚未加入、尚未运行。
4. 重要边界：stream.result() 的 newMessages 不等于后续请求的 currentContext.messages，mock 固定 done 不证明输入正确；副作用、事件存在、isError、结果内容和请求输入应分别观察。模型响应入口的输入检查不等于真实 Provider HTTP 验证。R3–R6 旧待巩固内容继续保留，包括仅改 branch 内存游标不跨重启持久化的原题错误、mock 模型不自动隔离真实工具、UI 与上下文数据路径不同；不能以本轮正确答案覆盖旧缺口。
5. 用户已授权且已尝试运行原 should handle tool calls and results 指定 mock 测试，见上方实际输出和退出异常。下一步可建议只增强此测试的 content / toolCallId 与第二次 mock 输入断言，但必须另获修改授权。失败阶段检查也仍是静态设计；不把这些未加入的检查算作已验证，不自行 npm ci/install、构建、修 shell 配置或调用真实模型。
6. 用户最新明确要求阶段性记录：只在大阶段结束或准备压缩/交接时集中更新 roadmap，不每轮写文件。讲解先给实际源码例子，问题要说清场景与期待输出，不机械叠题或恢复语法小测。本次未安装依赖或调用真实模型；CLI 缺失的旧阻塞已解除，不能再沿用旧环境结论。

### 测试环境与运行边界

2026-10-25 已获准检查环境并运行原指定 mock 测试。Vitest 输出指定文件 1 passed，测试 1 passed / 37 skipped；外层随后报 TTY 错误、整条命令退出码 2，未确认正常退出。本次没有安装依赖、修改源码或测试、运行构建或请求真实 provider。

本次实际运行的命令如下（工具工作目录为 packages/agent；从仓库根目录手动执行时先 cd，兼容 fish）。只运行指定测试，不运行全套 Vitest：

```fish
cd packages/agent
node ../../node_modules/vitest/dist/cli.js --run test/agent-loop.test.ts -t "should handle tool calls and results"
```

如入口或依赖仍缺失，先确认环境和处理范围，再遵循 [AGENTS.md](AGENTS.md) 的 `--ignore-scripts` 安装策略；不默认依赖已安装，不默认允许安装，也不把启动失败写成断言失败。

## 目标与方法

学习结束后应能：

- 沿一次具体请求解释 Pi 如何调用模型、执行工具、维护会话。
- 区分 `agentLoop`、`Agent`、`AgentSession`、`SessionManager` 的职责。
- 区分消息、事件、模型请求上下文和持久化会话。
- 从测试定位负责的实现，指出哪些行为有断言证据、哪些尚未覆盖；另行说明测试是否实际运行过。

不以背代码、记住全部字段、通读仓库或实现一个新 Agent 为目标。

学习顺序：**测试 → 追踪具体值 → 实现 → 整体架构**。先解释一个具体行为，再补充必要概念；TypeScript 语法按需补充。后续 R2–R7 是待学习路线，不是已完成记录。

## R0：整体地图

**状态：已有基础，非独立验收通过。**

入口：[packages/coding-agent/docs/how-pi-works.zh.md](packages/coding-agent/docs/how-pi-works.zh.md)。已读完，不因恢复会话就要求重读。

职责地图：

| 包 | 本阶段需要理解的职责 |
| --- | --- |
| `ai` | 模型协议与访问 |
| `agent` | loop、工具调度和运行状态 |
| `coding-agent` | Harness：把模型、工具、会话和交互接口组装成可用应用 |
| `tui` | 终端显示 |

**验收：**能沿“输入 → 模型 → 工具 → 再请求模型 → 回复”口述各包的职责。只需建立地图，不展开所有模块。

## R1：最小工具测试

**状态：基础静态阅读问答已核对，已转入后续阶段；原测试在 R7 首次实际运行。** Vitest 输出目标测试通过、37 个跳过，但外层终端 TTY 错误导致命令退出 2，未确认正常退出。类型关系由用户正确解释后，角色阻塞已澄清；这不表示所有实现细节均已掌握，也不覆盖尚未加入的新断言。

入口：[packages/agent/test/agent-loop.test.ts](packages/agent/test/agent-loop.test.ts)，测试 `it("should handle tool calls and results", ...)`。当前参考范围 305–399 行，定位以测试名为准。

### 四步学习范围

1. **拆测试结构：**找到输入 `userPrompt`、`context`、`config`、`tool`、mock `streamFn`、`agentLoop` 入口、事件收集和断言。
2. **追踪 `hello`：**从 mock 的 `arguments.value = "hello"` 到 `execute(_toolCallId, params)`，再到 `executed.push(params.value)`，最后对照 `expect(executed).toEqual(["hello"])`。
3. **区分三类证据：**`executed` 记录工具执行的副作用；`tool_execution_start` / `tool_execution_end` 是运行事件；`await stream.result()` 取得本次 loop 的消息结果。不能把这三者当成同一份数据。
4. **追踪结果加工：**`execute` 返回 `toolUsage` → `afterToolCall` 读取原 usage 并记录到 `observedToolUsage` → 返回 `{ usage: patchedToolUsage }` → 最终 `toolResult.usage` 使用补丁值。

这些步骤列出 R1 范围，不要求恢复时重讲。参数值链、回调、错误标记、断言缺口、两次响应理由、事件归类及联合类型关系已有逐段问答依据；当前进入 R2，在实际构造消息的路径继续巩固角色。完整列表中的自动 system 工具声明此前由助教遗漏，已纠正。静态理解不能替代实际测试运行记录。

### 测试中的具体数据

以下描述来自测试定义，不代表测试已成功运行。

- 工具名是 `echo`，参数 schema 要求 `value` 为字符串；`context.tools` 包含该工具。
- mock `streamFn` 第一次构造 `toolCall`：`id: "tool-1"`、`name: "echo"`、`arguments: { value: "hello" }`。
- 本场景的第二次模型响应是文本 `"done"`：先提出工具调用，拿到工具结果后再回复。不要理解成每个请求都固定调用模型两次。
- `execute` 把 `params.value` 推入 `executed`，返回文本 `"echoed: hello"`、`details: { value: "hello" }` 和 `usage: toolUsage`。
- `afterToolCall` 先观察原始 `result.usage`，再返回 `patchedToolUsage` 覆盖 usage。
- `tool_execution_start` 不含执行结果；`tool_execution_end` 含 `result` 和 `isError`。已讲解并通过错误标记场景核对部分理解，R2 中结合发出事件的代码继续巩固，不据此宣称所有字段均已掌握。

### 断言证明什么、不证明什么

| 测试中的断言 | 若测试运行通过，可支持的结论 |
| --- | --- |
| `expect(executed).toEqual(["hello"])` | 工具执行产生了预期副作用，收到的 `params.value` 是 `"hello"` |
| `expect(toolStart).toBeDefined()`、`expect(toolEnd).toBeDefined()` | 收集的事件中存在工具开始、结束事件 |
| `expect(toolEnd.isError).toBe(false)` | 该结束事件未标记工具错误 |
| `expect(observedToolUsage).toEqual(toolUsage)` | `afterToolCall` 观察到了工具原始 usage |
| 对 `toolResult.usage` 与 `patchedToolUsage` 的相等断言 | 消息结果中的工具 usage 已被替换 |

测试代码构造了 `"echoed: hello"`，但**没有直接断言工具结果的这段文本**。开始/结束事件存在，也不等于其所有字段都经过断言。应区分“代码做了什么”“断言覆盖什么”“实际运行是否通过”。

**验收：**能解释此场景为什么需要两次模型响应、为什么 `start` 无结果、哪条断言证明工具调用的副作用，以及行为与断言覆盖的区别；能追踪原 usage 到补丁 usage 的过程。不能仅凭听过讲解验收。

## R2：`agentLoop` 实现

**状态：基础主路径已逐段阅读并核对核心分支，已衔接 R3。** 首轮纯文本不执行工具的反事实已有用户解释；全套流式变体、队列/回调组合和实际测试未验收，相关细节在后续按需巩固，不将本轮基础阅读写成完全掌握或实测通过。

入口：

- [packages/agent/src/agent-loop.ts](packages/agent/src/agent-loop.ts)：`agentLoop`、`runLoop`、`streamAssistantResponse`。
- 按需查 [packages/agent/src/types.ts](packages/agent/src/types.ts) 中对应的消息、事件、工具和配置类型。

沿 R1 的值链追踪，不整份文件灌输：

1. 测试如何进入 `agentLoop`，`runLoop` 如何发起模型响应流。
2. 如何从模型响应识别 `toolCall`，查找工具并验证参数。
3. 沿 `executePreparedToolCall`、`finalizeExecutedToolCall`、`emitToolExecutionEnd` 追踪执行、结果加工和结束事件。
4. 工具结果如何成为消息，以及如何进入后续模型请求。
5. 区分 `toolCall`、`toolResult` 和 event；结合这次 trace 区分 turn 与整个 run，并说明循环继续或停止的条件。

**验收：**把 R1 的测试步骤映射到负责实现的函数；先预测“第一轮模型只返回文本、没有 `toolCall`”时走什么路径，再定位代码核对。

## R3：`Agent` 对象

**状态：主线与 echo 复习已讲解，独立理解仍待巩固，未验收通过。** 入口、状态、订阅、简化取消、steer / followUp 及运行与事件处理的连接已逐段说明；部分有问答依据，其余仍待核对。本次区分原测试直接调用 agentLoop 与 Agent.prompt 的源码映射，不新增取消或队列分支，不宣称 R3 验收通过。此前为降负荷暂停互动；用户后续指出问题，现已恢复轻量理解检查与短预习。

入口：

- [packages/agent/src/agent.ts](packages/agent/src/agent.ts)：`Agent`、`prompt`、`subscribe`、`abort`、`steer`、`followUp`。
- [packages/agent/test/agent.test.ts](packages/agent/test/agent.test.ts)：对应的状态、错误、取消、steering 和 follow-up 场景；阅读时用实际测试名定位。

重点：

- `prompt` 如何使用 loop，`Agent` 如何维护状态和消息。
- `subscribe` 接收到的事件与消息历史有什么不同。
- `abort`、错误和取消行为由哪些测试约束。
- 用“当前正在执行什么 → 何时放入消息 → 何时被取出”的具体时间顺序理解 `steer` 与 `followUp`，不只记名词。

**验收：**能区分 loop 的执行机制与 `Agent` 这个长期维护状态的对象，区分订阅事件与消息历史，并用一个时间顺序例子解释 steering 和 follow-up 的差别。

## R4：模型访问

**状态：主线静态阅读收束，可衔接 R5；不是全部独立验收通过。** 已沿 OpenAI Responses 的请求和响应路径阅读工具声明、结果回传、协议转换及流式参数收集。single 优先、调用 ID 关联和等待当前完整 assistant 响应的推演有用户回答依据；第二次请求输入随工具结果改变等纠正内容仍待巩固。未运行测试或真实模型，不以阶段衔接替代 R3/R4 全量掌握。

入口：

- [packages/ai/src/types.ts](packages/ai/src/types.ts)：模型访问相关类型。
- [packages/ai/src/index.ts](packages/ai/src/index.ts)：作为导出导航，不逐项通读。
- [packages/coding-agent/src/core/model-runtime.ts](packages/coding-agent/src/core/model-runtime.ts)：`streamSimple`、`prepareRequest`。
- [packages/ai/src/models.ts](packages/ai/src/models.ts)：`createProvider`、`apiFor`、`dispatch`；识别单实现和按协议选择实现。
- [packages/ai/src/api/openai-responses.ts](packages/ai/src/api/openai-responses.ts)：`streamSimple`、`stream`、`buildParams`；与 `openai-completions.ts` 对照实际请求格式。
- [packages/ai/src/api/openai-responses-shared.ts](packages/ai/src/api/openai-responses-shared.ts)：`convertResponsesMessages`、`processResponsesStream`；工具调用与结果双向转换，暂不展开其余协议分支。

重点：

- `model`、`provider`、`messages`、`tools` 之间的关系。
- 从统一协议沿一条 provider 路径追到模型访问，不遍历所有 provider。
- 区分模型响应流与 Agent 事件流。
- 回到 R1，说明 mock `streamFn` 注入在哪个边界、替代了哪一层。

**验收：**能解释 `streamFn` 模拟什么，以及为什么不需要 API key 就能测试工具循环。不以真实或付费调用作为默认验证手段。

## R5：Coding Agent Harness

**状态：基础主线静态阅读收束，可衔接 R6；不是全部独立验收通过。** 已以读取 README 讲解组装和工作目录、工具实际 I/O、运行中输入排队、两层订阅到 UI、message_end 到 SessionManager 的保存入口。用户对 Agent 执行/Session 配置的当前应用分工、followUp 场景选择、UI 待处理追踪和保存入口次数有正确回答依据；mock I/O 隔离等纠正内容仍待巩固。未改源码、未运行测试或模型。

Harness 指围绕 Agent 组装模型、工具、会话和对外接口的应用层，不是单个类的别名。

入口：

- [packages/coding-agent/src/core/sdk.ts](packages/coding-agent/src/core/sdk.ts)：`createAgentSession`。
- [packages/coding-agent/src/core/agent-session.ts](packages/coding-agent/src/core/agent-session.ts)：`AgentSession`、`prompt` 等调用链。
- [packages/coding-agent/src/core/tools/read.ts](packages/coding-agent/src/core/tools/read.ts)：read 工具的定义和执行。

以“读取 README”为具体例子：

1. 解释 SDK 如何组装模型、工具与会话。
2. 追踪 `AgentSession` 如何准备运行、处理事件并向上层提供输出。
3. 找到 read 工具的定义、参数与实际读取动作。
4. 分清模型提出调用、loop 调度执行、工具读取文件的职责，并讨论工具权限问题。

**验收：**能解释 `AgentSession` 是 Harness 的核心编排部分，但不等于整个 Harness；能定位上述三层职责。暂不深入 CLI 选项或 TUI 渲染。

## R6：上下文与会话

**状态：主线静态阅读收束，可衔接 R7；不等于全部独立验收或实测通过。** 已讲解条目树、活动路径、配置/消息重建、分支局部压缩、首次写入/追加、正常文件恢复。路径回溯、分支设置、非消息条目、分支外压缩不生效、已写入后继续追加，以及新节点 parentId / 重开 leafId 的推演有用户正确回答依据。仅 branch 的内存游标是否跨重启保存曾答错，已纠正，不能以追加新节点的正确回答覆盖原题缺口；未运行测试或真实模型。

入口：

- [packages/coding-agent/src/core/session-manager.ts](packages/coding-agent/src/core/session-manager.ts)：`SessionManager`。
- [packages/coding-agent/docs/session-format.zh.md](packages/coding-agent/docs/session-format.zh.md)。
- [packages/coding-agent/docs/compaction.zh.md](packages/coding-agent/docs/compaction.zh.md)。

重点：

- 区分内存消息与磁盘 JSONL 会话条目。
- 理解条目的 parent 关系、活动分支和请求上下文的重建。
- 追踪 system prompt、tools 和 context 变换如何影响实际模型请求。
- 理解压缩摘要不等于删除原始历史。
- 当前 CLI 会话主线是 `SessionManager`，不是 `durable` 实验运行时。

**验收：**能解释为什么一个会话文件可包含多个分支，但一次请求只取相关上下文；能说明“磁盘历史”不等于“本次发送给模型的全部内容”。

## R7：综合追踪与验证

**状态：首轮静态复述与断言设计已进行，原指定 mock 测试已有 Vitest 通过输出；不是整体验收完成。** 用户能概括工具主线，查找键及具体文件 I/O 等落点已由助教校准或补全；能提出结果按 ID/内容检查、失败阶段的副作用证据、第二次 mock 输入检查。本次运行原测试，Vitest 输出 1 passed / 37 skipped，但外层终端 TTY 错误导致整条命令退出 2，未确认正常退出；没有修改测试，这些新断言仍未加入或运行。UI/存储整合等完整独立追踪仍待巩固；增强测试须另确认授权，不自动安装依赖。

独立追踪“读 README 并总结”：

```text
输入 → AgentSession → Agent → loop → 模型
     → 工具执行 → 结果反馈 → 模型最终回复
```

在这条链上进一步定位事件如何展示、会话如何持久化，而不是把所有责任归给同一个对象。

提出有意义的 mock 回归检查，例如：

- 模型没有返回 `toolCall` 时，不应执行工具。
- 明确断言工具结果文本，而不只检查执行副作用或事件存在。
- 工具执行出错时，观察错误结果和相应事件。

先预测行为，再做针对性验证。提出检查思路不等于获得修改测试或源码的授权；必要的实际代码修改另按用户授权和项目规则执行。

**验收：**对一个具体行为，能同时说清预期行为、负责的实现、证明它的断言和覆盖局限；明确哪些检查实际运行过，哪些只是预测或静态阅读。

## 教学与进度更新协议

- 每段只处理一个具体问题；用户提问时先答当前问题，不借机展开整套架构。
- 控制抽象层次：优先用当前层的已查证接口规则解释因果；必要跨层时先标明目的与最小范围，证明后回到主线，不把每个追问连锁变成下一份作业。底层实现可记“已查证、待回看”，不是当前一律必须掌握。用户反馈晕或信息过载时，临时减量或暂停预习、测问，先收束主线，并说明恢复节点；不得将临时降负荷无限延长或误记为用户永久取消互动。
- 给出文件路径、函数或测试名和必要阅读范围。行号仅辅助，不能作为唯一定位依据。先用当前源码中的实际数据、请求片段或运行场景解释，再补充必要术语；不要只讲理论或用不清楚的“展开调用”要求替代教学。
- 每次教学回复末尾默认提供短预习：一个已核实的文件/函数、小范围、一个阅读问题，约 5–10 分钟，并说明暂时跳过什么；不把提前阅读当作验收通过。开始较长查阅前，已有可靠入口时可先给等待期间的阅读任务。
- 可开第二个辅助对话，只解释用户指定的局部代码或 TypeScript 语法；先读本路线了解范围，不推进主线、不修改 roadmap 或源码。仅主对话维护进度；不假设两个对话自动共享历史，辅助结论由用户带回主线核对。
- 常规节奏为“一小段讲解 → 一个理解问题 → 一小段预习”。先等用户回答，再明确同意或纠错并给源码或测试证据；不边出题边给答案、不将一个回答追成多份作业。问题优先检查具体调用链、参数流、职责边界或行为变化，不将基础语法辨认、找到唯一函数名当作有效预习；也不因回答有误否定用户对题目质量的反馈。必要时一份有边界的阅读任务可兼作理解检查，不机械要求每轮两份任务；不靠记忆背诵验收。
- 引入新公开方法或概念时，先说明要解决的问题、真实调用方和调用时机，再进入内部实现；不要省略上下文后直接使用“注册”等尚未引入的说法。用户说不理解时，先查缺的是调用链、数据来源还是设计理由，不能默认其不懂基础语法；本次 subscribe 疑问明确是调用方缺失。
- 复用已经确认的结论，不重复采访或重复灌输整份文档；TypeScript 语法只按当前问题补充。
- 按用户最新明确要求，只在大阶段结束或准备压缩、交接时集中更新 roadmap；不逐条消息写文件或为保存进度反复调用工具。记录当前阶段、已确认理解、已讲解待巩固、下一步和阻塞，不补造掌握证据。会话开始/恢复时读取，连续会话内可复用，不必每轮重读。
- 区分“资料已读”“助教已讲解”“用户已独立解释”“测试已运行并通过”。不得相互替代，也不虚构完成日期。
- 发现分支话题时，在停车区留下简短记录；不因遇到陌生名词自动改换主线。
- 不默认修改源码、安装依赖、运行测试或调用真实付费模型；需要时单独确认范围并遵守仓库规则。

## 非主线与停车区

以后可选修，但不影响当前继续 R1：

- Extensions、Skills、MCP。
- TUI 渲染与按键处理。
- 所有 provider 的协议细节。
- EventStream 内部完成策略、lazy 加载、ID 规范化和缓存；当前只用已查证的边界解释，不设为继续主线的统一门槛。
- `durable` 实验运行时。
- 发布、安装、打包和完整配置。

当前没有另外确认的临时分支任务；后续只在这里简短记录，不自动将选修项升级为当前任务。
