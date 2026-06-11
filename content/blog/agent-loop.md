---
title: An introduction of Claude Agent Loop
date: 2022-09-14
tags:
  - agent
  - engineering
publish: true
---


# Agent Loop的工作原理


启动Agent时，内部会运行与 Claude Code 相同的[执行循环](/en/how-claude-code-works#the-agentic-loop)：Claude 会评估您的提示，调用工具执行操作，接收结果，并重复此过程直至任务完成。本页面将解释该循环内部发生的情况，以便您能够有效地构建、调试和优化Agent。

## 什么是 Loop

每次Agent会话都遵循相同的流程：

<img src="https://mintcdn.com/claude-code/ikqp3_70mqIahteV/images/agent-loop-diagram.svg?fit=max&auto=format&n=ikqp3_70mqIahteV&q=85&s=1c6e8f28d80dba14a7287419656f1237" alt="Agent Loop图：您的提示进入Agent Loop，Claude 会进行评估，并请求工具调用（其结果反馈到另一次评估中），或返回最终答案" width="720" height="212" data-path="images/agent-loop-diagram.svg" />

1. **接收提示。** Claude 会接收您的提示，以及系统提示、工具定义和对话历史记录。SDK 会生成一个子类型为 `"init"` 的 [`SystemMessage`](#message-types)，其中包含会话元数据。
2. **评估并响应。** Claude 会评估当前状态并确定后续操作。它可能会以文本形式响应，请求一个或多个工具调用，或者两者都做。SDK 会生成一个包含文本和所有工具调用请求的 [`AssistantMessage`](#message-types)。
3. **执行工具。** SDK 会运行每个请求的工具并收集结果。每组工具结果都会反馈给 Claude，用于下一步决策。您可以使用[Hook](/en/agent-sdk/hooks)在工具运行前拦截、修改或阻止工具调用。
4. **重复。** 步骤 2 和 3 循环重复。每个完整循环算作一次操作。Claude 会持续调用工具并处理结果，直到不再调用工具且产生响应为止。
5. **返回结果。** SDK 会生成一个最终的 [`AssistantMessage`](#message-types)，其中包含文本响应（不调用工具），然后生成一个 [`ResultMessage`](#message-types)，其中包含最终文本、令牌使用情况、成本和会话 ID。

一个简单的问题（“这里有哪些文件？”）可能只需要调用一两次 `Glob` 函数并返回结果。而一个复杂的任务（“重构身份验证模块并更新测试”）则可能需要多次调用工具，包括读取文件、编辑代码和运行测试，Claude 会根据每次的结果调整其处理方式。

## 轮到你了

一次循环是指循环内的一个往返：Claude 生成包含工具调用的输出，SDK 执行这些工具，并将结果自动反馈给 Claude。整个过程无需将控制权交还给你的代码。循环持续进行，直到 Claude 不再生成任何工具调用的输出为止，此时循环结束，并输出最终结果。

设想一下，对于“修复 auth.ts 中失败的测试”这个提示，完整的会话可能会是什么样子。

首先，SDK 会将您的提示发送给 Claude，并返回一个包含会话元数据的 [`SystemMessage`](#message-types)。然后循环开始：

1. **第一轮：** Claude 调用 `Bash` 运行 `npm test`。SDK 生成一个包含工具调用的 [`AssistantMessage`](#message-types)，执行命令，然后生成一个包含输出的 [`UserMessage`](#message-types)（三个失败）。
2. **第二轮：** Claude 对 `auth.ts` 和 `auth.test.ts` 调用 `Read` 方法。SDK 返回文件内容并生成一个 `AssistantMessage`。
3. **第三步：** Claude 调用 `Edit` 修复 `auth.ts`，然后调用 `Bash` 重新运行 `npm test`。所有三个测试都通过了。SDK 返回一个 `AssistantMessage`。
4. **最后一步：** Claude 生成一个纯文本响应，不调用任何工具：“已修复身份验证错误，所有三个测试现在都通过了。” SDK 生成一个包含此文本的最终 `AssistantMessage`，然后生成一个包含相同文本以及成本和使用情况的 [`ResultMessage`](#message-types)。

一共四轮：三轮使用工具呼叫，最后一轮仅回复文字。

您可以使用 `max_turns` / `maxTurns` 来限制循环次数，该参数仅计算工具使用次数。例如，上述循环中的 `max_turns=2` 会在编辑步骤之前停止。您还可以使用 `max_budget_usd` / `maxBudgetUsd` 根据支出阈值来限制循环次数。

如果没有限制，循环会一直运行直到 Claude 自行完成，这对于范围明确的任务来说没问题，但对于开放式提示（例如“改进此代码库”）则可能会运行很长时间。为生产环境Agent设置预算是一个不错的默认设置。有关选项的参考，请参阅下面的[轮次和预算](#turns-and-budget)。

## 消息类型

循环运行时，SDK 会生成一个消息流。每条消息都带有类型标识，用于指示它来自循环的哪个阶段。五种核心类型是：

* **`SystemMessage`：**会话生命周期事件。`subtype` 字段用于区分它们：`"init"` 是第一条消息（会话元数据），而 `"compact_boundary"` 在 [compaction](#automatic-compaction) 之后触发。在 TypeScript 中，紧凑边界是其自身的 [`SDKCompactBoundaryMessage`](/en/agent-sdk/typescript#sdkcompactboundarymessage) 类型，而不是 `SDKSystemMessage` 的子类型。
* **`AssistantMessage`:** 在Claude每次做出回应后发出，包括最后一次纯文本回应。包含该回合的文本内容块和工具调用块。
* **`UserMessage`:** 每次工具执行后发出，并将工具结果内容发送回 Claude。循环过程中任何用户输入也会发出此消息。
* **`StreamEvent`:** 仅在启用部分消息时发出。包含原始 API 流事件（文本增量、工具输入块）。请参阅[流响应](/en/agent-sdk/streaming-output)。
* **`ResultMessage`:** 标志着Agent Loop的结束。它包含最终的文本结果、令牌使用情况、成本和会话 ID。检查 `subtype` 字段以确定任务是否成功或达到限制。少量尾随系统事件（例如 `prompt_suggestion`）可能会在此之后到达，因此应迭代流直至完成，而不是在结果处中断。请参阅[处理结果](#handle-the-result)。

这五种类型涵盖了两个 SDK 中完整的Agent Loop生命周期。TypeScript SDK 还会生成额外的可观测性事件（Hook事件、工具进度、速率限制、任务通知），这些事件提供更多细节，但并非驱动循环所必需。有关完整列表，请参阅[Python 消息类型参考](/en/agent-sdk/python#message-types)和[TypeScript 消息类型参考](/en/agent-sdk/typescript#message-types)。

### 处理消息

您处理哪些消息取决于您正在构建什么：

* **仅最终结果：** 处理 `ResultMessage` 以获取输出、成本以及任务是否成功或达到限制。
* **进度更新：**处理 `AssistantMessage` 以查看 Claude 每回合正在做什么，包括它调用了哪些工具。
* **实时流：**启用部分消息（Python 中的 `include_partial_messages`，TypeScript 中的 `includePartialMessages`）以实时获取 `StreamEvent` 消息。请参阅[实时流响应](/en/agent-sdk/streaming-output)。

检查消息类型的方法取决于 SDK：

* **Python:** 使用 `isinstance()` 检查从 `claude_agent_sdk` 导入的类的消息类型（例如，`isinstance(message, ResultMessage)`）。
* **TypeScript：**检查`type`字符串字段（例如，`message.type === "result"`）。`AssistantMessage`和`UserMessage`会将原始API消息包装在`.message`字段中，因此内容块位于`message.message.content`，而不是`message.content`。

<Accordion title="示例：检查消息类型并处理结果">
  <代码组>
    ```python Python 主题={null}
    from claude_agent_sdk import query, AssistantMessage, ResultMessage

    async for message in query(prompt="总结此项目"):
        如果 isinstance(message, AssistantMessage):
            print(f"回合完成：{len(message.content)} 个内容块")
        如果 isinstance(message, ResultMessage):
            如果 message.subtype == "success":
                print(message.result)
            别的：
                print(f"已停止：{message.subtype}")
    ```

    ```typescript TypeScript 主题={null}
    import { query } from "@anthropic-ai/claude-agent-sdk";

    for await (const message of query({ prompt: "总结此项目" })) {
      如果 (message.type === "assistant") {
        console.log(`回合完成：${message.message.content.length} 个内容块`);
      }
      如果 (message.type === "result") {
        如果 (message.subtype === "成功") {
          console.log(message.result);
        } 别的 {
          console.log(`已停止：${message.subtype}`);
        }
      }
    }
    ```
  </CodeGroup>
</Accordion>

## 工具执行

工具赋予您的Agent执行操作的能力。如果没有工具，Claude 只能回复文本。有了工具，Claude 可以读取文件、运行命令、搜索代码并与外部服务交互。

### 内置工具

该 SDK 包含了与 Claude Code 相同的工具：

| 类别 | 工具 | 功能 |
| :------------------ | :-------------------------------------------------------------- | :-------------------------------------------------------------------------- |
| **文件操作** | `读取`、`编辑`、`写入` | 读取、修改和创建文件 |
| **搜索** | `Glob`、`Grep` | 按模式查找文件，使用正则表达式搜索内容 |
| **执行** | `Bash` | 运行 shell 命令、脚本、git 操作 |
| **Web** | `WebSearch`、`WebFetch` | 搜索网络、获取和解析网页 |
| **发现** | `工具搜索` | 按需动态查找和加载工具，而不是预先加载所有工具 |
| **流程编排** | `Agent`、`Skill`、`AskUserQuestion`、`TaskCreate`、`TaskUpdate` | 生成子Agent、调用技能、询问用户、跟踪任务 |

除了内置工具之外，您还可以：

* **将外部服务**连接到 [MCP 服务器](/en/agent-sdk/mcp)（数据库、浏览器、API）
* **使用[自定义工具处理程序](/en/agent-sdk/custom-tools)定义自定义工具**
* **通过[设置源](/en/agent-sdk/claude-code-features)加载项目技能，以实现可重用的工作流

### 工具权限

Claude 会根据任务决定调用哪些工具，但您可以控制是否允许这些调用执行。您可以自动批准特定工具、完全阻止其他工具，或者要求所有工具都经过批准。以下三个选项共同决定哪些工具会运行：

* **`allowed_tools` / `allowedTools`** 会自动批准列出的工具。只读Agent的允许工具列表中包含 `["Read", "Glob", "Grep"]` 时，无需提示即可运行这些工具。未列出的工具仍然可用，但需要获得权限。
* **`disallowed_tools` / `disallowedTools`** 会阻止列出的工具运行，而忽略其他设置。有关工具运行前规则的检查顺序，请参阅[权限](/en/agent-sdk/permissions)。
* **`permission_mode` / `permissionMode`** 控制未被允许或拒绝规则涵盖的工具的处理方式。有关可用模式，请参阅[权限模式](#permission-mode)。

您还可以使用类似 `"Bash(npm *)"` 的规则来限定单个工具的权限范围，仅允许执行特定命令。有关完整的规则语法，请参阅[权限](/en/agent-sdk/permissions)。

当工具被拒绝时，Claude 会收到一条拒绝消息作为工具结果，通常会尝试不同的方法或报告无法继续。

### 并行工具执行

当 Claude 在单次操作中请求多个工具调用时，两个 SDK 可以根据工具的不同，选择并发或顺序运行。只读工具（例如 `Read`、`Glob`、`Grep` 和标记为只读的 MCP 工具）可以并发运行。修改状态的工具（例如 `Edit`、`Write` 和 `Bash`）则顺序运行，以避免冲突。

自定义工具默认顺序执行。要为自定义工具启用并行执行，请在其注解中设置 `readOnlyHint`。[TypeScript](/en/agent-sdk/typescript#tool) 和 [Python](/en/agent-sdk/python#tool) SDK 都使用 MCP SDK 中的此字段名称。

## 控制循环的运行方式

您可以限制循环的次数、成本、Claude 推理的深度，以及工具运行前是否需要批准。所有这些都是 [`ClaudeAgentOptions`](/en/agent-sdk/python#claudeagentoptions) (Python) / [`Options`](/en/agent-sdk/typescript#options) (TypeScript) 中的字段。

### 转弯和预算

| 选项 | 控制内容 | 默认值 |
| :--------------------------------------------- | :--------------------------- | :------- |
| 最大转弯次数（`max_turns` / `maxTurns`） | 最大工具使用往返次数 | 无限制 |
| 最大预算（`max_budget_usd` / `maxBudgetUsd`）| 停止前的最大费用 | 无限制 |

当达到任一限制时，SDK 会返回一个带有相应错误子类型（`error_max_turns` 或 `error_max_budget_usd`）的 `ResultMessage`。有关如何检查这些子类型，请参阅[处理结果](#handle-the-result)；有关语法，请参阅[`ClaudeAgentOptions`](/en/agent-sdk/python#claudeagentoptions) / [`Options`](/en/agent-sdk/typescript#options)。

### 努力程度

“努力程度”选项控制Claude运用推理的程度。较低的努力程度意味着每回合使用的标记更少，从而降低成本。并非所有模型都支持努力程度参数。请参阅[努力程度](https://platform.claude.com/docs/en/build-with-claude/effort)了解哪些模型支持该参数。

| 等级 | 行为 | 适合 |
| :--------- | :-------------------------------- | :------------------------------------------------------------- |
| `"低"` | 最少的推理，快速的响应 | 文件查找，目录列表 |
| `"中等"` | 平衡推理 | 常规编辑，标准任务 |
| “高” | 深入分析 | 重构、调试 |
| `"xhigh"` | 扩展推理深度 | 编码和智能体任务；推荐用于 Opus 4.8 和 Opus 4.7 |
| `"max"` | 最大推理深度 | 需要深度分析的多步骤问题 |

如果您不设置“effort”，则两个 SDK 都将该参数保持未设置状态，并遵循模型的默认行为。

<注>
  `effort` 参数会根据延迟和令牌成本来增加每次响应的推理深度。[扩展思维](https://platform.claude.com/docs/en/build-with-claude/extended-thinking) 是一个独立的功能，它会在输出中生成可见的思维链模块。它们是相互独立的：启用扩展思维时，你可以将 `effort` 设置为“low”，禁用扩展思维时，则设置为“max”。
</Note>

对于执行简单、范围明确的任务（例如列出文件或运行单个 grep 命令）的Agent，应使用较低的执行难度，以降低成本和延迟。可以在整个会话的顶级 `query()` 选项中设置 `effort` 值，也可以在 [`AgentDefinition`](/en/agent-sdk/subagents#agentdefinition-configuration) 中为每个子Agent设置 `effort` 字段，以覆盖会话级别的设置。

### 权限模式

权限模式选项（Python 中的 `permission_mode`，TypeScript 中的 `permissionMode`）控制Agent在使用工具之前是否请求批准：

| 模式 | 行为 |
| :------------------------- | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `"default"` | 未在允许规则中涵盖的工具会触发您的批准回调；没有回调则表示拒绝 |
| `"acceptEdits"` | 自动批准文件编辑和常用文件系统命令（`mkdir`、`touch`、`mv`、`cp` 等）；其他 Bash 命令遵循默认规则 |
| `"plan"` | Claude 会在不编辑源文件的情况下进行探索和规划；文件编辑永远不会自动批准，而是通过 `canUseTool` 回调提示 |
| `"dontAsk"` | 从不提示。只有[权限规则](/en/settings#permission-settings)预先批准的工具才能运行，其他所有操作均被拒绝 |
| `"auto"`（仅限 TypeScript）| 使用模型分类器来批准或拒绝每个工具调用。有关可用性和行为，请参阅[自动模式](/en/permission-modes#eliminate-prompts-with-auto-mode) |
| `"bypassPermissions"` | 除非显式匹配了 [`ask` 规则](/en/settings#permission-settings)，否则将运行所有允许的工具而无需询问；有关 ask 规则的优先级顺序，请参阅 [权限评估方式](/en/agent-sdk/permissions#how-permissions-are-evaluated)。在 Unix 系统上以 root 用户身份运行时无法使用。仅在隔离环境中使用，Agent的操作不会影响您关心的系统。

对于交互式应用程序，请使用带有工具审批回调的 `"default"` 来显示审批提示。对于开发机器上的自主Agent，`"acceptEdits"` 会自动批准文件编辑和常用文件系统命令（例如 `mkdir`、`touch`、`mv`、`cp` 等），同时仍然使用允许规则来限制其他 `Bash` 命令的执行。`"bypassPermissions"` 仅供 CI、容器或其他隔离环境使用。有关完整详细信息，请参阅[权限](/en/agent-sdk/permissions)。

＃＃＃ 模型

如果您未设置 `model`，SDK 将使用 Claude Code 的默认模型，该模型取决于您的身份验证方法和订阅。您可以显式设置 `model`（例如，`model="claude-sonnet-4-6"`）来指定特定模型，或者使用更小的模型来构建速度更快、成本更低的Agent。有关可用 ID，请参阅[models](https://platform.claude.com/docs/en/about-claude/models)。

## 上下文窗口

上下文窗口是指 Claude 在会话期间可获取的全部信息量。它不会在会话内的回合之间重置。所有内容都会累积：系统提示、工具定义、对话历史记录、工具输入和工具输出。在回合间保持不变的内容（系统提示、工具定义、CLAUDE.md）会自动进行[提示缓存](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)，从而降低重复前缀的开销和延迟。

### 什么消耗了语境

以下是各组件如何影响 SDK 中的上下文：

| 来源 | 加载时间 | 影响 |
| :----------------------- | :------------------------------------------------------------------------ | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **系统提示** | 每个请求 | 费用低廉且始终存在 |
| **CLAUDE.md 文件** | 通过 [`settingSources`](/en/agent-sdk/claude-code-features) 启动会话 | 每次请求都包含完整内容（但提示已缓存，因此只有第一次请求支付全部费用） |
| **工具定义** | 每个请求；MCP 模式默认延迟加载 | 内置工具模式会在每个请求中加载。[工具搜索](/en/agent-sdk/mcp#mcp-tool-search) 默认延迟加载 MCP 工具模式，在 Vertex AI 或非第一方 `ANTHROPIC_BASE_URL` 上回退到预先加载。有关完整矩阵，请参阅[配置工具搜索](/en/agent-sdk/tool-search#configure-tool-search)。
| **对话历史记录** | 逐轮累积 | 每轮都会增长：提示、回复、工具输入、工具输出 |
| **技能描述** | 通过设置源启动会话 | 简短摘要；完整内容仅在调用时加载 |

大型工具的输出会消耗大量的上下文信息。读取一个大文件或运行一个输出详细的命令，在单次操作中就可能消耗数千个令牌。上下文信息会在多轮操作中累积，因此，较长的会话（包含多次工具调用）比短会话积累的上下文信息要多得多。

### 自动压实

当上下文窗口接近其限制时，SDK 会自动压缩对话：它会汇总较早的历史记录以释放空间，同时保留最近的交流和关键决策。此时，SDK 会在流中发出一个 `type: "system"` 和 `subtype: "compact_boundary"` 的消息（在 Python 中，这是一个 `SystemMessage` 类型；在 TypeScript 中，这是一个单独的 `SDKCompactBoundaryMessage` 类型）。

压缩功能会将较早的消息替换为摘要，因此对话早期的一些具体指令可能无法保留。持久规则应放在 CLAUDE.md 文件中（通过 [`settingSources`](/en/agent-sdk/claude-code-features) 加载），而不是初始提示符中，因为 CLAUDE.md 的内容会在每次请求时重新注入。

您可以通过多种方式自定义压缩行为：

* **CLAUDE.md 中的摘要说明：** 压缩器会像读取其他任何上下文一样读取您的 CLAUDE.md 文件，因此您可以添加一个部分来告诉它在摘要时要保留哪些内容。部分标题是自由格式的（不是固定字符串）；压缩器会根据意图进行匹配。
* **`PreCompact` hook：** 在压缩操作执行之前运行自定义逻辑，例如归档完整转录文本。该 hook 接收一个 `trigger` 字段（`manual` 或 `auto`）。请参阅 [hooks](/en/agent-sdk/hooks)。
* **手动压缩：**发送 `/compact` 作为提示字符串，即可按需触发压缩。以这种方式发送的命令是 SDK 输入，而不是仅限 CLI 使用的快捷方式。请参阅[SDK 中的命令](/en/agent-sdk/slash-commands)。

<Accordion title="示例：CLAUDE.md 中的摘要说明">
  在项目的 CLAUDE.md 文件中添加一个章节，告诉压缩程序要保留哪些内容。标题名称没有特殊含义；使用任何清晰的标签即可。

  ```markdown CLAUDE.md 主题={null}
  # 简要说明

  总结这次谈话时，务必保留以下内容：
  - 当前任务目标和验收标准
  - 已读取或修改的文件路径
  - 测试结果和错误信息
  已做出的决定及其背后的理由
  ```
</Accordion>

### 保持上下文高效

针对长期运行的Agent人，以下是一些策略：

* **使用子Agent处理子任务。** 每个子Agent都从一个全新的对话开始（没有之前的消息历史记录，但它会加载自己的系统提示和项目级上下文，例如 CLAUDE.md）。它看不到父Agent的轮次，只有它的最终回复会作为工具结果返回给父Agent。主Agent的上下文会根据该摘要进行扩展，而不是根据完整的子任务记录。有关详细信息，请参阅[子Agent继承的内容](/en/agent-sdk/subagents#what-subagents-inherit)。
* **谨慎选择工具。** 每个工具定义都会占用上下文空间。请使用 [`AgentDefinition`](/en/agent-sdk/subagents#agentdefinition-configuration) 中的 `tools` 字段，将子Agent的范围限定为所需的最小集合。
* **注意 MCP 服务器开销。** [MCP 工具搜索](/en/agent-sdk/mcp#mcp-tool-search) 默认延迟加载 MCP 工具模式，并按需加载。当工具搜索关闭、启用 Vertex AI 或使用非第一方 `ANTHROPIC_BASE_URL` 时，每个 MCP 服务器都会将其所有工具模式添加到每个请求中，因此少数拥有大量工具的服务器可能会在Agent执行任何工作之前消耗大量上下文信息。
* **对于例行任务，请使用较低的工作量。** 将 [effort](#effort-level) 设置为 `"low"`，即可为仅需读取文件或列出目录的Agent提供服务。这将减少令牌使用量和成本。

有关每个功能上下文成本的详细细分，请参阅[了解上下文成本](/en/features-overview#understand-context-costs)。

## 会议和连续性

每次与 SDK 交互都会创建或延续一个会话。您可以从 `ResultMessage.session_id`（两个 SDK 都可用）获取会话 ID，以便稍后恢复操作。TypeScript SDK 还将其作为初始化 `SystemMessage` 中的一个直接字段公开；在 Python 中，它嵌套在 `SystemMessage.data` 中。

恢复后，之前所有步骤的完整上下文都会被恢复：已读取的文件、已执行的分析以及已采取的操作。您还可以 fork 一个会话，以便在不修改原始会话的情况下尝试不同的方法。

有关恢复、继续和分支模式的完整指南，请参阅[会话管理](/en/agent-sdk/sessions)。

<注>
  在 Python 中，`ClaudeSDKClient` 可以自动处理跨多个调用的会话 ID。详情请参阅[Python SDK 参考](/en/agent-sdk/python#choosing-between-query-and-claudesdkclient)。
</Note>

## 处理结果

循环结束后，`ResultMessage` 会告诉你发生了什么并给出输出结果。`subtype` 字段（两个 SDK 中均可用）是检查终止状态的主要方法。

| 结果子类型 | 发生了什么 | `result` 字段可用吗？ |
| :------------------------------------ | :------------------------------------------------------------------------------- | :-----------------------: |
| `成功` | Claude正常完成了任务 | 是的 |
| `error_max_turns` | 完成前已达到 `maxTurns` 限制 | 否 |
| `error_max_budget_usd` | 完成前已达到 `maxBudgetUsd` 限额 | 否 |
| `error_during_execution` | 执行过程中发生错误，导致循环中断（例如，API 故障或请求取消） | 否 |
| `error_max_structured_output_retries` | 结构化输出验证在达到配置的重试次数限制后失败 | 否 |

`result` 字段（最终文本输出）仅在 `success` 变体中存在，因此在读取它之前务必检查子类型。所有结果子类型都包含 `total_cost_usd`、`usage`、`num_turns` 和 `session_id`，以便您可以跟踪成本并在发生错误后恢复操作。在 Python 中，`total_cost_usd` 和 `usage` 被定义为可选字段，并且在某些错误路径下可能为 `None`，因此在格式化它们之前需要进行保护。有关如何解释 `usage` 字段的详细信息，请参阅[跟踪成本和使用情况](/en/agent-sdk/cost-tracking)。

结果中还包含一个 `stop_reason` 字段（TypeScript 中为 `string | null`，Python 中为 `str | None`），用于指示模型在最后一轮停止生成的原因。常见值包括 `end_turn`（模型正常完成）、`max_tokens`（达到输出标记限制）和 `refusal`（模型拒绝了请求）。对于错误结果子类型，`stop_reason` 的值取自循环结束前最后一个助手响应。要检测拒绝，请检查 `stop_reason === "refusal"`（TypeScript）或 `stop_reason == "refusal"`（Python）。有关完整类型，请参阅 [`SDKResultMessage`](/en/agent-sdk/typescript#sdkresultmessage)（TypeScript）或 [`ResultMessage`](/en/agent-sdk/python#resultmessage)（Python）。

## Hook

[Hook](/en/agent-sdk/hooks) 是在循环中的特定点触发的回调函数：例如工具运行之前、工具返回之后、Agent完成时等等。一些常用的Hook包括：

| Hook | 触发时机 | 常见用途 |
| :------------------------------- | :---------------------------------- | :----------------------------------------- |
| `PreToolUse` | 工具执行前 | 验证输入，阻止危险命令 |
| `PostToolUse` | 工具返回后 | 审核输出，触发副作用 |
| `UserPromptSubmit` | 当发送提示时 | 向提示中注入额外上下文 |
| `停止` | 当Agent完成操作时 | 验证结果，保存会话状态 |
| `SubagentStart` / `SubagentStop` | 当子Agent启动或完成时 | 跟踪并汇总并行任务结果 |
| `PreCompact` | 上下文压缩前 | 在进行摘要之前存档完整文本 |

Hook函数在应用程序进程中运行，而不是在Agent的上下文窗口中运行，因此它们不会消耗上下文。Hook函数还可以绕过循环：例如，一个拒绝工具调用的 `PreToolUse` Hook函数会阻止该调用执行，Claude 会收到拒绝消息。

两个 SDK 都支持上述所有事件。TypeScript SDK 还包含 Python 尚未支持的其他事件。有关完整的事件列表、各 SDK 的可用性以及完整的回调 API，请参阅[使用Hook控制执行](/en/agent-sdk/hooks)。

把所有内容整合起来

此示例将本页的关键概念整合到一个Agent中，该Agent可以修复失败的测试。它配置Agent，包括允许使用的工具（自动批准，以便Agent自主运行）、项目设置以及回合数和推理工作量的安全限制。循环运行时，它会捕获会话 ID 以便必要时恢复，处理最终结果，并打印总成本。

<代码组>
  ```python Python 主题={null}
  导入 asyncio
  从 claude_agent_sdk 导入 query、ClaudeAgentOptions 和 ResultMessage


  async def run_agent():
      session_id = None

      async for message in query(
          prompt="查找并修复导致身份验证模块测试失败的错误"
          options=ClaudeAgentOptions(
              allowed_tools=[
                  “读”，
                  “编辑”，
                  “Bash”，
                  “环球”
                  “Grep”，
              ], # 在此处列出工具会自动批准它们（无需提示）
              setting_sources=[
                  “项目”
              ], # 从当前目录加载 CLAUDE.md、技能和Hook
              max_turns=30，# 防止会话失控
              努力程度="高", # 复杂调试的彻底推理
          ），
      ）：
          # 处理最终结果
          如果 isinstance(message, ResultMessage):
              session_id = message.session_id # 保存以便将来恢复

              如果 message.subtype == "success":
                  print(f"完成：{message.result}")
              elif message.subtype == "error_max_turns":
                  # Agent人回合数已用完。请使用更高的回合数限制继续执行。
                  print(f"已达到转弯次数限制。请恢复会话 {session_id} 以继续。")
              elif message.subtype == "error_max_budget_usd":
                  print("达到预算限制。")
              别的：
                  print(f"已停止：{message.subtype}")
              如果 message.total_cost_usd 不为 None：
                  print(f"成本：${message.total_cost_usd:.4f}")


  asyncio.run(run_agent())
  ```

  ```typescript TypeScript 主题={null}
  import { query } from "@anthropic-ai/claude-agent-sdk";

  let sessionId: string | undefined;

  for await (const message of query({
    提示：“查找并修复导致身份验证模块测试失败的错误”，
    选项： {
      allowedTools: ["读取", "编辑", "Bash", "Glob", "Grep"], // 在此处列出工具会自动批准它们（无需提示）
      settingSources: ["project"], // 从当前目录加载 CLAUDE.md、技能和Hook
      maxTurns: 30, // 防止会话失控
      投入程度：高 // 复杂调试的彻底推理
    }
  })) {
    // 保存会话 ID 以便稍后根据需要恢复会话
    如果 (message.type === "system" && message.subtype === "init") {
      sessionId = message.session_id;
    }

    // 处理最终结果
    如果 (message.type === "result") {
      如果 (message.subtype === "成功") {
        console.log(`完成：${message.result}`);
      } else if (message.subtype === "error_max_turns") {
        // Agent的回合数已用完。请使用更高的回合数限制继续执行。
        console.log(`已达到回合数限制。请恢复会话 ${sessionId} 以继续。`);
      } else if (message.subtype === "error_max_budget_usd") {
        console.log("达到预算上限。");
      } 别的 {
        console.log(`已停止：${message.subtype}`);
      }
      console.log(`成本：$${message.total_cost_usd.toFixed(4)}`);
    }
  }
  ```
</CodeGroup>

后续步骤

既然你已经理解了循环，接下来就根据你要构建的内容，看看该怎么做：

* **还没有运行过Agent？** 首先使用[快速入门](/en/agent-sdk/quickstart)安装SDK，并查看完整的端到端运行示例。
* **准备好接入您的项目了吗？** [加载 CLAUDE.md、技能和文件系统Hook](/en/agent-sdk/claude-code-features)，以便Agent自动遵循您的项目约定。
* **构建交互式用户界面？** 启用[流式传输](/en/agent-sdk/streaming-output)以在循环运行时显示实时文本和工具调用。
* **需要更严格地控​​制Agent可以执行的操作？** 使用 [权限](/en/agent-sdk/permissions) 锁定工具访问权限，并使用 [Hook](/en/agent-sdk/hooks) 在执行工具调用之前对其进行审核、阻止或转换。
* **运行耗时或成本高昂的任务？** 将隔离的工作卸载到 [子Agent](/en/agent-sdk/subagents) 以保持主上下文的精简。

有关Agent Loop的更广泛概念图（非 SDK 特有的），请参阅[Claude Code 的工作原理](/en/how-claude-code-works)。