# Tools 模块详细分析

## 概述

Claude Code 的工具系统是其核心功能之一，支持 50+ 内置工具，包括文件操作、代码搜索、Bash 执行、Web 访问等。

> 来源: `src/tools.ts:191-249`

---

## 1. 工具接口 (Tool Interface)

### 1.1 核心接口定义

位置: `src/Tool.ts:362-695`

```typescript
export type Tool<
  Input extends AnyObject = AnyObject,
  Output = unknown,
  P extends ToolProgressData = ToolProgressData,
> = {
  // 工具名称
  readonly name: string
  
  // 可选别名
  aliases?: string[]
  
  // 关键词提示（用于 ToolSearch）
  searchHint?: string
  
  // 核心执行方法
  call(
    args: z.infer<Input>,
    context: ToolUseContext,
    canUseTool: CanUseToolFn,
    parentMessage: AssistantMessage,
    onProgress?: ToolCallProgress<P>,
  ): Promise<ToolResult<Output>>
  
  // 描述生成
  description(input, options): Promise<string>
  
  // 输入验证
  validateInput?(input, context): Promise<ValidationResult>
  
  // 权限检查
  checkPermissions(input, context): Promise<PermissionResult>
  
  // 进度渲染
  renderToolUseProgressMessage?(progressMessages, options): React.ReactNode
  
  // 结果渲染
  renderToolResultMessage?(content, progressMessages, options): React.ReactNode
}
```

> 来源: `src/Tool.ts:362-695`

### 1.2 buildTool 工厂函数

位置: `src/Tool.ts:783-792`

```typescript
export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return {
    ...TOOL_DEFAULTS,
    userFacingName: () => def.name,
    ...def,
  } as BuiltTool<D>
}
```

默认实现：
- `isEnabled` → `true`
- `isConcurrencySafe` → `false`
- `isReadOnly` → `false`
- `isDestructive` → `false`
- `checkPermissions` → `{ behavior: 'allow' }`

> 来源: `src/Tool.ts:757-792`

---

## 2. 工具注册表 (Tool Registry)

### 2.1 获取所有工具

位置: `src/tools.ts:191-249`

```typescript
export function getAllBaseTools(): Tools {
  return [
    AgentTool,
    TaskOutputTool,
    BashTool,
    GlobTool,
    GrepTool,
    ExitPlanModeV2Tool,
    FileReadTool,
    FileEditTool,
    FileWriteTool,
    NotebookEditTool,
    WebFetchTool,
    TodoWriteTool,
    WebSearchTool,
    TaskStopTool,
    AskUserQuestionTool,
    SkillTool,
    EnterPlanModeTool,
    TaskCreateTool,
    TaskGetTool,
    TaskUpdateTool,
    TaskListTool,
    ListMcpResourcesTool,
    ReadMcpResourceTool,
    ToolSearchTool,
    // ... 更多工具
  ]
}
```

### 2.2 条件加载工具

位置: `src/tools.ts:14-50`

部分工具根据特性标志或环境变量条件加载：

```typescript
const REPLTool = process.env.USER_TYPE === 'ant' 
  ? require('./tools/REPLTool/REPLTool.js').REPLTool 
  : null

const SuggestBackgroundPRTool = process.env.USER_TYPE === 'ant'
  ? require('./tools/SuggestBackgroundPRTool/SuggestBackgroundPRTool.js').SuggestBackgroundPRTool
  : null

const SleepTool = feature('PROACTIVE') || feature('KAIROS')
  ? require('./tools/SleepTool/SleepTool.js').SleepTool
  : null
```

### 2.3 简单模式工具

位置: `src/tools.ts:269-296`

`--simple` 模式只暴露基础工具：

```typescript
if (isEnvTruthy(process.env.CLAUDE_CODE_SIMPLE)) {
  const simpleTools: Tool[] = [BashTool, FileReadTool, FileEditTool]
  return filterToolsByDenyRules(simpleTools, permissionContext)
}
```

### 2.4 工具池组装

位置: `src/tools.ts:343-365`

```typescript
export function assembleToolPool(
  permissionContext: ToolPermissionContext,
  mcpTools: Tools,
): Tools {
  const builtInTools = getTools(permissionContext)
  const allowedMcpTools = filterToolsByDenyRules(mcpTools, permissionContext)
  
  // 合并并去重
  return uniqBy(
    [...builtInTools].sort(byName).concat(allowedMcpTools.sort(byName)),
    'name',
  )
}
```

---

## 3. 核心工具实现

### 3.1 BashTool

**位置**: `src/tools/BashTool/BashTool.tsx` (46,642 行)

**关键文件**:
- `bashPermissions.ts` (98,830 行) - 权限处理
- `bashSecurity.ts` (102,561 行) - 安全检查
- `pathValidation.ts` (43,679 行) - 路径验证
- `readOnlyValidation.ts` (68,322 行) - 只读验证

**功能**:
- Shell 命令执行
- 沙箱执行支持
- 路径验证
- 只读命令检测
- 破坏性命令警告

> 来源: `src/tools/BashTool/BashTool.tsx:1-50`

### 3.2 FileReadTool

**位置**: `src/tools/FileReadTool/FileReadTool.ts` (1,184 行)

**功能**:
- 文本文件读取
- 图片读取（PNG, JPEG, GIF, WebP）
- PDF 读取
- Jupyter notebook 读取
- 设备文件保护（/dev/*）

```typescript
export const FileReadTool = buildTool({
  name: FILE_READ_TOOL_NAME,
  maxResultSizeChars: Infinity, // 永不持久化
  isReadOnly() { return true },
  isConcurrencySafe() { return true },
  // ...
})
```

**去重逻辑**:
如果文件未修改且已读取过相同范围，返回 `file_unchanged` 类型结果。

> 来源: `src/tools/FileReadTool/FileReadTool.ts:337-430`

### 3.3 FileEditTool

**位置**: `src/tools/FileEditTool/FileEditTool.ts` (400+ 行)

**功能**:
- 原地文件编辑
- sed 语法验证
- 变更追踪
- 文件意外修改检测

```typescript
export const FileEditTool = buildTool({
  name: FILE_EDIT_TOOL_NAME,
  maxResultSizeChars: 100_000,
  isDestructive?(input) { return true }, // 默认破坏性
  // ...
})
```

> 来源: `src/tools/FileEditTool/FileEditTool.ts:86-140`

### 3.4 GrepTool

**位置**: `src/tools/GrepTool/GrepTool.ts` (577 行)

**功能**:
- 正则表达式搜索
- Glob 过滤
- 上下文显示（-B, -A, -C）
- 多种输出模式（content, files_with_matches, count）

```typescript
const outputSchema = z.object({
  mode: z.enum(['content', 'files_with_matches', 'count']),
  numFiles: z.number(),
  filenames: z.array(z.string()),
  content: z.string().optional(),
  appliedLimit: z.number().optional(),
})
```

> 来源: `src/tools/GrepTool/GrepTool.ts:144-156`

### 3.5 GlobTool

**位置**: `src/tools/GlobTool/GlobTool.ts` (198 行)

**功能**:
- 文件名模式匹配
- 目录遍历
- 结果限制（默认 100）

```typescript
export const GlobTool = buildTool({
  name: GLOB_TOOL_NAME,
  maxResultSizeChars: 100_000,
  isConcurrencySafe() { return true },
  isReadOnly() { return true },
  // ...
})
```

> 来源: `src/tools/GlobTool/GlobTool.ts:57-75`

### 3.6 AgentTool

**位置**: `src/tools/AgentTool/AgentTool.tsx` (800+ 行)

**功能**:
- 子 agent spawning
- 多 agent 协调
- 内置 agent 类型
- Worktree 隔离

**内置 Agent**:
- `generalPurposeAgent` - 通用任务
- `planAgent` - 计划模式
- `claudeCodeGuideAgent` - 引导
- `exploreAgent` - 代码探索

> 来源: `src/tools/AgentTool/AgentTool.tsx:1-100`

### 3.7 WebSearchTool & WebFetchTool

**位置**: `src/tools/WebSearchTool/`, `src/tools/WebFetchTool/`

**功能**:
- Web 搜索
- Web 内容获取
- 代理支持

---

## 4. MCP 工具集成

### 4.1 MCP 工具注册

位置: `src/services/mcp/types.ts`

```typescript
export type MCPServerConnection = {
  name: string
  tools: Tool[]
  resources: ServerResource[]
  commands: Command[]
}
```

### 4.2 工具去重

位置: `src/tools.ts:361-364`

MCP 工具与内置工具同名时，内置工具优先：

```typescript
return uniqBy(
  [...builtInTools].sort(byName).concat(allowedMcpTools.sort(byName)),
  'name',
)
```

---

## 5. 工具搜索 (Tool Search)

### 5.1 延迟加载

位置: `src/tools/ToolSearchTool/`

当启用 ToolSearch 时，部分工具以 `defer_loading: true` 发送：

```typescript
export type Tool = {
  readonly shouldDefer?: boolean
  readonly alwaysLoad?: boolean
}
```

### 5.2 关键词提示

位置: `src/Tool.ts:373-378`

```typescript
searchHint?: string  // 3-10 words for keyword matching
```

---

## 6. 文件路径索引

| 工具 | 文件路径 |
|------|----------|
| BashTool | `src/tools/BashTool/BashTool.tsx` |
| FileReadTool | `src/tools/FileReadTool/FileReadTool.ts` |
| FileEditTool | `src/tools/FileEditTool/FileEditTool.ts` |
| FileWriteTool | `src/tools/FileWriteTool/FileWriteTool.ts` |
| GrepTool | `src/tools/GrepTool/GrepTool.ts` |
| GlobTool | `src/tools/GlobTool/GlobTool.ts` |
| AgentTool | `src/tools/AgentTool/AgentTool.tsx` |
| WebSearchTool | `src/tools/WebSearchTool/WebSearchTool.ts` |
| WebFetchTool | `src/tools/WebFetchTool/WebFetchTool.ts` |
| ToolSearchTool | `src/tools/ToolSearchTool/ToolSearchTool.ts` |
| TodoWriteTool | `src/tools/TodoWriteTool/TodoWriteTool.ts` |
| TaskOutputTool | `src/tools/TaskOutputTool/TaskOutputTool.ts` |
