# 第 4 章 上下文构建与 System Prompt

> 对应源码主线：src/context.ts，配合 docs/context/system-prompt.mdx、docs/context/project-memory.mdx

## 4.1 Claude Code 不是把“当前消息”直接丢给模型

很多初学者理解 Agent 系统时，会把“用户输入”误当成模型调用的主要输入。

但在 Claude Code 里，用户输入只是最后一层。

真正送到模型面前的是：

1. system prompt
2. system context
3. user context
4. 历史消息
5. 工具描述
6. 运行态附加信息

其中 context.ts 负责的是 system context 和 user context 这两块。

从全书术语上说，这一章讲的是统一运行时如何为每一轮会话准备稳定上下文素材。

## 4.2 getSystemContext：系统侧环境快照

getSystemContext() 的职责很明确：生成一份在当前会话内稳定、适合挂到系统提示词尾部的环境快照。

关键代码如下：

```ts
export const getSystemContext = memoize(async (): Promise<{ [k: string]: string }> => {
  const gitStatus =
    isEnvTruthy(process.env.CLAUDE_CODE_REMOTE) || !shouldIncludeGitInstructions() ? null : await getGitStatus()

  const injection = feature('BREAK_CACHE_COMMAND') ? getSystemPromptInjection() : null

  return {
    ...(gitStatus && { gitStatus }),
    ...(feature('BREAK_CACHE_COMMAND') && injection ? { cacheBreaker: `[CACHE_BREAKER: ${injection}]` } : {}),
  }
})
```

这里最关键的点有两个：

1. 它是 memoize 的，也就是会话内缓存。
2. 它的主要内容是 gitStatus，而不是任意环境变量堆砌。

这意味着 Claude Code 认为“当前仓库状态”是系统级推理的重要上下文。

## 4.3 getGitStatus 其实是上下文压缩后的 Git 摘要

getGitStatus() 并没有把完整 git 信息原样塞给模型，而是并发执行几个核心命令：

- 当前分支
- main branch
- git status --short
- 最近 5 条 commit
- git user.name

并且对 status 做了字符上限裁剪：

```ts
const truncatedStatus =
  status.length > MAX_STATUS_CHARS
    ? status.substring(0, MAX_STATUS_CHARS) +
      '\n... (truncated because it exceeds 2k characters. If you need more information, run "git status" using BashTool)'
    : status
```

这个设计很值得学习：

- 能给模型的上下文一定是“高密度摘要”
- 过长信息不应该一股脑塞进去
- 如果模型确实需要更多细节，再引导它调用工具获取

这就是“系统上下文负责给方向，工具调用负责拿细节”。

## 4.4 getUserContext：用户侧长期知识注入

getUserContext() 负责的是：

1. CLAUDE.md 相关内容
2. 当前日期

对应代码：

```ts
export const getUserContext = memoize(async (): Promise<{ [k: string]: string }> => {
  const shouldDisableClaudeMd =
    isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_CLAUDE_MDS) ||
    (isBareMode() && getAdditionalDirectoriesForClaudeMd().length === 0)

  const claudeMd = shouldDisableClaudeMd ? null : getClaudeMds(filterInjectedMemoryFiles(await getMemoryFiles()))

  setCachedClaudeMdContent(claudeMd || null)

  return {
    ...(claudeMd && { claudeMd }),
    currentDate: `Today's date is ${getLocalISODate()}.`,
  }
})
```

这里要注意一个细节：CLAUDE.md 走的是 user context，而不是 system context。

这很合理，因为它更像“项目/用户知识补充”，而不是系统级运行环境快照。

## 4.5 CLAUDE.md 在这个工程里的角色

CLAUDE.md 的职责不是文档展示，而是让模型获得项目约束与协作约定。

它通常会承载：

- 项目背景
- 开发规范
- 常用命令
- 目录约定
- 对 Agent 的协作要求

在 Claude Code 里，CLAUDE.md 被自动收集后合并进上下文，这就使得“代码库本地约束”成为模型推理的一部分。

换句话说，项目不是通过 prompt engineering 在会话里临时描述自己，而是通过仓库内文件长期自描述。

## 4.6 为什么上下文也要做缓存

无论 getSystemContext 还是 getUserContext，都采用了 memoize。

这是因为：

1. git status 不应该每轮都重新跑
2. CLAUDE.md 扫描和合并不应该每轮都重新做
3. 会话里这些信息默认是稳定的

这样做的效果是：

- 减少 I/O
- 减少子进程开销
- 稳定 prompt cache 前缀

这说明这个项目并不是“功能先行”，而是一直在围绕 token 成本和启动/轮询成本做优化。

## 4.7 system prompt 不是一段字符串，而是一套可缓存的结构

参考 docs/context/system-prompt.mdx 可以看到，这个工程的 system prompt 设计重点并不是“写了什么提示词”，而是“如何结构化拼装，以便尽量命中缓存”。

核心思想是：

1. 把稳定部分和动态部分拆开
2. 让静态部分尽可能稳定
3. 让动态部分只在必要时发生变化
4. 尽量避免因为微小运行时差异而污染 prompt cache

这也是为什么源码里会非常在意：

- 哪些 section 能 cache
- 哪些 section 只能 org 级 cache
- 哪些内容必须放到动态边界之后

## 4.8 上下文系统与 QueryEngine 的接口关系

在 QueryEngine.ts 中，会通过 fetchSystemPromptParts 等逻辑把：

- defaultSystemPrompt
- userContext
- systemContext

组合成实际请求参数。

这意味着 context.ts 不直接负责“发送给模型”，它负责的是“提供会话级稳定上下文素材”。

这是一种非常好的职责分离：

- context.ts 负责收集和缓存
- QueryEngine 负责编排
- query.ts 负责真正进入调用循环

## 4.9 project memory：为什么它是上下文系统的一部分

在 docs/context/project-memory.mdx 中，项目记忆系统的核心思想非常清楚：

- 记忆不等于数据库
- 记忆是文件系统中的持久化知识组织
- 记忆的目标是跨对话保留“代码里推不出来但长期有用的信息”

这和 CLAUDE.md 的差别在于：

- CLAUDE.md 偏“仓库显式约定”
- project memory 偏“会话与协作沉淀出的长期知识”

二者一起构成了 Claude Code 的长期上下文来源。

## 4.10 这一章的阅读结论

这一章最核心的结论有四个：

1. Claude Code 的上下文不是临时拼几段字符串，而是分层构造的。
2. system context 负责系统/仓库运行快照，user context 负责项目/用户长期知识。
3. CLAUDE.md、memory、git status 共同构成模型“理解项目”的基础。
4. 上下文系统从设计一开始就考虑了缓存稳定性和 I/O 成本，而不是事后补优化。

下一章看 QueryEngine.ts，理解这些上下文是如何被组织进“一个真正可持续多轮运行的会话”里的。

## 4.11 fetchSystemPromptParts() 才是上下文真正进入会话编排的接缝

context.ts 本身只负责提供素材，但真正把它们拉进会话装配的是 utils/queryContext.ts 里的 fetchSystemPromptParts()。

它的逻辑很清楚：

```ts
const [defaultSystemPrompt, userContext, systemContext] = await Promise.all([
  customSystemPrompt !== undefined
    ? Promise.resolve([])
    : getSystemPrompt(...),
  getUserContext(),
  customSystemPrompt !== undefined ? Promise.resolve({}) : getSystemContext(),
])
```

这里有两个非常关键的设计点：

1. system prompt、user context、system context 三块是并发获取的
2. 如果设置了 customSystemPrompt，就直接跳过默认 system prompt 和 system context

这说明“自定义 system prompt 完全替换默认系统视角”是一个明确的产品语义，而不是简单追加文本。

把 `fetchSystemPromptParts()` 的核心骨架直接贴出来，会更清楚：

```ts
const [defaultSystemPrompt, userContext, systemContext] = await Promise.all([
  customSystemPrompt !== undefined
    ? Promise.resolve([])
    : getSystemPrompt(tools, mainLoopModel, additionalWorkingDirectories, mcpClients),
  getUserContext(),
  customSystemPrompt !== undefined ? Promise.resolve({}) : getSystemContext(),
])
```

这段代码有两个非常重要的信号。第一，system prompt、user context、system context 三块是并发获取的，说明它们在编排层里被视作并列的上下文素材。第二，customSystemPrompt 会直接让默认 `getSystemPrompt()` 和 `getSystemContext()` 双双退出，这不是“替换一段文字”，而是替换默认系统视角整套装配逻辑。

## 4.12 为什么自定义 system prompt 会跳过 systemContext

这一步很多人第一次看会觉得奇怪：

- 有 customSystemPrompt 时，为什么 systemContext 也不拿了？

原因在于这个工程把 systemContext 定义成“默认系统提示体系的一部分”。

既然 customSystemPrompt 表示调用方要替换默认系统视角，那继续往后面拼默认 systemContext 反而会污染调用方意图。

因此 fetchSystemPromptParts() 的策略是：

- custom 就完全 custom
- default 才搭配 default 的 systemContext

这是非常明确的提示词边界设计。

## 4.13 getSystemContext 与 trust 边界的关系

虽然 context.ts 里 getSystemContext() 看起来只是 memoized async 函数，但它实际受 trust 边界控制。

interactiveHelpers.tsx 里在 trust 成立之后才会：

```ts
setSessionTrustAccepted(true)
void getSystemContext()
```

这意味着 git status 这种系统级上下文，不是任意时刻都能预取，而是要等工作区信任已经建立。

这里背后的思路很重要：

- Git 命令本身也可能触发外部行为
- 所以 system context 收集并不被视作完全无害的纯读取

这说明上下文系统也服从安全模型，而不是独立于安全模型之外。

这条 trust 接缝在 `showSetupScreens()` 里写得非常直白：

```ts
setSessionTrustAccepted(true)
resetGrowthBook()
void initializeGrowthBook()
void getSystemContext()

if (allErrors.length === 0) {
  await handleMcpjsonServerApprovals(root)
}
```

顺序本身就是设计：先确认 trust 成立，再重置并初始化依赖 trust 的服务，然后才预取 system context 和处理 mcp 审批。这意味着 `getSystemContext()` 虽然表面上只是 git/status 汇总，但在统一运行时里它仍然属于“受信任工作区之后才能主动拉起”的系统信息。

## 4.14 getUserContext 与 bare mode 的语义

getUserContext() 里对 bare mode 的处理很有代表性：

```ts
const shouldDisableClaudeMd =
  isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_CLAUDE_MDS) ||
  (isBareMode() && getAdditionalDirectoriesForClaudeMd().length === 0)
```

这段逻辑表达的语义不是“bare mode 一律不要 CLAUDE.md”，而是：

- bare mode 下不做自动发现
- 但如果用户显式通过 --add-dir 指定目录，仍然尊重用户意图

这就是源码注释里说的那句话：

bare 的意思是“跳过我没要求的东西”，不是“忽略我明确要求的东西”。

## 4.15 上下文系统的缓存价值，不只是性能，还有 prompt 稳定性

前面已经提到 memoize 可以减少 I/O。

再往深一点看，它还有第二个价值：

- 减少同一会话内 system/user context 的非必要变化
- 进而减少 prompt cache 前缀抖动

对于 Claude Code 这种高度依赖长上下文、多轮调用和工具描述的系统来说，prompt 前缀稳定性直接影响 token 成本。

因此 context.ts 的缓存设计，本质上同时服务：

1. 本地性能
2. 远端成本

这是一个非常典型的“本地设计影响云端账单”的工程点。

另一个很容易被忽略但很关键的细节，是 `setSystemPromptInjection()` 会在 cache breaker 变化时立刻清空上下文缓存：

```ts
export function setSystemPromptInjection(value: string | null): void {
  systemPromptInjection = value
  getUserContext.cache.clear?.()
  getSystemContext.cache.clear?.()
}
```

这说明 context.ts 的缓存并不是“算过一次就永远不动”，而是围绕 prompt 前缀稳定性精细控制的。一旦 cache breaker 改变，user/system context 都必须立刻失效重算，否则缓存命中就会建立在错误前提上。

## 4.16 system prompt、system context、user context 的职责边界

到这里可以把三者边界总结得更明确：

### system prompt

定义 Claude Code 作为一个 Agent 的系统行为、规则、能力使用说明。

### system context

提供当前环境的系统级快照，例如 git 状态。

### user context

提供项目级、用户级长期知识，例如 CLAUDE.md、日期、memory 相关内容。

这样分层之后，后面的 QueryEngine 才能把它们按不同语义装配进同一个回合。

## 4.17 这一章和后续章节怎么衔接

第 4 章是前半本里承上启下的一章：

1. 它承接第 1 章的统一运行时全景，把“系统为什么不是只看当前输入”这个问题具体化为 `system context + user context + system prompt` 三层素材。
2. 它直接通向第 5 章，因为 `QueryEngine` 正是这些上下文素材真正进入会话编排的接缝。
3. 它也会回流到第 12 章，因为这里反复强调的缓存稳定性、prompt 边界和上下文分层，本质上就是后面 prompt cache 与上下文压缩能成立的前提。

所以第 4 章最好的读法，不是把它当成“提示词章节”，而是把它看成统一运行时里的上下文装配章节。
