> 译注：本文是[英文原文](compaction.md)的完整中文译文，中文内容可能滞后于原文；所有代码示例均保持原样。

<a id="compaction-reference"></a>
# 上下文压缩参考

本文介绍自动上下文压缩、分支摘要、持久化条目以及扩展钩子。用户操作流程见[会话与上下文](sessions.md#manage-conversation-context)。

**源文件**（[pi](https://github.com/earendil-works/pi)）：
- [`packages/coding-agent/src/core/compaction/compaction.ts`](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/compaction/compaction.ts) - 自动上下文压缩逻辑
- [`packages/coding-agent/src/core/compaction/branch-summarization.ts`](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/compaction/branch-summarization.ts) - 分支摘要
- [`packages/coding-agent/src/core/compaction/utils.ts`](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/compaction/utils.ts) - 共享工具函数（文件跟踪、序列化）
- [`packages/coding-agent/src/core/session-manager.ts`](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/session-manager.ts) - 条目类型（`CompactionEntry`、`BranchSummaryEntry`）
- [`packages/coding-agent/src/core/extensions/types.ts`](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/extensions/types.ts) - 扩展事件类型

项目中的 TypeScript 定义可在 `node_modules/@earendil-works/pi-coding-agent/dist/` 中查看。

<a id="overview"></a>
## 概览

Pi 有两种摘要机制：

| 机制 | 触发条件 | 用途 |
|-----------|---------|---------|
| 上下文压缩 | 上下文超过阈值，或执行 `/compact` | 对旧消息生成摘要，释放上下文空间 |
| 分支摘要 | 使用 `/tree` 导航 | 切换分支时保留上下文 |

两者采用密切相关的结构化格式，并累计跟踪文件操作。摘要请求会禁用提示词缓存写入，因为这些一次性提示词不太可能被复用。

<a id="compaction"></a>
## 上下文压缩

<a id="when-it-triggers"></a>
### 触发时机

满足以下条件时触发自动上下文压缩：

```
contextTokens > contextWindow - reserveTokens
```

默认情况下，`reserveTokens` 为 16384 个 token（可在 `~/.pi/agent/settings.json` 或 `<project-dir>/.pi/settings.json` 中配置）。这会为 LLM 的响应预留空间。

在多轮 Agent 运行期间，Pi 会在工具执行完成、结果追加之后，开始下一次助手响应之前，检查规范投影后的上下文。如果超过阈值，Pi 会在 `prepareNextTurn` 期间压缩上下文，随后在 `turn_start` 之前执行现有的补查引导消息轮询。如果已完成的工具批次使运行终止，且没有排队消息需要另一次响应，就会跳过这一轮次间检查。Pi 还会在新的用户提示词之前检查，并在底层运行结束后，对最终尝试执行溢出恢复。

提供商的上下文溢出错误，或提前出现的最终 `stopReason: "length"`，可以触发一次先压缩再重试的恢复尝试。因长度限制而结束且带有工具调用的响应，会保留其合成的失败工具结果，并遵循常规工具/队列调度器，而不是强制结束运行。

也可以使用 `/compact [instructions]` 手动触发，可选指令用于指定摘要重点。

<a id="how-it-works"></a>
### 工作原理

1. **查找截断点**：从最终确定的会话投影末尾向前遍历，累计 token 估算值，直到达到 `keepRecentTokens`（默认为 20k，可在 `~/.pi/agent/settings.json` 或 `<project-dir>/.pi/settings.json` 中配置）
2. **提取消息**：收集从上一次保留边界（或会话起点）到截断点之间的投影消息
3. **生成摘要**：调用 LLM 按结构化格式生成摘要；若有先前摘要，则将其作为迭代上下文传入
4. **追加条目**：保存包含摘要和 `firstKeptEntryId` 的 `CompactionEntry`
5. **重建上下文**：会话使用摘要加上从 `firstKeptEntryId` 开始的消息，为下一次请求重建上下文

```
Before compaction:

  entry:  0     1     2     3      4     5     6      7      8     9
        ┌─────┬─────┬─────┬──────┬─────┬─────┬──────┬──────┬─────┬─────┐
        │ hdr │ usr │ ass │ tool │ usr │ ass │ tool │ tool │ ass │ tool│
        └─────┴─────┴─────┴──────┴─────┴─────┴──────┴──────┴─────┴─────┘
                └────────┬───────┘ └──────────────┬──────────────┘
               messagesToSummarize            kept messages
                                   ↑
                          firstKeptEntryId (entry 4)

After compaction (new entry appended):

  entry:  0     1     2     3      4     5     6      7      8     9     10
        ┌─────┬─────┬─────┬──────┬─────┬─────┬──────┬──────┬─────┬─────┬─────┐
        │ hdr │ usr │ ass │ tool │ usr │ ass │ tool │ tool │ ass │ tool│ cmp │
        └─────┴─────┴─────┴──────┴─────┴─────┴──────┴──────┴─────┴─────┴─────┘
               └──────────┬──────┘ └──────────────────────┬───────────────────┘
                 not sent to LLM                    sent to LLM
                                                         ↑
                                              starts from firstKeptEntryId

What the LLM sees:

  ┌────────┬─────────┬─────┬─────┬──────┬──────┬─────┬──────┐
  │ system │ summary │ usr │ ass │ tool │ tool │ ass │ tool │
  └────────┴─────────┴─────┴─────┴──────┴──────┴─────┴──────┘
       ↑         ↑      └─────────────────┬────────────────┘
    prompt   from cmp          messages from firstKeptEntryId
```

重复压缩时，摘要范围从上一次压缩的保留边界（`firstKeptEntryId`）开始，而不是从压缩条目本身开始；如果在路径中找不到该保留条目，则回退到上一次压缩条目之后的条目。不保留任何消息的压缩会将自身 ID 记录为 `firstKeptEntryId`；再次压缩从该条目之后开始。这样会将上一次压缩后保留下来的消息也纳入下一轮摘要，避免丢失这些消息。Pi 还会在写入新的 `CompactionEntry` 之前，根据重建并应用上下文编辑后的会话投影重新计算 `tokensBefore`，使 token 数量反映实际被替换的压缩前上下文。被省略的原始条目仍保留在存储中，但不影响截断点选择、摘要、检查点或 token 估算。

<a id="overflow-and-length-recovery-ordering"></a>
### 溢出与长度限制恢复的执行顺序

恢复会保留现有生命周期和队列顺序。已完成的尝试仍对 `turn_end` 和 `agent_end` 可见；运行后的恢复随后会修复持久化的模型上下文，再发起一次全新的重试：

```text
persist final assistant response
→ extension/public turn_end
→ extension/public agent_end
→ append context_edit omissions for the selected attempt
→ for overflow/length: run session_before_compact and append compaction on success
→ start the retry as a fresh run
```

如果恢复压缩失败或被取消，Pi 会保留省略编辑，不追加压缩条目，也不安排内部重试。已有排队工作仍由常规引导消息和后续消息规则管理。恢复处理后，`agent_before_settle` 看到的是修复后的投影。原始对话历史、导出、计费总额以及历史搜索扩展仍可查看被省略的尝试。

<a id="split-user-message-spans"></a>
### 拆分用户消息区段

用户消息区段从一条用户消息开始，包含直到下一条用户消息之前的所有轮次。通常，上下文压缩在用户消息边界处截断。

当某个用户消息区段超过 `keepRecentTokens` 时，截断点会落在该区段内部的一条助手消息处。这就是拆分用户消息区段的情况：

```
Split user-message span (one span exceeds budget):

  entry:  0     1     2      3     4      5      6     7      8
        ┌─────┬─────┬─────┬──────┬─────┬──────┬──────┬─────┬──────┐
        │ hdr │ usr │ ass │ tool │ ass │ tool │ tool │ ass │ tool │
        └─────┴─────┴─────┴──────┴─────┴──────┴──────┴─────┴──────┘
                ↑                                     ↑
         turnStartIndex = 1                  firstKeptEntryId = 7
                │                                     │
                └──── turnPrefixMessages (1-6) ───────┘
                                                      └── kept (7-8)

  isSplitTurn = true
  messagesToSummarize = []  (no earlier user-message spans)
  turnPrefixMessages = [usr, ass, tool, ass, tool, tool]
```

对于拆分的用户消息区段，Pi 会生成两份摘要并合并：
1. **历史摘要**：先前的上下文（如果有）
2. **用户消息区段前缀摘要**：被拆分用户消息区段的前半部分

<a id="cut-point-rules"></a>
### 截断点规则

有效的截断点包括：
- 用户消息
- 助手消息
- BashExecution 消息
- 自定义消息（custom_message、branch_summary）

绝不在工具结果处截断（工具结果必须与对应工具调用保留在一起）。

准备阶段只有在一个对上下文不可见的后缀包含被省略的助手尝试，且不包含任何未被省略、会产生上下文内容的条目时，才会将保留边界向前推进至该后缀。恢复时的 `context_edit` 省略操作满足此规则；本身对上下文不可见的元数据可以与它们共存。单独的元数据和新追加的自定义消息不会移动截断点。影响候选输入或待摘要前缀的替换编辑也会阻止推进，因为被省略的助手回答的是编辑前的输入；对最终会被省略的后缀条目进行替换则仍然安全。这样既能对恢复后超出预算的输入生成摘要，又能保留让已放弃尝试继续被省略的编辑，避免让记录管理操作改变新模型输入是否被原样保留。

<a id="compactionentry-structure"></a>
### CompactionEntry 结构

定义于 [`session-manager.ts`](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/session-manager.ts)：

```typescript
interface CompactionEntry<T = unknown> {
  type: "compaction";
  id: string;
  parentId: string | null;
  timestamp: string;
  summary: string;
  firstKeptEntryId: string;
  tokensBefore: number;
  usage?: Usage;       // LLM usage that generated the summary
  fromHook?: boolean;  // true if provided by extension (legacy field name)
  details?: T;         // implementation-specific data
}

// Default compaction uses this for details (from compaction.ts):
interface CompactionDetails {
  readFiles: string[];
  modifiedFiles: string[];
}
```

扩展可以在 `details` 中存储任何可 JSON 序列化的数据。默认上下文压缩会跟踪文件操作，但自定义扩展实现可以使用自己的结构。无论是生成的摘要还是扩展提供的摘要，都会在有 LLM `usage` 时存储它，使会话总量包含摘要生成的用量。

实现见 [`prepareCompaction()`](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/compaction/compaction.ts) 和 [`compact()`](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/compaction/compaction.ts)。直接通过程序生成摘要时，`generateSummary()` 返回摘要文本，`generateSummaryWithUsage()` 返回 `{ text, usage }`。

<a id="branch-summarization"></a>
## 分支摘要

<a id="when-it-triggers-1"></a>
### 触发时机

使用 `/tree` 导航到其他分支时，Pi 会询问是否对你即将离开的工作生成摘要。这会将离开分支的上下文注入新分支。

<a id="how-it-works-1"></a>
### 工作原理

1. **查找共同祖先**：新旧位置共享的最深节点
2. **收集条目**：从旧叶节点向前回溯到共同祖先
3. **按预算准备**：在 token 预算内纳入消息（最新消息优先）
4. **生成摘要**：调用 LLM 按结构化格式生成摘要
5. **追加条目**：在导航位置保存 `BranchSummaryEntry`

```
Tree before navigation:

         ┌─ B ─ C ─ D (old leaf, being abandoned)
    A ───┤
         └─ E ─ F (target)

Common ancestor: A
Entries to summarize: B, C, D

After navigation with summary:

         ┌─ B ─ C ─ D
    A ───┤
         └─ E ─ F ─ [summary of B,C,D] (new leaf)
```

<a id="cumulative-file-tracking"></a>
### 累计文件跟踪

默认上下文压缩和分支摘要会累计跟踪文件。两者都会从待摘要消息的工具调用中提取文件操作。上下文压缩还会继承上一次 Pi 生成的压缩摘要中的文件列表。分支摘要则会继承待摘要条目中由 Pi 生成的分支摘要所含的文件列表。

因此，文件跟踪会跨默认上下文压缩和嵌套的默认分支摘要累计。对于扩展生成且 `fromHook` 字段为 `true` 的摘要，Pi 不会自动继承其文件列表；扩展自行管理自己的 `details` 格式。

<a id="branchsummaryentry-structure"></a>
### BranchSummaryEntry 结构

定义于 [`session-manager.ts`](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/session-manager.ts)：

```typescript
interface BranchSummaryEntry<T = unknown> {
  type: "branch_summary";
  id: string;
  parentId: string | null;
  timestamp: string;
  summary: string;
  fromId: string;      // Entry we navigated from
  usage?: Usage;       // LLM usage that generated the summary
  fromHook?: boolean;  // true if provided by extension (legacy field name)
  details?: T;         // implementation-specific data
}

// Default branch summarization uses this for details (from branch-summarization.ts):
interface BranchSummaryDetails {
  readFiles: string[];
  modifiedFiles: string[];
}
```

与上下文压缩相同，扩展可以在 `details` 中存储自定义数据。

实现见 [`collectEntriesForBranchSummary()`](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/compaction/branch-summarization.ts)、[`prepareBranchEntries()`](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/compaction/branch-summarization.ts) 和 [`generateBranchSummary()`](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/compaction/branch-summarization.ts)。

<a id="summary-format"></a>
## 摘要格式

两种格式都包含目标、约束与偏好、进度、关键决策以及后续步骤。上下文压缩摘要还包含关键上下文。分支摘要在后续步骤之后结束。相关时，Pi 会为任一种格式追加文件列表。

上下文压缩摘要使用以下格式：

```markdown
## Goal
[What the user is trying to accomplish]

## Constraints & Preferences
- [Requirements mentioned by user]

## Progress
### Done
- [x] [Completed tasks]

### In Progress
- [ ] [Current work]

### Blocked
- [Issues, if any]

## Key Decisions
- **[Decision]**: [Rationale]

## Next Steps
1. [What should happen next]

## Critical Context
- [Data needed to continue]

<read-files>
path/to/file1.ts
path/to/file2.ts
</read-files>

<modified-files>
path/to/changed.ts
</modified-files>
```

<a id="message-serialization"></a>
### 消息序列化

生成摘要之前，消息会通过 [`serializeConversation()`](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/compaction/utils.ts) 序列化为文本：

```
[User]: What they said
[Assistant]: Response text
[Assistant tool calls]: read(path="foo.ts"); edit(path="bar.ts", ...)
[Tool result]: Output from tool
```

这样可以防止模型将其视为需要继续的对话。

思考块会被省略。提供商返回的是真实推理的摘要，而且重放的思考内容往往并不属于模型当前的上下文，因此纳入它们可能使摘要请求大于它要概括的上下文。

序列化期间，工具结果会截断至 2000 个字符。超出限制的内容会替换为一个标记，说明截断了多少字符。这使摘要请求保持在合理的 token 预算内，因为工具结果（尤其来自 `read` 和 `bash` 的结果）通常是上下文大小的主要来源。

<a id="custom-summarization-via-extensions"></a>
## 通过扩展自定义摘要

扩展可以拦截并自定义上下文压缩和分支摘要。事件类型定义见 [`extensions/types.ts`](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/extensions/types.ts)。

<a id="session_before_compact"></a>
### session_before_compact

在自动上下文压缩或 `/compact` 之前触发。可以取消压缩或提供自定义摘要。参见类型文件中的 `SessionBeforeCompactEvent` 和 `CompactionPreparation`。

```typescript
pi.on("session_before_compact", async (event, ctx) => {
  const { preparation, branchEntries, customInstructions, reason, willRetry, signal } = event;

  // preparation.messagesToSummarize - messages to summarize
  // preparation.turnPrefixMessages - user-message-span prefix (if isSplitTurn)
  // preparation.previousSummary - previous compaction summary
  // preparation.fileOps - extracted file operations
  // preparation.tokensBefore - context tokens before compaction
  // preparation.firstKeptEntryId - where kept messages start
  // preparation.settings - effective settings after applying model overrides

  // branchEntries - all entries on current branch (for custom state)
  // reason - "manual" (/compact), "threshold", or "overflow"
  // willRetry - whether the aborted turn is retried after compaction (overflow recovery)
  // signal - AbortSignal (pass to LLM calls)

  // Cancel:
  return { cancel: true };

  // Custom summary:
  return {
    compaction: {
      summary: "Your summary...",
      firstKeptEntryId: preparation.firstKeptEntryId,
      tokensBefore: preparation.tokensBefore,
      // usage: summaryResponse.usage, // Optional; included in session totals
      details: { /* custom data */ },
    }
  };
});
```

<a id="converting-messages-to-text"></a>
#### 将消息转换为文本

若要使用自己的模型生成摘要，请通过 `serializeConversation` 将消息转换为文本：

```typescript
import { convertToLlm, serializeConversation } from "@earendil-works/pi-coding-agent";

pi.on("session_before_compact", async (event, ctx) => {
  const { preparation } = event;
  
  // Convert AgentMessage[] to Message[], then serialize to text
  const conversationText = serializeConversation(
    convertToLlm(preparation.messagesToSummarize)
  );
  // Returns:
  // [User]: message text
  // [Assistant]: response text
  // [Assistant tool calls]: read(path="..."); bash(command="...")
  // [Tool result]: output text

  // Now send to your model for summarization
  const { summary, usage } = await myModel.summarize(conversationText);
  
  return {
    compaction: {
      summary,
      firstKeptEntryId: preparation.firstKeptEntryId,
      tokensBefore: preparation.tokensBefore,
      usage,
    }
  };
});
```

使用不同模型的完整示例见 [custom-compaction.ts](../examples/extensions/custom-compaction.ts)。

<a id="session_compact_failed"></a>
### session_compact_failed

手动或自动上下文压缩失败或被中止时触发。这适用于需要将 `session_before_compact` 尝试与最终结果配对的遥测扩展。

```typescript
pi.on("session_compact_failed", async (event, ctx) => {
  const { reason, errorMessage, aborted, willRetry, fromExtension } = event;
  // reason - "manual" (/compact), "threshold", or "overflow"
  // errorMessage - present for non-abort failures
  // aborted - true for canceled/aborted compactions
  // willRetry - whether the aborted turn would have retried after compaction
  // fromExtension - whether extension-provided compaction content was being used
});
```

<a id="session_before_tree"></a>
### session_before_tree

在 `/tree` 导航之前触发。无论用户是否选择生成摘要，都会触发。可以取消导航或提供自定义摘要。

```typescript
pi.on("session_before_tree", async (event, ctx) => {
  const { preparation, signal } = event;

  // preparation.targetId - where we're navigating to
  // preparation.oldLeafId - current position (being abandoned)
  // preparation.commonAncestorId - shared ancestor
  // preparation.entriesToSummarize - entries that would be summarized
  // preparation.userWantsSummary - whether user chose to summarize

  // Cancel navigation entirely:
  return { cancel: true };

  // Provide custom summary (only used if userWantsSummary is true):
  if (preparation.userWantsSummary) {
    return {
      summary: {
        summary: "Your summary...",
        // usage: summaryResponse.usage, // Optional; included in session totals
        details: { /* custom data */ },
      }
    };
  }
});
```

参见类型文件中的 `SessionBeforeTreeEvent` 和 `TreePreparation`。

<a id="settings"></a>
## 设置

在 `~/.pi/agent/settings.json` 或 `<project-dir>/.pi/settings.json` 中配置上下文压缩：

```json
{
  "compaction": {
    "enabled": true,
    "reserveTokens": 16384,
    "keepRecentTokens": 20000
  }
}
```

| 设置 | 默认值 | 说明 |
|---------|---------|-------------|
| `enabled` | `true` | 启用自动上下文压缩 |
| `reserveTokens` | `16384` | 为 LLM 响应预留的 token 数量 |
| `keepRecentTokens` | `20000` | 保留的近期 token 数量（不生成摘要） |

通过 `"enabled": false` 禁用自动上下文压缩。你仍可以使用 `/compact` 手动压缩。

<a id="per-model-overrides"></a>
### 按模型覆盖设置

使用 `compaction.modelOverrides` 为不同模型调整 token 预算：

```json
{
  "compaction": {
    "reserveTokens": 16384,
    "keepRecentTokens": 20000,
    "modelOverrides": {
      "some-provider/big-model": {
        "reserveTokens": 400000
      }
    }
  }
}
```

对于上下文窗口为 1M 的模型，这一覆盖设置会在超过 600K 个 token 时触发压缩，并保留常规设置指定的近期 20000 个 token。其他模型仍采用常规的 16384-token 预留量。`reserveTokens` 还会影响摘要输出限制，且受模型最大输出 token 数量约束；它并不只是触发阈值。

键必须是精确、区分大小写的 `provider/modelId` 值，包括模型 ID 内部的所有斜杠。每个 `reserveTokens` 和 `keepRecentTokens` 值都独立按模型覆盖设置、常规设置、内置默认值的顺序回退。值必须是非负安全整数。匹配模型的覆盖设置中若有无效值，读取时会报错；只有省略字段才会回退到常规设置。模型覆盖条目必须是对象。无效的常规 token 设置在读取时会报错，即使当前模型存在有效覆盖设置也一样。只有省略的常规值才使用内置默认值。`enabled` 仍为全局设置，不区分模型。

这些解析后的值用于手动上下文压缩、所有自动阈值检查、溢出恢复，以及扩展可见的 `preparation.settings`。切换模型会影响后续检查和压缩，但不会改变常规设置。已经进行中的压缩使用该操作开始时捕获的模型和设置。分支摘要设置不受影响。

覆盖设置同时适用于全局和项目设置。文件会在查找前递归合并，因此全局的模型专用值优先于项目级的通用回退值；项目必须覆盖对应的模型条目才能修改它。详情见[设置](settings.md#per-model-compaction-overrides)。
