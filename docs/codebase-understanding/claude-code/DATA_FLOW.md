# Claude Code 数据流分析

## 核心流程

### 1. 用户输入 → API 响应
```
User Input (终端)
    ↓
REPL.tsx (接收输入)
    ↓
QueryEngine.submitMessage() (会话管理)
    ↓
query() (核心引擎)
    ↓
API 调用 (services/api/claude.ts)
    ↓
流式响应处理
    ↓
工具执行 (StreamingToolExecutor / runTools)
    ↓
AssistantMessage (渲染)
    ↓
MessageRow → Messages → 终端输出
```

### 2. 工具调用循环
```
API 响应 (包含 tool_use)
    ↓
canUseTool() (权限检查)
    ↓
Tool.call() (工具执行)
    ↓
ToolResult (结果)
    ↓
提交结果给 API
    ↓
继续响应或结束
```

### 3. 上下文构建流程
```
getSystemContext()
    ├── getGitStatus() → branch, status, log
    └── getUserContext()
        ├── getClaudeMds() → CLAUDE.md 内容
        └── getMemoryFiles() → 内存文件

→ 组装 system prompt → API 调用
```

## 关键数据模型

### Message 类型
```typescript
export type Message = {
  type: 'user' | 'assistant' | 'system' | 'attachment' | 'progress'
  uuid: UUID
  isMeta?: boolean
  isCompactSummary?: boolean
  toolUseResult?: unknown
  message?: {
    role?: string
    content?: MessageContent
    usage?: BetaUsage
  }
}
```

### AppState
```typescript
export type AppState = DeepImmutable<{
  settings: SettingsJson
  tasks: { [taskId: string]: TaskState }
  mcp: { clients, tools, commands, resources }
  plugins: { ... }
  fileHistory: FileHistoryState
}>
```

### Tool 接口
```typescript
export interface Tool {
  name: string
  call(args, context, canUseTool, parentMessage, onProgress): Promise<ToolResult>
  description(input, options): Promise<string>
  checkPermissions(input, context): Promise<PermissionResult>
}
```

## 模块间数据流

| 流程 | 起点 | 终点 | 数据格式 |
|------|------|------|----------|
| 用户输入 | REPL.tsx | QueryEngine | string |
| 会话提交 | QueryEngine | query() | ProcessUserInputContext |
| API 调用 | query() | services/api | messages[], tools[] |
| 工具执行 | query() | Tool.call() | Tool arguments |
| 状态更新 | AppStateProvider | 组件 | AppState |
| 权限检查 | canUseTool | PermissionResult | allow/deny |
| 上下文 | context.ts | query() | system prompt |
