# REPL.tsx 详细分析

## 概述

`src/screens/REPL.tsx` (58705 tokens, 5000+ 行) 是 Claude Code 的核心交互界面组件，负责处理用户输入、消息显示、工具权限审批和键盘快捷键。

## 主要 Props

> 来源: `/Users/sunxianfeng/Desktop/n-true/claude_code_book/reference/claude-code/src/screens/REPL.tsx:527-571`

```typescript
export type Props = {
  commands: Command[];
  debug: boolean;
  initialTools: Tool[];
  initialMessages?: MessageType[];
  pendingHookMessages?: Promise<HookResultMessage[]>;
  initialFileHistorySnapshots?: FileHistorySnapshot[];
  initialContentReplacements?: ContentReplacementRecord[];
  initialAgentName?: string;
  initialAgentColor?: AgentColorName;
  mcpClients?: MCPServerConnection[];
  dynamicMcpConfig?: Record<string, ScopedMcpServerConfig>;
  autoConnectIdeFlag?: boolean;
  strictMcpConfig?: boolean;
  systemPrompt?: string;
  appendSystemPrompt?: string;
  onBeforeQuery?: (input: string, newMessages: MessageType[]) => Promise<boolean>;
  onTurnComplete?: (messages: MessageType[]) => void | Promise<void>;
  disabled?: boolean;
  mainThreadAgentDefinition?: AgentDefinition;
  disableSlashCommands?: boolean;
  taskListId?: string;
  remoteSessionConfig?: RemoteSessionConfig;
  directConnectConfig?: DirectConnectConfig;
  sshSession?: SSHSession;
  thinkingConfig: ThinkingConfig;
};
```

## 核心状态

### 1. 屏幕状态

> 来源: `/Users/sunxianfeng/Desktop/n-true/claude_code_book/reference/claude-code/src/screens/REPL.tsx:704`

```typescript
const [screen, setScreen] = useState<Screen>('prompt' | 'transcript');
```

两种模式：
- `prompt`: 默认交互模式，显示消息和输入框
- `transcript`: 转录模式，支持搜索和滚动浏览

### 2. 消息状态

> 来源: `/Users/sunxianfeng/Desktop/n-true/claude_code_book/reference/claude-code/src/screens/REPL.tsx:1183-1223`

```typescript
const [messages, rawSetMessages] = useState<MessageType[]>(initialMessages ?? []);
const messagesRef = useRef(messages);

// 使用 Zustand 模式: ref 是真相源，React state 是渲染投影
const setMessages = useCallback((action: React.SetStateAction<MessageType[]>) => {
  const prev = messagesRef.current;
  const next = typeof action === 'function' ? action(messagesRef.current) : action;
  messagesRef.current = next;
  rawSetMessages(next);
}, []);
```

### 3. 输入状态

> 来源: `/Users/sunxianfeng/Desktop/n-true/claude_code_book/reference/claude-code/src/screens/REPL.tsx:1332-1364`

```typescript
const [inputValue, setInputValueRaw] = useState(() => consumeEarlyInput());
const inputValueRef = useRef(inputValue);

const setInputValue = useCallback((value: string) => {
  // 自动重新固定滚动（用户开始输入时）
  if (inputValueRef.current === '' && value !== '' && ...) {
    repinScroll();
  }
  inputValueRef.current = value;
  setInputValueRaw(value);
  setIsPromptInputActive(value.trim().length > 0);
}, []);
```

### 4. 查询状态 (QueryGuard)

> 来源: `/Users/sunxianfeng/Desktop/n-true/claude_code_book/reference/claude-code/src/screens/REPL.tsx:898-917`

使用 `QueryGuard` 同步状态机替代双状态模式：

```typescript
const queryGuard = React.useRef(new QueryGuard()).current;
const isQueryActive = React.useSyncExternalStore(
  queryGuard.subscribe, 
  queryGuard.getSnapshot
);
const isLoading = isQueryActive || isExternalLoading;
```

### 5. 工具权限队列

> 来源: `/Users/sunxianfeng/Desktop/n-true/claude_code_book/reference/claude-code/src/screens/REPL.tsx:1102`

```typescript
const [toolUseConfirmQueue, setToolUseConfirmQueue] = useState<ToolUseConfirm[]>([]);
```

## 关键子组件

### TranscriptModeFooter

> 来源: `/Users/sunxianfeng/Desktop/n-true/claude_code_book/reference/claude-code/src/screens/REPL.tsx:322-363`

转录模式下的页脚，显示快捷键提示。

### TranscriptSearchBar

> 来源: `/Users/sunxianfeng/Desktop/n-true/claude_code_book/reference/claude-code/src/screens/REPL.tsx:369-473`

less 风格的搜索栏，支持增量搜索和高亮。

### AnimatedTerminalTitle

> 来源: `/Users/sunxianfeng/Desktop/n-true/claude_code_book/reference/claude-code/src/screens/REPL.tsx:485-526`

终端标题动画组件，在查询运行时显示动态前缀。

## 核心机制

### 1. 虚拟滚动适配

> 来源: `/Users/sunxianfeng/Desktop/n-true/claude_code_book/reference/claude-code/src/screens/REPL.tsx:604-609`

```typescript
const disableVirtualScroll = useMemo(
  () => isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_VIRTUAL_SCROLL), 
  []
);
```

### 2. 消息延迟渲染

> 来源: `/Users/sunxianfeng/Desktop/n-true/claude_code_book/reference/claude-code/src/screens/REPL.tsx:1316-1323`

```typescript
// 延迟消息渲染以保持输入响应
const deferredMessages = useDeferredValue(messages);
const deferredBehind = messages.length - deferredMessages.length;
```

### 3. 本地 JSX 命令

> 来源: `/Users/sunxianfeng/Desktop/n-true/claude_code_book/reference/claude-code/src/screens/REPL.tsx:1033-1101`

支持即时命令（如 `/btw`）在模型处理时持续显示：

```typescript
const setToolJSX = useCallback((args: {...} | null) => {
  if (args?.isLocalJSXCommand) {
    localJSXCommandRef.current = { ...rest, isLocalJSXCommand: true };
    setToolJSXInternal(rest);
    return;
  }
  // ...
}, []);
```

### 4. 滚动管理

> 来源: `/Users/sunxianfeng/Desktop/n-true/claude_code_book/reference/claude-code/src/screens/REPL.tsx:1293-1310`

```typescript
const composedOnScroll = useCallback((sticky: boolean, handle: ScrollBoxHandle) => {
  lastUserScrollTsRef.current = Date.now();
  if (sticky) {
    onRepin();
  } else {
    onScrollAway(handle);
    // KAIROS: 异步加载历史
    if (feature('KAIROS')) maybeLoadOlder(handle);
    // 滚动时关闭 companion bubble
    if (feature('BUDDY')) { ... }
  }
}, [...]);
```

## 导出类型

> 来源: `/Users/sunxianfeng/Desktop/n-true/claude_code_book/reference/claude-code/src/screens/REPL.tsx:527-572`

```typescript
export type Props = { ... };
export type Screen = 'prompt' | 'transcript';
export function REPL({ ... }: Props): React.ReactNode
```
