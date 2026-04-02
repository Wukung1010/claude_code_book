# 第 19 章 Swarm 协作、InProcessTeammate 与邮箱协议

> 这一章聚焦 Claude Code 里一个很容易被低估的系统：多智能体协作并不是简单地“再起几个 agent”，而是被做成了带身份、带邮箱、带审批桥接、带前后台视图切换能力的一整套协同运行时。

> 对应源码主线：src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx、src/tasks/InProcessTeammateTask/types.ts、src/hooks/useInboxPoller.ts、src/utils/teammateMailbox.ts、src/utils/swarm/permissionSync.ts

## 19.1 先给结论：这一章到底在讲什么？

如果说第 15 章讲的是“子代理怎么被创建和运行”，第 17 章讲的是“任务怎样进入统一 task 框架”，那么这一章讲的就是：

1. 为什么有些 agent 不需要新进程，也不需要远程连接，而是直接在当前 Node.js 进程里并行存在。
2. 这些协作节点如何拥有自己的身份、消息流、前台视图和生命周期。
3. leader 和 worker 之间如何不用共享内存对象，而通过邮箱协议与权限同步文件完成协作。
4. 为什么权限审批、计划确认、sandbox 放行、关闭请求都被抽象成统一的消息协议。

可以把这一章的对象理解成一句话：Claude Code 在 REPL 内部又搭了一层“小型协作操作系统”。

为了和全书术语保持一致，本章里的 `teammate` 在解释性文字里都可以理解为“协作节点”，但在涉及源码名、类型名和 UI 名称时仍保留英文。

## 19.2 Swarm 体系里的几个核心角色

本章要读的几份源码分别承担不同角色：

1. src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx
2. src/tasks/InProcessTeammateTask/types.ts
3. src/hooks/useInboxPoller.ts
4. src/utils/teammateMailbox.ts
5. src/utils/swarm/permissionSync.ts

从职责上看，它们大致分成三层。

### 2.1 任务实体层：InProcessTeammateTask

这一层回答的问题是：一个协作节点在前端状态树里到底是什么。

它不是一段松散的消息，也不是 QueryEngine 的一个布尔标记，而是明确实现了 Task 接口的任务对象。

这意味着：

1. 它天然可以被任务框架托管。
2. 它可以进入前台查看态，也可以留在后台执行态。
3. 它可以有独立标题、独立消息流、独立 loading/idle 状态。
4. 它可以和 LocalAgentTask、LocalMainSessionTask 一起被统一导航与渲染。

### 2.2 协议层：teammateMailbox

这一层回答的问题是：leader 与协作节点之间到底交换什么。

关键不是“发字符串”，而是发结构化消息。邮箱消息里明确区分了：

1. 普通消息
2. permission_request / permission_response
3. sandbox_permission_request / sandbox_permission_response
4. plan approval request / response
5. shutdown request / approval

这意味着 swarm 协作并不依赖调用方和被调用方同时持有某个 React 回调，而是可以退化为一条稳定协议。

### 2.3 同步层：useInboxPoller 与 permissionSync

这一层回答的问题是：消息到了以后谁来处理，以及审批结果怎样可靠地传回去。

这里用了两种机制：

1. useInboxPoller 负责周期性读取邮箱，把不同类型消息路由到 REPL/AppState 的正确队列。
2. permissionSync 负责把某些 leader-worker 审批过程落到文件系统目录里，形成 worker 请求、leader 决议、worker 取回结果的闭环。

这样做的关键价值是：协作关系不会被某个瞬时内存引用绑死。

## 19.3 InProcessTeammateTask：为什么它不是普通子代理

先看类型定义最能抓住本质。

在 src/tasks/InProcessTeammateTask/types.ts 里，协作节点的状态不是一两个字段，而是一份很完整的运行时快照，核心信息包括：

1. identity：agentId、agentName、teamName、颜色等身份信息。
2. permissionMode：它不是借用主会话一个全局权限，而是显式记录自己的权限模式。
3. currentWorkAbortController：说明它有独立的当前工作单元，可被中断。
4. pendingUserMessages：说明它支持“排队注入”的用户消息，而不要求马上执行。
5. messages：说明它有自己的消息历史。
6. isIdle：说明协作节点不是单纯 running/stopped 二态，而是有空闲概念。
7. shutdownRequested：说明关闭流程不是粗暴销毁，而是进入一条显式状态机。

从这里已经能看出来，InProcessTeammateTask 的定位并不是“runAgent 的临时包装”，而是“一个常驻的协作工作节点”。

可以直接看一段状态定义：

```ts
export type InProcessTeammateTaskState = TaskStateBase & {
  type: 'in_process_teammate'
  identity: TeammateIdentity
  prompt: string
  model?: string
  selectedAgent?: AgentDefinition
  abortController?: AbortController
  currentWorkAbortController?: AbortController
  awaitingPlanApproval: boolean
  permissionMode: PermissionMode
  messages?: Message[]
  pendingUserMessages: string[]
  isIdle: boolean
  shutdownRequested: boolean
  lastReportedToolCount: number
  lastReportedTokenCount: number
}

export const TEAMMATE_MESSAGES_UI_CAP = 50
```

这段代码很能说明问题：

1. `identity` 说明协作节点不是匿名 worker，而是带明确团队身份的执行体。
2. `currentWorkAbortController` 说明它可以只中断“当前轮工作”，而不必销毁整个协作节点。
3. `pendingUserMessages` 说明它天然带有排队消费语义。
4. `isIdle` 和 `shutdownRequested` 说明它的生命周期被显式建模。
5. `TEAMMATE_MESSAGES_UI_CAP` 说明 UI 镜像是有上限的，任务运行态和展示态被主动区分。

源码里还有一个很关键但容易忽略的点：UI 消息数量是有限制的。

这说明设计者有意区分了两类历史：

1. 运行时需要完整轨迹的内部状态。
2. 给前台面板展示时需要裁剪的 UI 历史。

也就是说，协作节点既要像任务一样可观察，又不能让它的可视层无限膨胀。

## 19.4 Task 实现告诉我们：同进程协作是如何被纳入统一任务框架的

src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx 里最重要的不是 JSX，而是那组任务语义方法。

从源码行为上，可以把这个 Task 概括成下面几类能力。

### 4.1 生命周期能力

它支持：

1. 启动后驻留
2. 标记空闲与活跃
3. 请求关闭
4. 真正完成关闭

这意味着协作节点不是“一次性调用完成即销毁”的 job，更像一个长期存在的 worker。

### 4.2 消息能力

它支持：

1. 追加自身消息
2. 注入用户消息
3. 查询或排序协作节点列表
4. 维护 zoomed transcript / 当前展示轨迹

这几项能力非常关键，因为它说明 swarm 不是后台黑盒。用户在 REPL 里是可以“看见某个协作节点正在说什么、做什么、处于什么状态”的。

### 4.3 中断能力

currentWorkAbortController 的存在表明，协作节点正在进行的单次工作可以独立取消，而不必把整个团队上下文全部干掉。

这和第 17 章任务框架里的 retain / foreground / evict 思想是连起来的：

1. 任务对象长期存在。
2. 任务内部当前工作单元可以短生命周期。
3. 前台视图切换与真实执行生命周期被有意拆开。

## 19.5 为什么要做 mailbox，而不是直接调函数

这个问题很重要。

如果 swarm 只是若干个同进程对象，理论上完全可以：

1. leader 直接调用某个协作节点的方法
2. 协作节点再把结果回调给 leader

但源码没有这么做，而是引入了 src/utils/teammateMailbox.ts。

原因有三个。

### 5.1 协作关系需要协议化，而不是对象引用化

一旦改成 mailbox message，leader 与 worker 之间的交互就不再依赖“我正好拿着你的实例引用”。

这会直接带来几个收益：

1. 容易做延迟处理
2. 容易做重试和去重
3. 容易把相同协议迁移到别的 transport
4. 容易做 leader 统一仲裁

换句话说，mailbox 是在为“协作协议独立于执行载体”打地基。

### 5.2 权限与计划审批天然是异步事务

普通消息可以同步处理，但权限审批不是。

一个协作节点发起工具权限请求以后：

1. 需要 leader 看到它
2. 需要进入主界面的审批 UI
3. 需要等待用户决策
4. 再把 allow/deny 回传给协作节点

这是一条天然的异步链。用 mailbox message 建模，比直接函数回调更稳定。

### 5.3 关闭流程也需要双向握手

shutdown request / approval 这组消息说明，关闭协作节点不是单向 kill。

这是一个非常工程化的选择，因为协作节点可能正处于：

1. 有未完成任务
2. 有待确认权限
3. 有待同步计划
4. 有待刷新的消息状态

所以它需要一条可确认、可拒绝、可延迟的关闭协议。

## 19.6 useInboxPoller：leader 端真正的“收发室”

如果 teammateMailbox 定义的是“信封格式”，那么 src/hooks/useInboxPoller.ts 就是“收发室工作人员”。

这个 hook 的关键不在于轮询本身，而在于它做了清晰的分类路由。

从源码能看出，它至少会把未读消息拆成几类：

1. 权限请求与响应
2. sandbox 权限请求与响应
3. plan approval 请求与响应
4. shutdown 消息
5. 常规消息

这里可以直接看分类代码的骨架：

```ts
const permissionRequests: TeammateMessage[] = []
const permissionResponses: TeammateMessage[] = []
const sandboxPermissionRequests: TeammateMessage[] = []
const sandboxPermissionResponses: TeammateMessage[] = []
const shutdownRequests: TeammateMessage[] = []
const shutdownApprovals: TeammateMessage[] = []
const teamPermissionUpdates: TeammateMessage[] = []
const modeSetRequests: TeammateMessage[] = []
const planApprovalRequests: TeammateMessage[] = []
const regularMessages: TeammateMessage[] = []

for (const m of unread) {
  const permReq = isPermissionRequest(m.text)
  const permResp = isPermissionResponse(m.text)
  const sandboxReq = isSandboxPermissionRequest(m.text)
  const sandboxResp = isSandboxPermissionResponse(m.text)
  const shutdownReq = isShutdownRequest(m.text)
  const shutdownApproval = isShutdownApproved(m.text)
  const teamPermUpdate = isTeamPermissionUpdate(m.text)
  const modeSetReq = isModeSetRequest(m.text)
  const planApprovalReq = isPlanApprovalRequest(m.text)

  if (permReq) permissionRequests.push(m)
  else if (permResp) permissionResponses.push(m)
  else if (sandboxReq) sandboxPermissionRequests.push(m)
  else if (sandboxResp) sandboxPermissionResponses.push(m)
  else if (shutdownReq) shutdownRequests.push(m)
  else if (shutdownApproval) shutdownApprovals.push(m)
  else if (teamPermUpdate) teamPermissionUpdates.push(m)
  else if (modeSetReq) modeSetRequests.push(m)
  else if (planApprovalReq) planApprovalRequests.push(m)
  else regularMessages.push(m)
}
```

这段代码的价值不只是“把消息分组”，而是把 mailbox 正式提升成控制面协议：同一个 inbox 文件里，既可能装用户可读消息，也可能装权限控制、计划审批、模式切换和关闭握手。

### 6.1 最关键的动作：把 worker 的 permission request 提升到 leader UI 队列

这一点是 swarm 设计的中心。

worker 侧提出的不是一个立即执行的动作，而是一个待仲裁事件。useInboxPoller 会把这类事件提升为主会话里的 ToolUseConfirm 队列项。

这一步意味着：

1. worker 并不直接拥有最终审批权。
2. leader UI 才是用户可见审批入口。
3. swarm 下的权限系统和主 REPL 的权限系统不是两套，而是被桥接为一套。

这也是为什么第 18 章会说权限批准最终会汇入同一条权限仲裁链。

### 6.2 普通消息与控制消息被故意分流

如果所有消息都混到一条流里，前台只会得到一堆“对话气泡”。

但源码里明确把 control-plane message 和 content message 分开处理，这说明设计重点不是展示，而是状态驱动：

1. 有的消息要变成 UI 通知
2. 有的消息要变成任务状态迁移
3. 有的消息要变成权限队列
4. 有的消息只是追加到 transcript

这才是协作系统真正复杂的地方。

## 19.7 permissionSync：为什么审批还要落文件系统

只看 useInboxPoller 还不够，因为还有一条很关键的补强链：src/utils/swarm/permissionSync.ts。

这个文件最值得注意的是它把审批过程做成了文件系统协议。

源码中的思路可以概括为：

1. worker 把请求写入 pending 目录
2. leader 读取待处理请求
3. leader 写回 resolved 结果
4. worker 再读取自己的审批结果

permissionSync 里的 schema 也把这种“轻量事务”语义写得很明确：

```ts
export const SwarmPermissionRequestSchema = lazySchema(() =>
  z.object({
    id: z.string(),
    workerId: z.string(),
    workerName: z.string(),
    teamName: z.string(),
    toolName: z.string(),
    toolUseId: z.string(),
    description: z.string(),
    input: z.record(z.string(), z.unknown()),
    permissionSuggestions: z.array(z.unknown()),
    status: z.enum(['pending', 'approved', 'rejected']),
    resolvedBy: z.enum(['worker', 'leader']).optional(),
    resolvedAt: z.number().optional(),
    feedback: z.string().optional(),
    updatedInput: z.record(z.string(), z.unknown()).optional(),
    permissionUpdates: z.array(z.unknown()).optional(),
    createdAt: z.number(),
  }),
)
```

这里最值得注意的是三组字段：

1. `status`、`resolvedBy`、`resolvedAt`：明确表明它不是即时回调，而是可落盘、可追踪的状态流转。
2. `updatedInput`、`permissionUpdates`：说明 leader 不只是二元批准/拒绝，还可能把修正后的输入和规则变更一并回传。
3. `workerId`、`workerName`、`teamName`：说明权限请求是团队路由对象，不是单纯的本地弹窗事件。

### 7.1 这不是多余，而是在补可靠性

为什么不全靠内存？

因为 swarm 协作里最敏感的就是“请求发出去了，但决议没回来”。

一旦引入文件系统同步，就获得了：

1. 明确的请求标识
2. 更稳定的中间状态持久化
3. 更容易调试的落盘痕迹
4. 进程内复杂状态之外的一层保底通道

这种设计特别像把审批事件做成轻量级事务日志。

### 7.2 它说明 swarm 并不是一个只为 UI 写的特性

如果只是为了前台展示，根本不需要 pending/resolved 目录。

真正原因是：多智能体协作一旦进入权限仲裁，就必须考虑可靠传递、失败恢复、重复消费和状态可观测性。

这已经是“分布式系统思维”了，只不过被缩小到了单机协作范围内。

## 19.8 一条完整链路：协作节点发起权限请求之后发生了什么

把这几份源码合起来，一条典型链路大致是这样：

1. InProcessTeammateTask 内部某次工作需要调用受控工具。
2. worker 侧构造 permission request。
3. 请求通过 mailbox 协议或 permissionSync 进入 leader 可见通道。
4. useInboxPoller 发现未读权限请求。
5. leader 把它转成主界面的 ToolUseConfirm 项。
6. 用户在主界面批准或拒绝。
7. leader 把决议写回 mailbox response 或 resolved 文件。
8. worker 拿到结果后继续执行，或者进入 deny 分支。

这条链路有两个非常重要的设计信号。

### 8.1 主审批权永远在 leader

即便协作节点与 leader 同进程存在，它也没有绕过主界面直接批准自己的权限请求。

这说明 swarm 是协作扩展，不是权限边界突破口。

### 8.2 worker 的执行连续性由协议保障，而不是由同步调用保障

也就是说，worker 可以发出请求后等待，而不要求 leader 的那一帧调用栈还活着。

这会大幅提升系统在复杂交互下的稳定性。

## 19.9 待处理用户消息：为什么协作节点需要 pendingUserMessages

types.ts 里有个非常有意思的字段：pendingUserMessages。

它说明协作节点并不一定“现在收到、现在处理”。

这背后其实是两个问题。

### 9.1 协作节点需要显式的工作队列语义

如果当前协作节点正在忙，新的用户消息不能直接打断到任意内部状态上。

所以系统给它一个待处理队列，使其可以：

1. 先完成当前工作
2. 再消费新输入
3. 或在安全点切换上下文

### 9.2 这让协作节点更像一个 actor

从建模方式上说，InProcessTeammateTask 很像 actor：

1. 有身份
2. 有 mailbox
3. 有自己的 message queue
4. 有自己的状态
5. 通过异步消息交互而不是共享执行栈交互

如果用这个视角回头看 swarm，很多实现都会立刻更清楚。

## 19.10 为什么 UI 视图和团队执行要解耦

第 17 章已经讲过任务框架把“后台执行”和“前台查看”拆开。本章在协作节点上把这个思想推得更彻底。

协作节点之所以能被 zoom、被切换、被查看，是因为它的状态和消息流本来就是独立任务对象。

这直接带来三个效果：

1. 用户可以查看某个协作节点的工作过程，而不影响其他协作节点。
2. leader 可以继续主线对话，同时后台协作节点继续跑。
3. 协作系统的可观察性来自任务抽象，而不是来自临时日志打印。

这也解释了为什么 InProcessTeammateTask 必须实现 Task，而不是仅仅实现一个 agent runner 接口。

## 19.11 这一章和后续章节怎么衔接

把本章放回全书主线，可以得到一个很清楚的定位：

1. 第 15 章回答“agent 怎样被定义与启动”。
2. 第 17 章回答“任务怎样被统一托管”。
3. 第 18 章回答“复杂权限回调怎样仲裁”。
4. 第 19 章回答“多个协作节点怎样通过 mailbox 与审批桥接拼成一支团队”。
5. 第 20 章则会继续沿着这里的控制面思路，解释这种统一仲裁和统一状态接口如何跨出本地 swarm，扩展到 bridge、direct connect 和 ssh。

也就是说，本章其实是前面三章的交汇点。

## 19.12 阅读这部分源码时最容易忽略的点

### 12.1 Swarm 不是多开 QueryEngine 那么简单

它真正新增的是：

1. 身份层
2. 协议层
3. 邮箱层
4. 审批桥接层
5. 前后台视图层

### 12.2 mailbox 的意义大于消息传输本身

它真正解决的是控制面解耦，而不是内容传输。

### 12.3 permissionSync 暗示了系统把协作问题当成可靠性问题来处理

这不是普通聊天应用会做的事，而是 agent runtime 在面对多执行体协同时必须做的事。

## 19.13 这一章的阅读结论

这一章读完，应该建立起这样一个判断：Claude Code 的 swarm 协作不是“agent 数量增加”，而是“系统运行时模型升级”。

在这个模型里：

1. 协作节点是 Task，不是临时调用。
2. 协作节点有身份、有队列、有消息历史、有前台视图。
3. leader 与 worker 通过 mailbox 和文件同步协议协作。
4. 权限、计划、关闭这些高风险动作都通过统一控制消息完成。

到了这里，Claude Code 已经不只是单体 CLI Agent，而是开始显露出多执行体协作平台的形状。

下一章继续沿着这个方向，把本地 REPL、Remote Control、Bridge、Direct Connect、SSH 这几条控制面路线放到一张图里，理解它如何把不同 transport 统一成同一种“会话控制接口”。
