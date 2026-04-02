# 第 5 章 QueryEngine 会话编排

> 对应源码主线：src/QueryEngine.ts

## 5.1 为什么既有 QueryEngine.ts，又有 query.ts

这是整个工程最关键的结构问题之一。

很多人第一次看会觉得重复：

- QueryEngine.ts
- query.ts

但这两个文件的职责完全不同：

1. QueryEngine.ts 负责会话级编排
2. query.ts 负责单轮智能体循环

为了和全书术语保持一致，这里后文提到的“会话级 orchestration 层”，都可以直接理解为统一运行时里的“会话编排层”。

可以这样理解：

- QueryEngine 解决“这次会话带着什么状态进入一轮任务”
- query.ts 解决“这一轮任务如何不断调用模型与工具直到结束”

## 5.2 QueryEngineConfig：会话装配清单

QueryEngine 的配置对象很长，但恰好说明了它的地位。

它需要知道：

- cwd
- tools
- commands
- mcpClients
- agents
- canUseTool
- AppState 的读写接口
- initialMessages
- readFileCache
- system prompt 相关参数
- model/fallback/thinking/maxTurns/maxBudget/taskBudget
- orphaned permission、SDK status、abortController 等运行态

这表明 QueryEngine 并不是一个“模型 API wrapper”，而是整个会话执行上下文的收口者。

## 5.3 QueryEngine 持有的状态是什么

类内部最关键的字段如下：

```ts
private mutableMessages: Message[]
private abortController: AbortController
private permissionDenials: SDKPermissionDenial[]
private totalUsage: NonNullableUsage
private readFileState: FileStateCache
private discoveredSkillNames = new Set<string>()
private loadedNestedMemoryPaths = new Set<string>()
```

这些字段说明，QueryEngine 真正维护的是跨轮状态：

- 历史消息
- 中断控制器
- 权限拒绝记录
- token 用量累计
- 文件读取缓存
- 技能发现状态
- memory 加载状态

这就是“会话引擎”这个名字的由来。

## 5.4 submitMessage() 是会话级主入口

真正执行用户输入的是 submitMessage()。

它不是把 prompt 直接扔给 query()，而是会先做一整套前置编排。

其中最关键的一步，是包装 canUseTool：

```ts
const wrappedCanUseTool: CanUseToolFn = async (
  tool,
  input,
  toolUseContext,
  assistantMessage,
  toolUseID,
  forceDecision,
) => {
  const result = await canUseTool(tool, input, toolUseContext, assistantMessage, toolUseID, forceDecision)

  if (result.behavior !== 'allow') {
    this.permissionDenials.push({
      type: 'permission_denial',
      tool_name: sdkCompatToolName(tool.name),
      tool_use_id: toolUseID,
      tool_input: input,
    })
  }

  return result
}
```

这里很重要的一点是：权限系统并没有被藏在工具内部，QueryEngine 还会把权限拒绝结果变成会话级可追踪数据。

## 5.5 submitMessage() 先构造系统提示素材，再进入 query()

submitMessage() 会先调用 fetchSystemPromptParts()，拿到：

- defaultSystemPrompt
- userContext
- systemContext

然后根据 customSystemPrompt、appendSystemPrompt、memoryMechanicsPrompt、agent/coordinator 模式等因素，构造最终送入 query() 的系统提示词。

这里体现出一个分层思想：

- prompt 素材获取在 QueryEngine
- 单轮循环在 query.ts

所以 QueryEngine 是“把本轮 query 所需世界状态准备好”的地方。

## 5.6 QueryEngine 的一个核心价值：把 REPL 与 headless/SDK 路径统一起来

源码注释里已经点明：

“One QueryEngine per conversation. Each submitMessage() call starts a new turn within the same conversation.”

这意味着 QueryEngine 是可以同时服务：

- REPL 交互模式
- SDK/headless 模式

也就是说，它把“会话编排逻辑”从 UI 路径里剥离出来了。

这一步对大型工程非常重要，因为否则：

- REPL 路径会有一套逻辑
- print/sdk 又会复制一套逻辑

而 QueryEngine 的存在，就是把两者抽成共享核心。

## 5.7 QueryEngine 不是纯函数，因为它必须记住会话

submitMessage() 是 async generator，但 QueryEngine 本身并不是纯函数风格。

这是必要的，因为会话需要跨轮保存：

- mutableMessages
- totalUsage
- permissionDenials
- file cache

如果每轮都完全无状态重建，那么 resume、usage、文件历史、权限追踪这些能力都会变得非常昂贵或者根本做不到。

所以 QueryEngine 是“状态化 orchestrator”，这是合理设计，不是工程妥协。

## 5.8 QueryEngine 和 query.ts 的调用边界

可以把它们的边界理解成下面这样：

### QueryEngine 负责

- 把用户输入标准化
- 组装 system/user context
- 组装 prompt
- 组装工具、命令、MCP、agent、permissions
- 跟踪 session 级 usage 和 denial
- 持久化 transcript 与缓存

### query.ts 负责

- 发起单轮模型调用
- 处理流式响应
- 执行 tool_use
- 把 tool_result 回灌给模型
- 处理 compact、fallback、error recovery

只有把这条边界看清楚，后面读 query.ts 才不容易混。

## 5.9 这一章的阅读结论

读完 QueryEngine.ts，要建立三个认识：

1. QueryEngine 是会话级 orchestration 层，不是模型 API 包装层。
2. 它负责把“一个任务请求”提升成“带着会话状态、上下文、权限、工具池的一轮智能体执行”。
3. 它的存在，是 REPL、SDK、resume、usage、权限追踪这些能力能够共用核心逻辑的关键。

下一章进入 query.ts，这才是 Claude Code 真正意义上的 agentic loop 内核。

## 5.10 submitMessage() 的函数级执行链

如果要真正读懂 QueryEngine，不能只停留在“它会调用 query()”。

submitMessage() 内部其实有一条非常清楚的函数级时序：

1. 读取 config，建立这轮执行所需的运行参数
2. 清空 discoveredSkillNames，重置本轮技能发现状态
3. 包装 canUseTool，把权限拒绝记录到 permissionDenials
4. 调用 fetchSystemPromptParts()，把 defaultSystemPrompt、userContext、systemContext 取回来
5. 根据 customSystemPrompt、appendSystemPrompt、memoryMechanicsPrompt 拼出最终 systemPrompt
6. 构造第一版 processUserInputContext
7. 如果存在 orphanedPermission，先走 handleOrphanedPermission()
8. 调用 processUserInput()，把本次用户输入转成 messagesFromUserInput
9. 先把用户消息写入 transcript，保证 query 还没真正开始时也能 resume
10. 更新 toolPermissionContext，处理 allowedTools
11. 预取 skills 和已启用 plugins，yield 一条 buildSystemInitMessage()
12. 最终进入 for await (const message of query(...)) 主循环

这条链路说明，QueryEngine 真正做的是“把一条原始输入升级成一个可运行的 agent turn”。

## 5.11 为什么要先 processUserInput()，再进 query()

很多人一开始会下意识认为：

- 用户输一段话
- 直接进 query()

但源码不是这么做的。

QueryEngine 会先调用：

```ts
const {
  messages: messagesFromUserInput,
  shouldQuery,
  allowedTools,
  model: modelFromUserInput,
  resultText,
} = await processUserInput({ ... })
```

这一步的意义非常大，因为用户输入在 Claude Code 里并不只是“文本 prompt”，它可能是：

- 普通消息
- slash command
- 带附件消息
- 带本地命令输出的消息
- 改写 model / effort / allowed tools 的控制消息

所以 processUserInput() 的作用就是先把“用户操作”解释成“可进入 query loop 的标准消息与控制参数”。

这就是 QueryEngine 和 query.ts 的天然边界。

## 5.12 transcript 为什么要在 query 前先写一遍

这一段是 QueryEngine 里非常关键的工程细节：

```ts
if (persistSession && messagesFromUserInput.length > 0) {
  const transcriptPromise = recordTranscript(messages)
  ...
}
```

源码注释已经把原因说得很透：

如果用户消息已经被系统接受，但 API 还没返回、进程就被杀掉，那么如果 transcript 里还没有这条用户消息，resume 时就会找不到这次对话的有效起点。

因此，QueryEngine 的策略是：

- 用户消息一旦被接受，就尽早持久化
- 不把 transcript 记录完全依赖在后续 assistant 返回上

这是一个非常成熟的恢复性设计。

## 5.13 buildSystemInitMessage() 的意义

在真正进入 query() 之前，QueryEngine 还会先 yield 一条 system init 消息：

```ts
yield buildSystemInitMessage({
  tools,
  mcpClients,
  model: mainLoopModel,
  permissionMode: initialAppState.toolPermissionContext.mode as PermissionMode,
  commands,
  agents,
  skills,
  plugins: enabledPlugins,
  fastMode: initialAppState.fastMode,
})
```

这条消息的意义不是给模型看的，而是给外部消费者看的。

比如：

- REPL / Desktop / Remote client 需要知道本轮会话启动时有哪些能力
- SDK/headless 调用方需要在真正输出前拿到当前运行环境摘要

它相当于会话的“运行时能力声明”。

## 5.14 QueryEngine 与 query() 的真正接缝

真正进入 query loop 的代码如下：

```ts
for await (const message of query({
  messages,
  systemPrompt,
  userContext,
  systemContext,
  canUseTool: wrappedCanUseTool,
  toolUseContext: processUserInputContext,
  fallbackModel,
  querySource: 'sdk',
  maxTurns,
  taskBudget,
})) {
  ...
}
```

这里可以清楚看到两层系统的接缝：

- QueryEngine 准备 messages、systemPrompt、user/system context、permissions、toolUseContext
- query() 只接收这些已经准备好的输入，然后进入单轮循环

所以 QueryEngine 并不会深入 query loop 细节，它的工作在这里基本完成。

## 5.15 QueryEngine 在 query() 返回之后还要做什么

很多读者只注意到“它调了 query()”，但后半段同样重要。

在 for-await 消费 query() 产出时，QueryEngine 还在持续做这些事情：

- 把 assistant、user、compact boundary 写回 mutableMessages
- 把 progress、attachment、api_retry 等事件规范化后转给 SDK/上层
- 累计 usage
- 记录 lastStopReason
- 在 compact boundary 到来时裁剪 mutableMessages，释放旧消息
- 在 max turns reached 或 structured_output 等 attachment 上生成最终结果对象

也就是说，QueryEngine 不只是“调用 query 的上游”，它还是 query 产物的会话级归档和归一化层。

## 5.16 这一章和后续章节怎么衔接

第 5 章在全书里的位置非常关键，因为它正好卡在“上下文装配”和“执行循环”之间：

1. 它承接第 4 章，把上下文系统提供的素材真正组织进一个可持续多轮运行的会话。
2. 它向下游连接第 6 章，因为 `query.ts` 只处理单轮循环，而这一章先把进入循环前的世界状态准备好。
3. 它也会回流到第 7 章和第 10 章，因为工具池、权限包装、processUserInput、REPL 提交链最后都要在这里收口成一次 `submitMessage()`。

所以如果说第 1 章给出了总线图，第 5 章就是第一次真正把这张总线图变成“会话编排接口”的地方。
