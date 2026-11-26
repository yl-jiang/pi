# @earendil-works/pi-agent-core

带工具执行和事件流的有状态 agent。构建于 `@earendil-works/pi-ai` 之上。

## 安装

```bash
npm install @earendil-works/pi-agent-core
```

## 快速开始

```typescript
import { Agent } from "@earendil-works/pi-agent-core";
import { createModels } from "@earendil-works/pi-ai";
import { anthropicProvider } from "@earendil-works/pi-ai/providers/anthropic";

const models = createModels();
models.setProvider(anthropicProvider());
const model = models.getModel("anthropic", "claude-sonnet-4-6");
if (!model) throw new Error("Model not found");

const agent = new Agent({
  initialState: {
    systemPrompt: "You are a helpful assistant.",
    model,
  },
  streamFn: models.streamSimple.bind(models),
});

agent.subscribe((event) => {
  if (event.type === "message_update" && event.assistantMessageEvent.type === "text_delta") {
    // 只流式输出新增的文本块
    process.stdout.write(event.assistantMessageEvent.delta);
  }
});

await agent.prompt("Hello!");
```

## 核心概念

### AgentMessage 与 LLM Message

agent 处理的是 `AgentMessage`，一种灵活的类型，可以包含：
- 标准 LLM 消息（`user`、`assistant`、`toolResult`）
- 通过声明合并（declaration merging）扩展的自定义应用专属消息类型

LLM 只认识 `user`、`assistant`、`toolResult`。`convertToLlm` 函数在每次调用 LLM 之前过滤并转换消息，衔接这个差异。

### 消息流转

```
AgentMessage[] → transformContext() → AgentMessage[] → convertToLlm() → Message[] → LLM
                    (可选)                                (必需)
```

1. **transformContext**：裁剪旧消息、注入外部上下文
2. **convertToLlm**：过滤掉仅用于 UI 的消息，将自定义类型转换为 LLM 格式

## 事件流

agent 会发出事件用于 UI 更新。理解事件发生的顺序有助于构建响应式界面。

### prompt() 的事件顺序

当你调用 `prompt("Hello")` 时：

```
prompt("Hello")
├─ agent_start
├─ turn_start
├─ message_start   { message: userMessage }      // 你的 prompt
├─ message_end     { message: userMessage }
├─ message_start   { message: assistantMessage } // LLM 开始回复
├─ message_update  { message: partial... }       // 流式输出的片段
├─ message_update  { message: partial... }
├─ message_end     { message: assistantMessage } // 完整回复
├─ turn_end        { message, toolResults: [] }
└─ agent_end        { messages: [...] }
```

### 带工具调用的情况

如果 assistant 调用了工具，循环会继续：

```
prompt("Read config.json")
├─ agent_start
├─ turn_start
├─ message_start/end  { userMessage }
├─ message_start      { assistantMessage with toolCall }
├─ message_update...
├─ message_end        { assistantMessage }
├─ tool_execution_start  { toolCallId, toolName, args }
├─ tool_execution_update { partialResult }           // 如果工具支持流式输出
├─ tool_execution_end    { toolCallId, result }
├─ message_start/end  { toolResultMessage }
├─ turn_end           { message, toolResults: [toolResult] }
│
├─ turn_start                                        // 下一轮
├─ message_start      { assistantMessage }           // LLM 针对工具结果作出回应
├─ message_update...
├─ message_end
├─ turn_end
└─ agent_end
```

工具执行模式是可配置的：

- `parallel`（默认）：依次对工具调用做预检（preflight），并发执行被允许的工具，每个工具定稿后立即发出 `tool_execution_end`，再按 assistant 消息中的原始顺序发出 toolResult 消息和 `turn_end.toolResults`
- `sequential`：逐个执行工具调用，与历史行为一致

在 parallel 模式下，工具完成事件按工具实际完成的顺序触发，但持久化的 toolResult 消息仍然按 assistant 消息中的原始顺序排列。

该模式可以通过 agent 配置中的 `toolExecution` 全局设置，也可以通过 `AgentTool` 上的 `executionMode` 针对单个工具设置。如果一批工具调用中任意一个指向被标记为 `executionMode: "sequential"` 的工具，那么无论全局设置如何，整批都会按顺序执行。

`beforeToolCall` 钩子在 `tool_execution_start` 和参数校验完成之后运行。它可以阻止执行，并在被阻止的结果上附加 `terminate: true`。`afterToolCall` 钩子在工具执行完成之后、`tool_execution_end` 和最终的工具结果消息事件发出之前运行。

工具本身、被 `beforeToolCall` 阻止的结果，以及 `afterToolCall` 的覆盖结果，都可以返回 `terminate: true`，用来提示自动的后续 LLM 调用应该被跳过。只有当这一批中每一个已定稿的工具结果都设置了 `terminate: true` 时，循环才会提前停止。混合批次（部分 terminate、部分不 terminate）会正常继续。

当你使用 `Agent` 类时，assistant 的 `message_end` 处理会被当作一个屏障（barrier），在工具预检开始之前完成。这意味着 `beforeToolCall` 看到的 agent 状态已经包含了那条请求工具调用的 assistant 消息。

### 请求准备与轮次收尾

`prepareRequest` 会在每一次对话型 provider 请求之前立即运行，包括第一次请求。可以用它在排队输入发出之后安装标准的持久化上下文：

```typescript
agent.prepareRequest = async ({ context }) => ({
  context: { ...context, messages: await session.loadModelContext() },
});
```

`prepareRequest` 运行期间不会轮询队列。在它运行时排入的 steering 消息，会等到下一次正常的 steering 轮询时才被处理。

`finishTurn` 会在 assistant 消息和所有工具结果都定稿之后、但在 `turn_end` 之前运行。无论是正常响应、出错还是被中止，它都会运行：

```typescript
agent.finishTurn = async ({ message }) => {
  if (message.stopReason === "error" || message.stopReason === "aborted") return;
  if (shouldEndRun(message)) return { action: "end" };
  return needsAnotherResponse(message) ? { action: "continue" } : undefined;
};
```

返回 `undefined` 会保留正常的调度逻辑。`{ action: "end" }` 会在 `turn_end` 之后立即停止，不再轮询 steering 或 follow-up 队列，也不会准备下一次请求。对于正常响应，`{ action: "continue" }` 会确保紧接着发出一次请求。如果工具结果、steering 或 follow-up 本身已经会触发这次请求，那么它们就满足了这个决定，不会产生额外的请求；否则循环会发出一次只带上下文、不带新输入的请求。出错和被中止的响应仍然是硬性退出，它们的决定会被忽略。`finishTurn` 会在下一次请求之后再次运行，所以无条件返回 `{ action: "continue" }` 会造成死循环。

要从已移除的 `shouldStopAfterTurn` 迁移过来，返回 `{ action: "end" }` 即可。为保留旧钩子"只在正常响应时调用"的行为，尤其是当判断逻辑带有副作用或假定响应一定成功时，需要对出错和中止的响应加以保护：

```typescript
finishTurn: async (turn, signal) => {
  if (turn.message.stopReason === "error" || turn.message.stopReason === "aborted") return;
  return (await shouldStop(turn, signal)) ? { action: "end" } : undefined;
},
```

每一次 provider 轮次都遵循这样的生命周期：

```text
选定的输入事件
→ prepareRequest
→ provider 响应
→ 工具结果
→ finishTurn
→ turn_end
→ 既有的继续调度逻辑，或者 agent_end
```

### continue() 与排队输入

`continue()` 保留原有的队列行为。空的或仅含系统消息的转录会被拒绝，且不消费队列。当末尾不是 assistant 消息时，会从现有上下文继续：启动时会轮询 steering，而 follow-up 输入会等到响应自然停止后才被处理。

```typescript
agent.followUp({ role: "user", content: "After the retry", timestamp: Date.now() });
await agent.continue(); // 第一次请求会重试已有的 user/toolResult 尾部消息。
```

末尾是 assistant 消息时不能直接发送，所以 `continue()` 会回退为：先取一批排队的 steering 消息，再取一批排队的 follow-up 消息。队列模式仍然控制这批被选中的消息是只含一条还是含全部：

```typescript
agent.steer({ role: "user", content: "Continue from here", timestamp: Date.now() });
await agent.continue(); // 因为末尾是 assistant 消息，所以只使用排队中的这一条消息。
```

### 事件类型

| 事件 | 描述 |
|-------|-------------|
| `agent_start` | agent 开始处理 |
| `agent_end` | 本次运行的最后一个事件。被 await 的订阅者在此事件上完成后才计入"已结算" |
| `turn_start` | 新一轮开始（一次 LLM 调用加若干次工具执行） |
| `turn_end` | 一轮结束，带有 assistant 消息和工具结果 |
| `message_start` | 任意消息开始（user、assistant、toolResult） |
| `message_update` | **仅 assistant。** 包含带有 delta 的 `assistantMessageEvent` |
| `message_end` | 消息完成 |
| `tool_execution_start` | 工具开始执行 |
| `tool_execution_update` | 工具流式输出进度 |
| `tool_execution_end` | 工具执行完成 |

`Agent.subscribe()` 的监听器按注册顺序依次被 await。`agent_end` 意味着循环不会再发出更多事件，但 `await agent.waitForIdle()` 和 `await agent.prompt(...)` 只会在被 await 的 `agent_end` 监听器完成之后才会结算完毕。

## Agent 选项

```typescript
const agent = new Agent({
  // 初始状态。除非 messages 已经以一条系统消息开头，否则 systemPrompt 和 tools
  // 会组合成最前面的系统消息。
  initialState: {
    systemPrompt: string,
    model: Model<any>,
    thinkingLevel: "off" | "minimal" | "low" | "medium" | "high" | "xhigh" | "max",
    tools: AgentTool<any>[],
    messages: AgentMessage[],
  },

  // 将 AgentMessage[] 转换为 LLM Message[]（如果有自定义消息类型则为必需）
  convertToLlm: (messages) => messages.filter(...),

  // 在 convertToLlm 之前转换上下文（用于裁剪、压缩）
  transformContext: async (messages, signal) => pruneOldMessages(messages),

  // Steering 模式："one-at-a-time"（默认）或 "all"
  steeringMode: "one-at-a-time",

  // Follow-up 模式："one-at-a-time"（默认）或 "all"
  followUpMode: "one-at-a-time",

  // 必需的流式函数。接收一个 TranscriptContext：prompt 和 tools
  // 位于转录（transcript）的系统消息中，而不在 context 上。
  streamFn: models.streamSimple.bind(models),

  // 用于 provider 缓存的会话 ID
  sessionId: "session-123",

  // 请求如何向 provider 标识当前应用（pi-ai 的 ClientIdentity），会合并覆盖
  // DEFAULT_AGENT_IDENTITY（{ attribution: PI_ATTRIBUTION }：provider 归属头，名称为 pi）。
  // 传入你自己应用的 attribution，或传 null 表示不发送任何归属信息。
  identity: { attribution: { name: "my-app", url: "https://my-app.example" } },

  // 动态 API key 解析（用于会过期的 OAuth token）
  getApiKey: async (provider) => refreshToken(),

  // 工具执行模式："parallel"（默认）或 "sequential"
  toolExecution: "parallel",

  // 在参数校验之后对每个工具调用做预检。可以阻止执行。
  beforeToolCall: async ({ toolCall, args, context }) => {
    if (toolCall.name === "bash") {
      return { block: true, reason: "bash is disabled", terminate: true };
    }
  },

  // 在工具执行完成之后、最终的工具事件发出之前，对每个工具结果做后处理。
  afterToolCall: async ({ toolCall, result, isError, context }) => {
    if (toolCall.name === "notify_done" && !isError) {
      return { terminate: true };
    }
    if (!isError) {
      return { details: { ...result.details, audited: true } };
    }
  },

  // 在每次 provider 请求之前立即重建已定稿的上下文。
  prepareRequest: async ({ context }, signal) => {
    return { context: { ...context, messages: await loadCanonicalMessages(signal) } };
  },

  // 在 turn_end 发出之前，完成一轮的收尾工作。
  // `continue` 确保紧接着发出一次请求；已有的工具/队列调度可以满足这个要求。
  // `end` 会在 turn_end 之后结束本次运行，不再轮询队列。
  finishTurn: async ({ message, toolResults }, signal) => {
    return shouldContinue(message, toolResults) ? { action: "continue" } : undefined;
  },

  // 基于 token 的 provider 所使用的自定义思考预算
  thinkingBudgets: {
    minimal: 128,
    low: 512,
    medium: 1024,
    high: 2048,
  },
});
```

## Agent 状态

```typescript
interface AgentState {
  model: Model<any>;
  thinkingLevel: ThinkingLevel;
  tools: AgentTool<any>[];
  messages: AgentMessage[];
  readonly isStreaming: boolean;
  readonly streamingMessage?: AgentMessage;
  readonly pendingToolCalls: ReadonlySet<string>;
  readonly errorMessage?: string;
}
```

通过 `agent.state` 访问状态。

赋值 `agent.state.tools = [...]` 或 `agent.state.messages = [...]` 会先复制顶层数组再存储。对返回数组的直接修改，会改变当前 agent 的状态。

转录（transcript）拥有系统提示词和工具声明：最前面的系统消息就是 prompt，后续的系统消息会对它打补丁（参见 pi-ai 中的 `SystemMessage`）。`agent.state.systemPrompt` 是只读的，它是对转录的重放结果。`agent.state.tools` 是实际可执行的工具装载集；在每次请求之前，循环会把它和转录中声明的工具做差异对比，如果不同，就会在一条系统消息中宣布这个变化（如果已经存在一条待处理的系统消息，则合并进去）。pi-ai 的 `getCurrentSystemMessage(messages)` 对任意消息数组（包括带自定义角色的 agent 转录）都会返回重放出来的最前消息，包括其中声明的工具。

要在对话中途更改 prompt，可以追加一条带有 `content`（追加的说明）或 `sections`（按名字替换的部分）的系统消息：

```typescript
await agent.prompt([
  { role: "system", content: "", sections: { skills: "<skills>...</skills>" }, timestamp: Date.now() },
  { role: "user", content: "Continue", timestamp: Date.now() },
]);
```

在流式输出期间，`agent.state.streamingMessage` 包含当前这条未完成的 assistant 消息。

`agent.state.isStreaming` 会保持为 `true`，直到本次运行完全结算完毕，包括被 await 的 `agent_end` 订阅者。

## 方法

### 发起 Prompt

```typescript
// 文本 prompt
await agent.prompt("Hello");

// 带图片
await agent.prompt("What's in this image?", [
  { type: "image", data: base64Data, mimeType: "image/jpeg" }
]);

// 直接传 AgentMessage
await agent.prompt({ role: "user", content: "Hello", timestamp: Date.now() });

// 继续已有的非 assistant 输入；如果末尾是 assistant 消息，可以回退为使用排队的输入
await agent.continue();
```

### 状态管理

```typescript
agent.state.model = getModel("openai", "gpt-4o");
agent.state.thinkingLevel = "medium";
agent.state.tools = [myTool];
agent.toolExecution = "sequential";
agent.beforeToolCall = async ({ toolCall }) => undefined;
agent.afterToolCall = async ({ toolCall, result }) => undefined;
agent.prepareRequest = async ({ context }) => ({
  context: { ...context, messages: await loadCanonicalMessages() },
});
agent.finishTurn = async () => undefined;
agent.state.messages = newMessages; // 顶层数组会被复制
agent.state.messages.push(message);
const nextQueuedMessages = agent.peekQueuedMessages(); // 遵循队列模式；不会消费队列
agent.reset();
```

### 会话与思考预算

```typescript
agent.sessionId = "session-123";

agent.thinkingBudgets = {
  minimal: 128,
  low: 512,
  medium: 1024,
  high: 2048,
};
```

### 控制

```typescript
agent.abort();           // 取消当前操作
await agent.waitForIdle(); // 等待完成
```

### 事件

```typescript
const unsubscribe = agent.subscribe(async (event, signal) => {
  if (event.type === "agent_end") {
    // 本次运行最后的屏障性收尾工作
    await flushSessionState(signal);
  }
});
unsubscribe();
```

## Steering 与 Follow-up

Steering 消息让你可以在工具还在运行时打断 agent。Follow-up 消息让你可以在 agent 原本会停下来的时候继续排队新的工作。

```typescript
agent.steeringMode = "one-at-a-time";
agent.followUpMode = "one-at-a-time";

// 在 agent 正在执行工具时
agent.steer({
  role: "user",
  content: "Stop! Do this instead.",
  timestamp: Date.now(),
});

// 在 agent 完成当前工作之后
agent.followUp({
  role: "user",
  content: "Also summarize the result.",
  timestamp: Date.now(),
});

const steeringMode = agent.steeringMode;
const followUpMode = agent.followUpMode;

agent.clearSteeringQueue();
agent.clearFollowUpQueue();
agent.clearAllQueues();
```

使用 clearSteeringQueue、clearFollowUpQueue 或 clearAllQueues 来丢弃排队中的消息。

当一轮结束之后检测到 steering 消息时：
1. 当前 assistant 消息中的所有工具调用都已经执行完毕
2. steering 消息被注入
3. LLM 在下一轮中作出回应

Follow-up 消息只有在没有更多工具调用、也没有 steering 消息时才会被检查。如果有排队的 follow-up 消息，它们会被注入，并再运行一轮。

## 自定义消息类型

通过声明合并（declaration merging）扩展 `AgentMessage`：

```typescript
declare module "@earendil-works/pi-agent-core" {
  interface CustomAgentMessages {
    notification: { role: "notification"; text: string; timestamp: number };
  }
}

// 现在这是合法的
const msg: AgentMessage = { role: "notification", text: "Info", timestamp: Date.now() };
```

在 `convertToLlm` 中处理自定义类型：

```typescript
const agent = new Agent({
  streamFn: models.streamSimple.bind(models),
  convertToLlm: (messages) => messages.flatMap(m => {
    if (m.role === "notification") return []; // 过滤掉
    return [m];
  }),
});
```

## 工具

使用 `AgentTool` 定义工具。`parameters` 可以是任意 JSON Schema 对象；如果用 `@earendil-works/pi-schema` 的构建器来构建，它还会为 `params` 提供类型（其他 schema 会得到 `unknown`，`AgentTool<any>` 会得到 `any`）：

```typescript
import { obj, str } from "@earendil-works/pi-schema";

const readFileParameters = obj({
  path: str({ description: "File path" }),
});

const readFileTool: AgentTool<typeof readFileParameters> = {
  name: "read_file",
  label: "Read File",  // 用于 UI 显示
  description: "Read a file's contents",
  parameters: readFileParameters,
  // 针对该工具单独覆盖执行模式（可选）。
  // "sequential" 会强制整批都按顺序执行。
  // "parallel" 允许与其它工具调用并发执行。
  // 如果省略，则沿用全局的 toolExecution 配置。
  executionMode: "sequential",
  execute: async (toolCallId, params, signal, onUpdate) => {
    const content = await fs.readFile(params.path, "utf-8");

    // 可选：流式输出进度
    onUpdate?.({ content: [{ type: "text", text: "Reading..." }], details: {} });

    // 可选：在这里加上 `terminate: true`，当这批中每一个已定稿的工具结果
    // 都这样设置时，可以跳过自动的后续 LLM 调用
    return {
      content: [{ type: "text", text: content }],
      details: { path: params.path, size: content.length },
    };
  },
};

agent.state.tools = [readFileTool];
```

### 错误处理

工具失败时**抛出一个错误**。不要把错误信息作为 content 返回。

```typescript
execute: async (toolCallId, params, signal, onUpdate) => {
  if (!fs.existsSync(params.path)) {
    throw new Error(`File not found: ${params.path}`);
  }
  // 只在成功时返回 content
  return { content: [{ type: "text", text: "..." }] };
}
```

抛出的错误会被 agent 捕获，并作为带有 `isError: true` 的工具错误上报给 LLM。

从 `execute()`、被阻止的 `beforeToolCall`、或 `afterToolCall` 返回 `terminate: true`，可以提示 agent 在当前这批工具执行完之后停止。只有当这一批中每一个已定稿的工具结果都带有这个标记时，这个提示才会真正生效。这个提示只在运行时起作用；发出的 `toolResult` 转录消息仍然是标准的 LLM 工具结果。

### MCP 与 Codemode

`@earendil-works/pi-mcp` 用于连接 MCP 服务器，`@earendil-works/pi-codemode` 用于运行模型编写、调用工具的 JavaScript 代码。[examples/mcp-codemode](examples/mcp-codemode) 把两者都包装成了 `AgentTool`：每个 MCP 工具对应一个工具，外加一个 `codemode` 工具，它的脚本通过 `runToolCall()` 调用 agent 的工具，所以 `beforeToolCall` 和 `afterToolCall` 也会作用于这些调用。

## 代理（Proxy）用法

适用于通过后端做代理的浏览器应用：

```typescript
import { Agent, streamProxy } from "@earendil-works/pi-agent-core";

const agent = new Agent({
  streamFn: (model, context, options) =>
    streamProxy(model, context, {
      ...options,
      authToken: "...",
      proxyUrl: "https://your-server.com",
    }),
});
```

## 底层 API

如果不想用 `Agent` 类而需要直接控制：

```typescript
import { agentLoop, agentLoopContinue } from "@earendil-works/pi-agent-core";

const context: AgentContext = {
  messages: [{ role: "system", content: "You are helpful.", timestamp: Date.now() }],
  tools: [],
};

const config: AgentLoopConfig = {
  model: getModel("openai", "gpt-4o"),
  convertToLlm: (msgs) => msgs.filter(m => ["user", "assistant", "toolResult"].includes(m.role)),
  toolExecution: "parallel",  // 如果设置了单个工具的 executionMode，则以其为准
  beforeToolCall: async ({ toolCall, args, context }) => undefined,
  afterToolCall: async ({ toolCall, result, isError, context }) => undefined,
};

const userMessage = { role: "user", content: "Hello", timestamp: Date.now() };

const streamFn = models.streamSimple.bind(models);
for await (const event of agentLoop([userMessage], context, config, undefined, streamFn)) {
  console.log(event.type);
}

// 从已有上下文继续
for await (const event of agentLoopContinue(context, config, undefined, streamFn)) {
  console.log(event.type);
}
```

这些底层的流是只读观测性的。它们会保留事件顺序，但不会在后续的生产者阶段继续执行之前，等待你的异步事件处理逻辑完成结算。如果你需要消息处理在工具预检开始之前起到屏障（barrier）的作用，请使用 `Agent` 类，而不是直接用 `agentLoop()` 或 `agentLoopContinue()`。

## 许可证

MIT
