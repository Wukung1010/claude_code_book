# 第 6 章 query.ts 智能体循环

> 对应源码主线：src/query.ts，配合 docs/conversation/the-loop.mdx

## 6.1 真正的 Agent 内核在这里

如果只能选一个文件来代表 Claude Code 的“智能体本体”，那就是 query.ts。

原因很简单：

这里不是做配置，也不是做 UI，而是在做下面这件事：

1. 组织本轮消息
2. 请求模型
3. 流式接收输出
4. 识别 tool_use
5. 执行工具
6. 把结果回灌给模型
7. 直到本轮任务真正完成

这就是 Agentic Loop。

## 6.2 query() 和 queryLoop() 的关系

query() 本身只是一个薄封装：

```ts
export async function* query(params: QueryParams) {
  const consumedCommandUuids: string[] = []
  const terminal = yield* queryLoop(params, consumedCommandUuids)
  for (const uuid of consumedCommandUuids) {
    notifyCommandLifecycle(uuid, 'completed')
  }
  return terminal
}
```

真正的循环在 queryLoop() 里。

这个设计很干净：

- 外层处理生命周期收尾
- 内层负责无限循环本体

## 6.3 queryLoop() 的 state 设计非常关键

state 里保存了整个循环跨迭代需要延续的信息：

```ts
type State = {
  messages: Message[]
  toolUseContext: ToolUseContext
  autoCompactTracking: AutoCompactTrackingState | undefined
  maxOutputTokensRecoveryCount: number
  hasAttemptedReactiveCompact: boolean
  maxOutputTokensOverride: number | undefined
  pendingToolUseSummary: Promise<ToolUseSummaryMessage | null> | undefined
  stopHookActive: boolean | undefined
  turnCount: number
  transition: Continue | undefined
}
```

这说明 query.ts 并不是“每轮从头算起”，而是一个有连续状态的执行循环。

尤其重要的是：

- compact 状态要延续
- 输出截断恢复次数要延续
- stop hook 状态要延续
- toolUseContext 可能会在循环中变化

## 6.4 每轮循环的第一步：上下文预处理

queryLoop() 在每一轮真正调用模型之前，会先处理消息上下文。

它会串起一系列上下文优化/压缩逻辑，包括：

- tool result budget
- auto compact
- post compact message rebuild
- memory attachment 处理
- system/user context 拼接

这里体现出一个非常核心的工程判断：

对 Agent 来说，“如何在有限窗口里保住有效上下文”与“如何调用模型”同样重要。

## 6.5 流式响应处理是这个文件的核心难点

query.ts 不是拿到一个完整响应后再处理，而是直接消费流式事件。

这意味着它在一轮请求里要同时处理：

1. assistant 文本输出
2. thinking block
3. redacted thinking
4. tool_use block
5. API 错误
6. fallback 触发
7. 中途中止

这也是为什么文件很长，因为它实现的是“流式状态机”，不是普通 RPC。

## 6.6 tool_use 的处理逻辑，决定了它是 Agent 而不是 Chat

query.ts 最关键的分叉是：模型返回的 assistant message 里是否包含 tool_use。

如果没有 tool_use，那么这轮可能结束。

如果有 tool_use，那么系统不会把这一轮当成“回答完成”，而是进入工具执行路径。

源码中与工具执行直接关联的关键导入是：

```ts
import { StreamingToolExecutor } from './services/tools/StreamingToolExecutor.js'
import { runTools } from './services/tools/toolOrchestration.js'
```

这说明工具执行既支持流式并行，也支持常规编排。

也就是说，Claude Code 不是“让模型说要调用什么工具”，而是真的把工具系统嵌进了循环本体。

## 6.7 为什么要有 yieldMissingToolResultBlocks()

query.ts 前面有一个生成器函数：yieldMissingToolResultBlocks()。

它的职责是在异常情况下，为已经发出的 tool_use 合成对应的 tool_result 错误块。

这背后的原因是：

模型发出了 tool_use，就意味着对话轨迹进入了“等待工具返回”的状态。

如果此时中断、失败、fallback，不把这些 tool_use 补成 tool_result，整条 assistant trajectory 就是不闭合的，后面恢复和对话一致性都会出问题。

这类细节正是 Agent 工程和普通聊天工程的差别。

## 6.8 输出截断恢复与 Prompt Too Long 恢复

query.ts 内部非常重视两类恢复：

1. max_output_tokens
2. prompt too long

为什么？因为 Agent 的任务链往往很长，这两个问题是生产环境里最常见的中断源。

源码里用这几个字段和逻辑控制恢复：

- maxOutputTokensRecoveryCount
- hasAttemptedReactiveCompact
- isPromptTooLongMessage()
- isWithheldMaxOutputTokens()

设计重点不是“抛错”，而是“先看是否还能继续把任务做完”。

这体现出 Agent 系统的一条核心原则：

优先恢复执行，实在不行再失败退出。

## 6.9 compact 在 query.ts 里不是附属能力，而是主循环的一部分

query.ts 引入了大量 compact 相关逻辑：

- autoCompact
- buildPostCompactMessages
- reactiveCompact
- contextCollapse
- snip compact

这说明上下文压缩不是后台优化，而是每轮循环都要考虑的主路径问题。

原因很现实：

- Agent 对话轮数长
- tool result 很容易膨胀
- system prompt 本身也不短

如果没有这一层，Claude Code 很快就会在复杂任务里卡死在上下文窗口上。

## 6.10 query.ts 的本质：以消息为状态，以工具为动作，以模型为规划器

如果抽象地看，这个文件其实定义了一台状态机：

- 状态：messages + toolUseContext + compact tracking + recovery counters
- 规划器：模型
- 动作执行器：tools
- 观察结果：tool_result / assistant stream / stop hooks / API errors

所以 Claude Code 的 agentic loop 并不是“模型自己在循环”，而是：

宿主程序维护循环，模型只负责在每一步给出下一阶段的规划与输出。

## 6.11 这一章的阅读结论

对 query.ts 的正确理解应该是：

1. 这是单轮任务的执行内核。
2. 它通过流式消费 assistant 输出，把 tool_use 真正转成工具执行，再把结果回灌。
3. 它同时承担上下文压缩、错误恢复、fallback、中断处理等生产级职责。
4. 它是 Claude Code 从“聊天程序”变成“Agent 系统”的根本所在。

下一章转去看 tools.ts 和权限系统，因为没有受控工具池，这个循环就无法安全地落地到真实操作。

## 6.12 queryLoop() 的单轮结构可以拆成五段

如果把 queryLoop() 的 while(true) 主体压成流程图，最适合记忆的是下面五段：

1. 读取上一轮 state
2. 处理上下文与压缩链
3. 发起流式模型请求
4. 执行工具并合并结果
5. 决定 return 还是 continue

也就是说，这个循环不是“只有模型调用”，而是“模型调用前后各有一层宿主系统逻辑”。

## 6.13 上下文预处理链的顺序不能随便换

query.ts 中一个特别值得精读的点，是上下文处理顺序非常讲究。

从源码注释看，主顺序大致是：

1. tool result budget
2. snip
3. microcompact
4. context collapse
5. autocompact

这里的关键不是“有五个处理器”，而是它们的相对顺序决定了最终行为。

例如源码明确提到：

- microcompact 要在 autocompact 前面
- context collapse 也要在 autocompact 前面

原因是如果前面的轻量裁剪已经把上下文拉回阈值内，就不应该再触发更重的压缩，否则会过度损失细节。

所以 queryLoop() 实际上是在做一个“多级上下文治理管线”，不是单纯 token 截断。

## 6.14 callModel() 只是一轮中的中段，不是整轮本身

query.ts 里真正调用模型的地方是：

```ts
for await (const message of deps.callModel({
  ...
})) {
  ...
}
```

这里的 deps.callModel() 被设计成依赖注入，而不是直接写死调用某个 API 模块。

这样做至少有三个目的：

1. 让 query.ts 更容易测试
2. 让 autocompact、callModel 等关键依赖可替换
3. 让“单轮循环逻辑”和“具体 API 实现”解耦

这说明 query.ts 的作者非常清楚自己在写的是 orchestration 层，而不是 provider adapter。

## 6.15 tool_use 进入执行路径后，消息轨迹必须闭合

query.ts 里一个很容易忽略但非常重要的原则是：

只要 assistant 发出了 tool_use，这条轨迹最终就必须得到对应的 tool_result，即使失败、中断或 fallback 也一样。

这就是 yieldMissingToolResultBlocks() 和 synthetic error 相关逻辑存在的原因。

因为对于后续模型来说，未闭合的轨迹会造成三类问题：

1. 它不知道上一个工具到底执行没执行
2. 恢复时无法判断该轮是否已完成
3. transcript 与真实运行状态不一致

这一点是 Agent 轨迹一致性的核心要求。

## 6.16 StreamingToolExecutor 与 runTools 的分工

query.ts 里同时支持两类工具执行方式：

- StreamingToolExecutor：边收流边起工具
- runTools：拿到整批 tool_use 后统一执行

因此 queryLoop 的工具阶段并不是固定单形态，而是根据当前执行器选择不同路径：

- 如果是流式工具执行器，query loop 会在流仍在进行时就逐步消费工具结果
- 如果是常规模式，则在 assistant 输出完成后对 tool_use blocks 做批处理

这意味着 query.ts 不只是“模型调用循环”，它同时还是“模型输出流”和“工具执行流”的同步点。

## 6.17 413 和 max_output_tokens 恢复为什么都放在 query 层

这两类错误理论上可以在 API 层处理，但 Claude Code 没这么做。

它们被放到 query.ts，是因为能否恢复并不是纯 API 问题，而是取决于整个回合状态：

- 当前消息是不是已经 compact 过
- 有没有 withheld 错误消息
- 当前输出截断是第一次还是第 N 次
- reactiveCompact 有没有已经尝试过

这些都是 loop state，而不是 provider state。

所以恢复逻辑必须放在 queryLoop 这种“看得见整轮上下文”的层上。

## 6.18 停止条件不是“没工具了”这么简单

queryLoop 的结束并不是简单判断“这一轮没有 tool_use 了”。

它还会考虑：

- stop hooks 是否要求阻塞或重试
- 是否 hit 到 token budget continuation/diminishing returns
- 是否因为 abort、fallback、prompt-too-long、max turns 等事件提前结束

所以真正的 return 点，是多种策略共同判定出来的。

这就是为什么 query.ts 很长，因为它维护的是生产环境里的复杂终止语义。
