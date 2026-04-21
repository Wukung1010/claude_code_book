# State 模块详细分析

## 概述

Claude Code 使用 Zustand 风格的状态管理模式，核心在 `src/state/` 目录。状态管理分为：
1. Store 实现（底层）
2. AppState 定义（中层）
3. React 集成（上层）

> 来源: `src/state/store.ts:1-34` | `src/state/AppStateStore.ts:1-100`

---

## 1. Store 实现

### 1.1 核心 Store 类型

位置: `src/state/store.ts:1-34`

```typescript
type Listener = () => void
type OnChange<T> = (args: { newState: T; oldState: T }) => void

export type Store<T> = {
  getState: () => T
  setState: (updater: (prev: T) => T) => void
  subscribe: (listener: Listener) => () => void
}
```

### 1.2 createStore 工厂函数

```typescript
export function createStore<T>(
  initialState: T,
  onChange?: OnChange<T>,
): Store<T> {
  let state = initialState
  const listeners = new Set<Listener>()

  return {
    getState: () => state,

    setState: (updater: (prev: T) => T) => {
      const prev = state
      const next = updater(prev)
      if (Object.is(next, prev)) return  // 防止无意义更新
      state = next
      onChange?.({ newState: next, oldState: prev })
      for (const listener of listeners) listener()
    },

    subscribe: (listener: Listener) => {
      listeners.add(listener)
      return () => listeners.delete(listener)  // 返回取消订阅函数
    },
  }
}
```

**特点**:
- 不可变更新（使用 `Object.is` 检测变化）
- 变更回调支持
- 订阅/取消订阅模式

> 来源: `src/state/store.ts:10-34`

---

## 2. AppState 定义

### 2.1 AppState 结构

位置: `src/state/AppStateStore.ts:89-452`

```typescript
export type AppState = DeepImmutable<{
  // 设置与模式
  settings: SettingsJson
  verbose: boolean
  mainLoopModel: ModelSetting
  mainLoopModelForSession: ModelSetting
  
  // 视图状态
  expandedView: 'none' | 'tasks' | 'teammates'
  isBriefOnly: boolean
  viewSelectionMode: 'none' | 'selecting-agent' | 'viewing-agent'
  footerSelection: FooterItem | null
  
  // 权限上下文
  toolPermissionContext: ToolPermissionContext
  
  // 任务状态
  tasks: { [taskId: string]: TaskState }
  agentNameRegistry: Map<string, AgentId>
  foregroundedTaskId?: string
  viewingAgentTaskId?: string
  
  // MCP 状态
  mcp: {
    clients: MCPServerConnection[]
    tools: Tool[]
    commands: Command[]
    resources: Record<string, ServerResource[]>
    pluginReconnectKey: number
  }
  
  // 插件状态
  plugins: {
    enabled: LoadedPlugin[]
    disabled: LoadedPlugin[]
    commands: Command[]
    errors: PluginError[]
    installationStatus: { ... }
    needsRefresh: boolean
  }
  
  // Agent 定义
  agentDefinitions: AgentDefinitionsResult
  
  // 文件历史
  fileHistory: FileHistoryState
  attribution: AttributionState
  
  // Todo 列表
  todos: { [agentId: string]: TodoList }
  
  // 通知
  notifications: {
    current: Notification | null
    queue: Notification[]
  }
  
  // 推测执行
  speculation: SpeculationState
  speculationSessionTimeSavedMs: number
  
  // 团队上下文
  teamContext?: {
    teamName: string
    teamFilePath: string
    leadAgentId: string
    teammates: { [teammateId: string]: { ... } }
  }
  
  // REPL 上下文
  replContext?: {
    vmContext: vm.Context
    registeredTools: Map<string, { name, description, schema, handler }>
    console: { log, error, warn, info, debug, ... }
  }
  
  // 更多字段...
}>
```

### 2.2 默认状态

位置: `src/state/AppStateStore.ts:456-569`

```typescript
export function getDefaultAppState(): AppState {
  return {
    settings: getInitialSettings(),
    tasks: {},
    agentNameRegistry: new Map(),
    verbose: false,
    mainLoopModel: null,
    expandedView: 'none',
    isBriefOnly: false,
    toolPermissionContext: {
      ...getEmptyToolPermissionContext(),
      mode: initialMode,
    },
    mcp: {
      clients: [],
      tools: [],
      commands: [],
      resources: {},
      pluginReconnectKey: 0,
    },
    // ...
  }
}
```

> 来源: `src/state/AppStateStore.ts:456-569`

---

## 3. React 集成

### 3.1 AppStateProvider

位置: `src/state/AppState.tsx:37-116`

```typescript
export function AppStateProvider({ children, initialState, onChangeAppState }) {
  // 创建 store
  const store = createStore(initialState ?? getDefaultAppState(), onChangeAppState)
  
  // 权限模式检查
  useEffect(() => {
    const { toolPermissionContext } = store.getState()
    if (toolPermissionContext.isBypassPermissionsModeAvailable && 
        isBypassPermissionsModeDisabled()) {
      store.setState(createDisabledBypassPermissionsContext(prev))
    }
  }, [store])
  
  return (
    <AppStoreContext.Provider value={store}>
      {children}
    </AppStoreContext.Provider>
  )
}
```

### 3.2 useAppState Hook

位置: `src/state/AppState.tsx:142-163`

```typescript
export function useAppState<R>(selector: (state: AppState) => R): R {
  const store = useAppStore()
  
  return useSyncExternalStore(
    store.subscribe,
    () => selector(store.getState()),
    () => selector(store.getState()),
  )
}
```

**使用示例**:
```typescript
const verbose = useAppState(s => s.verbose)
const model = useAppState(s => s.mainLoopModel)
const tasks = useAppState(s => s.tasks)
```

### 3.3 useSetAppState Hook

位置: `src/state/AppState.tsx:170-172`

```typescript
export function useSetAppState() {
  return useAppStore().setState
}
```

---

## 4. 状态变更处理

### 4.1 onChangeAppState

位置: `src/state/onChangeAppState.ts`

处理状态变更的副作用：
- 权限设置变更
- 模型变更
- 设置变更
- 插件状态变更

### 4.2 选择器

位置: `src/state/selectors.ts`

常用选择器函数：
```typescript
export const selectVerbose = (s: AppState) => s.verbose
export const selectModel = (s: AppState) => s.mainLoopModel
export const selectToolPermissionContext = (s: AppState) => s.toolPermissionContext
```

---

## 5. 特殊状态类型

### 5.1 SpeculationState

位置: `src/state/AppStateStore.ts:52-77`

```typescript
export type SpeculationState =
  | { status: 'idle' }
  | {
      status: 'active'
      id: string
      abort: () => void
      startTime: number
      messagesRef: { current: Message[] }
      writtenPathsRef: { current: Set<string> }
      boundary: CompletionBoundary | null
      suggestionLength: number
      toolUseCount: number
      isPipelined: boolean
    }
```

### 5.2 CompletionBoundary

位置: `src/state/AppStateStore.ts:41-50`

```typescript
export type CompletionBoundary =
  | { type: 'complete'; completedAt: number; outputTokens: number }
  | { type: 'bash'; command: string; completedAt: number }
  | { type: 'edit'; toolName: string; filePath: string; completedAt: number }
  | { type: 'denied_tool'; toolName: string; detail: string; completedAt: number }
```

---

## 6. 子模块状态

### 6.1 BashTool State

位置: `src/tools/BashTool/src/state/AppState.ts`

BashTool 可能有自己独立的状态子集。

### 6.2 MCP State

位置: `src/services/mcp/src/state/AppState.ts`

MCP 连接和工具状态。

---

## 7. 文件路径索引

| 文件 | 功能 |
|------|------|
| `src/state/store.ts` | Store 实现 |
| `src/state/AppState.tsx` | React Provider 和 Hooks |
| `src/state/AppStateStore.ts` | AppState 类型定义 |
| `src/state/onChangeAppState.ts` | 状态变更处理 |
| `src/state/selectors.ts` | 选择器函数 |
| `src/state/teammateViewHelpers.ts` | 团队视图辅助 |
