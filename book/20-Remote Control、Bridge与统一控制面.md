# 第 20 章 Remote Control、Bridge 与统一控制面

> 到这一章，整本书会进入一个非常关键的收束：Claude Code 并不只是在本地终端里跑一个 REPL，它实际上已经把本地 REPL、网页端接入、直连服务端、远端 SDK 会话、SSH 会话，收敛成了一套统一控制面。

> 对应源码主线：src/hooks/useReplBridge.tsx、src/bridge/initReplBridge.ts、src/bridge/bridgePermissionCallbacks.ts、src/hooks/useDirectConnect.ts、src/server/createDirectConnectSession.ts、src/server/directConnectManager.ts、src/hooks/useRemoteSession.ts、src/hooks/useSSHSession.ts、src/ssh/SSHSessionManager.ts，以及 src/screens/REPL.tsx、src/main.tsx 中相关装配

## 20.1 先给结论：什么叫“统一控制面”？

这一章最重要的判断是：

1. transport 可以不同，可能是 WebSocket、HTTP、SSH 子进程、bridge 会话。
2. 但 REPL 关心的上层能力基本相同：收消息、发消息、处理中断、处理权限请求、处理断连、维护会话状态。
3. 因此源码的真正设计目标不是“支持多种远程模式”，而是“把多种远程模式收敛成同一种交互语义”。

为了和全书术语保持一致，本章后面出现的 `session control plane`、`transport 级控制面`、`远端控制面对象`，都可以统一理解为“统一控制面”在不同层次上的展开。

本章主要围绕下面几组源码：

1. src/hooks/useReplBridge.tsx
2. src/bridge/initReplBridge.ts
3. src/bridge/bridgePermissionCallbacks.ts
4. src/hooks/useDirectConnect.ts
5. src/server/createDirectConnectSession.ts
6. src/server/directConnectManager.ts
7. src/hooks/useRemoteSession.ts
8. src/hooks/useSSHSession.ts
9. src/ssh/SSHSessionManager.ts
10. src/screens/REPL.tsx 中 activeRemote 的装配片段
11. src/main.tsx 中 remote、remote-control、sdkUrl、ssh 等模式入口

## 20.2 main.tsx：为什么控制面问题先从入口层看

很多人看到 bridge、remote、ssh 会直奔 transport 实现。但从阅读顺序上，更好的入口其实是 main.tsx。

因为 main.tsx 先解决了一件事：这些模式不是各写一个独立程序，而是在同一个 CLI 入口里被统一解析和装配。

从源码可以看出，入口期至少会处理这些相关选项：

1. sdkUrl
2. remote
3. remoteControl / rc
4. teleport
5. ssh 相关分支
6. direct connect 相关分支

### 2.1 入口层做的不是 transport 连接，而是模式判定与前置约束

例如 main.tsx 会先处理：

1. 鉴权与 onboarding
2. managed settings 与 policy limit
3. trust 建立
4. feature gate
5. 输出模式与输入模式重写
6. remote-control 启用后的设备注册与 token 处理

这意味着远程控制并不是一个后加的网络插件，而是从程序入口就参与运行模式决策。

### 2.2 Remote Control 之所以复杂，是因为它不是单一 transport

入口层实际上已经暗示了几种不同场景：

1. 当前 CLI 作为本地交互终端，但允许外部远程接入。
2. 当前 CLI 直接连接到某个 claude server。
3. 当前 CLI 变成远端会话查看器。
4. 当前 CLI 通过 SSH 代理远端执行。

这些模式对用户来说都像“远程工作”，但工程上完全不是一回事。

## 20.3 useReplBridge：把本地 REPL 变成可远程接入的会话

先看 src/hooks/useReplBridge.tsx。

这个 hook 的职责不是“远程聊天”，而是把本地 REPL 会话镜像到一个 bridge session 上，使外部客户端可以接入当前会话。

源码里的关键判断可以概括成三点。

### 3.1 它是一个常驻后台连接，而不是一次性请求

hook 内部维护了：

1. handleRef
2. teardownPromiseRef
3. lastWrittenIndexRef
4. flushedUUIDsRef
5. consecutiveFailuresRef

这几个状态一眼就说明，bridge 不是“发一条消息出去”这么简单，而是一个需要长期维护、可重连、可去重、可熔断的会话桥。

尤其是 consecutiveFailuresRef 与 MAX_CONSECUTIVE_INIT_FAILURES 的设计，很清楚地表明：

1. bridge 初始化失败不是普通错误提示。
2. 它要防止进入无穷重试。
3. 它要主动把 replBridgeEnabled 熔断掉，避免反复打 401 或反复注册环境。

这属于非常典型的控制面防抖与自我保护逻辑。

这一点可以直接从 hook 开头那组 ref 看出来：

```ts
export const BRIDGE_FAILURE_DISMISS_MS = 10_000
const MAX_CONSECUTIVE_INIT_FAILURES = 3

const handleRef = useRef<ReplBridgeHandle | null>(null)
const teardownPromiseRef = useRef<Promise<void> | undefined>(undefined)
const lastWrittenIndexRef = useRef(0)
const flushedUUIDsRef = useRef(new Set<string>())
const failureTimeoutRef = useRef<ReturnType<typeof setTimeout> | undefined>(undefined)
const consecutiveFailuresRef = useRef(0)
```

这六个状态变量分别对应：连接句柄、拆桥等待、已写入游标、去重集合、错误提示定时器和熔断计数器。也就是说，`useReplBridge` 本质上已经在维护一个长期存在的 session bridge runtime。

### 3.2 它不是简单转发消息，而是在维护“会话镜像”

hook 会把当前 REPL 中的消息作为 initialMessages 提供给 bridge，并通过 flushedUUIDsRef 避免重复写入。

也就是说，bridge 面对的不是“从现在开始的一条 socket 流”，而是一份尽量完整、可续接的会话镜像。

源码注释里强调：如果重复发送相同 UUID，服务端会直接关闭 WebSocket。这恰恰说明 bridge 层要求的是稳定的事件序列，而不是随便 append 文本。

### 3.3 它把远端进入的消息重新注入本地 REPL 队列

handleInboundMessage 这一段非常关键。

它会：

1. 解析 inbound message 字段
2. 处理附件下载与本地文件前缀拼接
3. 必要时做 webhook 内容清洗
4. 最终通过 enqueue 注回 REPL 输入队列

这里最重要的设计点是：远端输入并不会绕开本地 REPL 主线，而是被重新注入现有输入通道。

这意味着：

1. slash command 安全限制仍然有效
2. 本地输入与远端输入共用同一套 processUserInput 路径
3. bridge 只是输入来源变化，不是执行模型变化

## 20.4 initReplBridge：为什么还要再包一层初始化器

src/bridge/initReplBridge.ts 的意义，是把 bridge core 与 REPL 运行时语境接上。

从源码可以看出，它不是裸创建连接，而是额外处理了几件 REPL 场景特有的事：

1. 读取 bootstrap state
2. 处理 OAuth / policy gating
3. 推导会话标题
4. 组装当前环境配置
5. 在 assistant mode 下处理 perpetual session 语义

### 4.1 这说明 bridge core 是通用层，initReplBridge 是 REPL 适配层

这个分层非常值得学习。

如果把所有逻辑都塞进 useReplBridge，最后会变成“React hook 直接知道所有 bridge 内核细节”。

现在的结构是：

1. hook 负责 React 生命周期
2. initReplBridge 负责 REPL 语义初始化
3. 更下层 bridge core 负责真正的连接/会话逻辑

这是一种典型的分层收束。

### 4.2 perpetual session 体现了控制面不是一次会话，而是长期终端身份

assistant mode 下，bridge session 可以跨 CLI 重启持续存在。

这里暴露出一个重要设计思想：有些场景下用户需要的不是“重新开一个新连接”，而是“继续同一个远程身份和同一个会话壳”。

这已经很接近“终端代理常驻体”的概念了。

## 20.5 bridgePermissionCallbacks：为什么权限回调会被单独抽象

src/bridge/bridgePermissionCallbacks.ts 文件不大，但地位很高。

它把 bridge 场景下的权限交互抽象成独立回调接口，至少覆盖：

1. request
2. response
3. cancel

这件事的重要性在于，bridge 权限不是普通本地工具确认框。

bridge 一旦存在，就至少有三方：

1. 本地 REPL
2. 远端控制端
3. 真正执行工具的一侧

如果不把权限回调单独抽象，权限事件就会散落到多个 transport 处理器里，最终很难统一仲裁。

也正因为有这层抽象，第 18 章讲的 permission race 才能扩展到 bridge 场景。

## 20.6 Direct Connect：它不是 bridge 的别名

很多人会把 Direct Connect 和 Remote Control 混在一起，但源码其实切得很清楚。

先看 src/server/createDirectConnectSession.ts。

这里先通过 HTTP 创建一个 direct-connect session，返回配置对象，里面包含：

1. serverUrl
2. sessionId
3. wsUrl
4. authToken

然后 src/server/directConnectManager.ts 再用这份配置建立 WebSocket 级别的会话管理。

### 6.1 create session 与 manage session 被拆开，说明 direct connect 有两阶段语义

这比直接 new WebSocket 更复杂，原因是 direct connect 明显不是纯浏览器 socket，而是带服务端会话资源的。

先创建，再连接，说明系统想要：

1. 先拿到合法会话身份
2. 再进入消息流阶段
3. 将鉴权、会话创建、传输层连接分层处理

### 6.2 DirectConnectSessionManager 负责的是 transport 级控制面

从 directConnectManager.ts 能看出，它不仅收发普通 SDK 消息，还负责：

1. control request
2. permission request
3. interrupt
4. permission response

这非常关键，因为它说明 direct connect 的 WebSocket 不是一根文本流，而是一条控制总线。

## 20.7 useDirectConnect：REPL 视角下，直连模式被压成什么接口

再看 src/hooks/useDirectConnect.ts。

这个 hook 很适合拿来观察“统一控制面”的上层接口长什么样。

它最终返回的是：

1. isRemoteMode
2. sendMessage
3. cancelRequest
4. disconnect

源码里这个接口写得非常直接：

```ts
type UseDirectConnectResult = {
  isRemoteMode: boolean
  sendMessage: (content: RemoteMessageContent) => Promise<boolean>
  cancelRequest: () => void
  disconnect: () => void
}
```

这个接口之所以重要，是因为它几乎就是 REPL 顶层对远程 transport 的最小要求。只要某种 transport 能被压成这四个动作，它就能接进统一控制面。

这几项非常有代表性，因为它说明 REPL 上层真正需要的不是 transport 细节，而是有限几个会话控制动作。

### 7.1 权限请求在这里被重新包装成 ToolUseConfirm

onPermissionRequest 里，远端 permission request 会被包装成 synthetic assistant message 与 ToolUseConfirm 对象，然后塞入本地 setToolUseConfirmQueue。

对应代码骨架如下：

```ts
onPermissionRequest: (request, requestId) => {
  const tool = findToolByName(toolsRef.current, request.tool_name) ?? createToolStub(request.tool_name)

  const syntheticMessage = createSyntheticAssistantMessage(request, requestId)

  const toolUseConfirm: ToolUseConfirm = {
    assistantMessage: syntheticMessage,
    tool,
    description: request.description ?? `${request.tool_name} requires permission`,
    input: request.input,
    toolUseID: request.tool_use_id,
    onAllow(updatedInput) {
      manager.respondToPermissionRequest(requestId, {
        behavior: 'allow',
        updatedInput,
      })
    },
    onReject(feedback) {
      manager.respondToPermissionRequest(requestId, {
        behavior: 'deny',
        message: feedback ?? 'User denied permission',
      })
    },
  }

  setToolUseConfirmQueue((queue) => [...queue, toolUseConfirm])
}
```

这段代码把“远端权限请求如何回到本地审批”写得很清楚：transport 只负责把请求带回来，真正的仲裁入口仍然是本地 REPL 的 `ToolUseConfirm` 队列。

这再次证明：

1. 远端权限请求最终还是走本地统一审批 UI。
2. transport 层不会自己决定 allow/deny。
3. REPL 的权限交互层是上位统一入口。

### 7.2 Direct Connect 与本地 REPL 的接缝就是“消息转换 + 队列桥接”

hook 里一方面用 convertSDKMessage 把远端 SDK message 转成本地消息，另一方面用 createSyntheticAssistantMessage / createToolStub 把远端工具权限变成本地可展示对象。

也就是说，远端模式想接进 REPL，最核心的工作不是联网，而是协议适配。

## 20.8 useRemoteSession：远端 viewer/controller 形态下的统一接口

src/hooks/useRemoteSession.ts 代表的是另一种远程模式。

它和 useDirectConnect 很像，但又多了一些更接近“远端会话镜像器”的职责，例如：

1. 把 SDK message 转成 REPL message
2. 跟踪远端 task count
3. 跟踪 compaction state
4. 处理远端权限请求与用户审批回传

### 8.1 它说明远端会话不仅是消息流，还有状态流

如果只是聊天内容，convertSDKMessage 就够了。

但源码还维护 remote task count 和 compaction state，说明远端 viewer 需要看见的不只是说了什么，还包括：

1. 远端现在是不是忙
2. 有多少任务在跑
3. 上下文是否发生压缩

这其实是在把“远端运行态”投影回本地 UI。

### 8.2 这里再次复用了同一种权限桥接语义

无论 direct connect 还是 remote session，只要出现远端权限请求，最终都要变成本地 ToolUseConfirm 队列，再由用户批准后回写。

这正是统一控制面的核心证据：不同 transport，共用同一审批语义。

## 20.9 useSSHSession：为什么 SSH 也能被纳入同一控制面

SSH 最容易让人误以为是“完全不同的一套实现”。

但读完 src/hooks/useSSHSession.ts 和 src/ssh/SSHSessionManager.ts 以后，会发现源码刻意把它压成与 direct connect、remote session 相似的外观。

hook 层同样负责：

1. 接消息
2. 发消息
3. 中断
4. 权限请求处理
5. 断开与重连
6. graceful shutdown

而 SSHSessionManager.ts 即便当前是 stub/interface，也已经把契约面暴露出来了，例如：

1. connect
2. disconnect
3. sendMessage
4. sendInterrupt
5. respondToPermissionRequest

### 9.1 这说明 SSH 在架构上不是例外，而是 transport 的又一种实现

一旦把 SSH 也压成这组接口，REPL 上层就不需要知道底层到底是：

1. WebSocket
2. HTTP + WebSocket
3. child_process + stdin/stdout

这就是统一控制面的价值。

## 20.10 REPL.tsx：activeRemote 这一行，几乎把全章讲完了

在 src/screens/REPL.tsx 里，有一段极其关键的装配：

1. 先分别创建 remoteSession
2. 再创建 directConnect
3. 再创建 sshRemote
4. 然后通过 isRemoteMode 选择 activeRemote

对应源码几乎就是一眼看穿抽象层的程度：

```ts
const remoteSession = useRemoteSession({ ... })
const directConnect = useDirectConnect({ ... })
const sshRemote = useSSHSession({ ... })

const activeRemote = sshRemote.isRemoteMode
	? sshRemote
	: directConnect.isRemoteMode
		? directConnect
		: remoteSession
```

注意这里没有出现 WebSocket、SSH stdin/stdout、HTTP session create 之类的细节。到了 REPL 顶层，它们已经全部被折叠成“谁当前是活跃远程控制面对象”。这正是统一控制面成立的直接证据。

这几行代码之所以重要，是因为它把前面所有分层设计压缩成了一个结论：

REPL 上层并不关心 transport 具体是谁，它只关心当前有没有一个满足统一接口的 activeRemote。

这说明第 20 章的主题不是“远程功能很多”，而是“远程功能已经被抽象得足够统一，所以 REPL 只需要接一个活跃控制面对象”。

## 20.11 一条完整链路：远端权限请求是怎样回到本地审批的

把这几份文件串起来，远端权限请求的路径大致如下：

1. 远端执行侧产生 permission request。
2. 对应 manager 或 session hook 收到请求。
3. hook 把请求转换为 synthetic assistant message 与 ToolUseConfirm。
4. 该请求进入本地 REPL 的 toolUseConfirmQueue。
5. 用户在本地 UI 点击允许或拒绝。
6. hook 再把 allow/deny 转回 RemotePermissionResponse 或 bridge response。
7. 远端执行侧继续或中止。

这条链路说明，Claude Code 的统一控制面，核心统一的不是网络库，而是审批模型；换句话说，权限仲裁链是这套控制面得以成立的前提之一。

## 20.12 Remote Control 为什么不是一个普通的“把终端画面同步到网页”

这一点非常值得强调。

如果只是屏幕共享，不需要：

1. inbound message 注入
2. permission callback 协议
3. session create + connect 两阶段
4. transport 级 interrupt
5. task count / compaction state 投影
6. 统一 ToolUseConfirm 回流

正因为它做的是“可控会话”，而不是“可看画面”，所以它必须维护一整套控制面状态。

## 20.13 这一章和前文怎么收束

读到这里，前面很多章节会重新连起来：

1. 第 2 章的 main.tsx 大而复杂，是因为它要装配多种运行模式。
2. 第 7 章和第 18 章的权限系统之所以被做得这么抽象，是因为权限需要跨本地/远端/bridge/swarm 复用。
3. 第 10 章 REPL 之所以巨大，是因为它是所有控制面最终汇合的交互壳。
4. 第 14 章的 remote/worktree/assistant viewer 只是空间拓扑切换，而本章讲的是这些拓扑如何共享统一会话接口。

所以本章其实是整本书的一个架构收口。

从交叉引用的角度再看一次，会更容易把后半本书串起来：

1. 第 14 章先告诉你运行拓扑会变化，本章则解释这些不同拓扑为什么还能共用同一种控制面接口。
2. 第 17 章定义了本地任务与前后台状态机，本章则把远端 task 状态和会话控制投影回同一个 REPL 外壳。
3. 第 18 章定义了多来源权限仲裁，本章把这套仲裁模型扩展到了 bridge、direct connect 和 remote session。
4. 第 19 章解释了 swarm 内部的 leader-worker 控制面，本章则把这种控制面思维继续推广到跨 transport 的远程接入。

## 20.14 阅读这部分源码时最容易忽略的点

### 20.14.1 transport 不是主角，接口才是主角

别被 WebSocket、SSH、HTTP 这些词带偏。真正该盯住的是：

1. sendMessage
2. cancelRequest / interrupt
3. disconnect
4. permission request / response
5. message conversion

### 20.14.2 useReplBridge 和 useRemoteSession 解决的是不同问题

前者更像“把本地会话开放出去”，后者更像“把远端会话投影进来”。

它们都叫 remote，但方向正好相反。

### 20.14.3 activeRemote 是统一控制面的最直观证据

如果一个复杂系统最后能在 REPL 顶层被压成一个 activeRemote 变量，说明抽象层做对了。

## 20.15 这一章的阅读结论

写到第 20 章，整套源码其实已经呈现出一个很清晰的形状。

它不是一个“在终端里调用模型 API 的脚本”，而是一套完整的 agent runtime：

1. 入口层有复杂的模式装配与环境治理。
2. 会话层有 QueryEngine 负责状态编排。
3. 循环层有 query.ts 负责模型-工具-结果闭环。
4. 能力层有 tools、commands、skills、MCP、memory、hooks。
5. 控制层有权限仲裁、任务框架、后台 agent、swarm 协作。
6. 拓扑层有 worktree、remote、bridge、direct connect、ssh。

最关键的是，这些部分不是松散拼接，而是通过一组稳定抽象连起来：

1. Task
2. QueryEngine
3. ToolUseConfirm
4. AppState
5. message / SDKMessage 适配层
6. 统一控制面接口

如果用一句话为整本教程收尾：

Claude Code 真正值得精读的地方，不在于它“能调多少工具”，而在于它已经把终端里的单代理交互、后台任务、多代理协作、远端接入和权限控制，收束成了一套统一运行时。

当你看清这套运行时之后，再回头看 main.tsx、REPL.tsx、QueryEngine.ts、query.ts，就不会觉得它们只是“大文件”，而会意识到它们是在承担一个小型 agent 操作系统的总线职责。
