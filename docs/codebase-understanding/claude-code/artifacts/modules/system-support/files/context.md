# Context 模块详细分析

## 概述

Context 模块负责构建发送给 Claude API 的系统上下文，包括 Git 状态、CLAUDE.md 内容、内存文件等。

> 来源: `src/context.ts:1-190`

---

## 1. 核心上下文函数

### 1.1 getGitStatus

位置: `src/context.ts:36-111`

```typescript
export const getGitStatus = memoize(async (): Promise<string | null> => {
  // 检查是否为 Git 仓库
  const isGit = await getIsGit()
  if (!isGit) return null

  // 并行获取多个 Git 信息
  const [branch, mainBranch, status, log, userName] = await Promise.all([
    getBranch(),
    getDefaultBranch(),
    execFileNoThrow(gitExe(), ['--no-optional-locks', 'status', '--short'], ...),
    execFileNoThrow(gitExe(), ['--no-optional-locks', 'log', '--oneline', '-n', 5], ...),
    execFileNoThrow(gitExe(), ['config', 'user.name'], ...),
  ])

  // 截断过长的状态
  const truncatedStatus = status.length > MAX_STATUS_CHARS
    ? status.substring(0, MAX_STATUS_CHARS) + '\n... (truncated)'
    : status

  return [
    `Current branch: ${branch}`,
    `Main branch: ${mainBranch}`,
    ...(userName ? [`Git user: ${userName}`] : []),
    `Status:\n${truncatedStatus || '(clean)'}`,
    `Recent commits:\n${log}`,
  ].join('\n\n')
})
```

**特点**:
- Memoized 缓存（单次会话内）
- 使用 `--no-optional-locks` 避免锁定
- 状态截断到 2000 字符

> 来源: `src/context.ts:36-111`

### 1.2 getSystemContext

位置: `src/context.ts:116-150`

```typescript
export const getSystemContext = memoize(async (): Promise<{ [k: string]: string }> => {
  const gitStatus = isEnvTruthy(process.env.CLAUDE_CODE_REMOTE) || !shouldIncludeGitInstructions()
    ? null
    : await getGitStatus()

  const injection = feature('BREAK_CACHE_COMMAND')
    ? getSystemPromptInjection()
    : null

  return {
    ...(gitStatus && { gitStatus }),
    ...(feature('BREAK_CACHE_COMMAND') && injection ? { cacheBreaker: `[CACHE_BREAKER: ${injection}]` } : {}),
  }
})
```

**缓存清除**: 当 system prompt injection 改变时清除缓存

```typescript
export function setSystemPromptInjection(value: string | null): void {
  systemPromptInjection = value
  getUserContext.cache.clear?.()
  getSystemContext.cache.clear?.()
}
```

> 来源: `src/context.ts:116-150`

### 1.3 getUserContext

位置: `src/context.ts:155-189`

```typescript
export const getUserContext = memoize(async (): Promise<{ [k: string]: string }> => {
  const shouldDisableClaudeMd = isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_CLAUDE_MDS)
    || (isBareMode() && getAdditionalDirectoriesForClaudeMd().length === 0)

  const claudeMd = shouldDisableClaudeMd
    ? null
    : getClaudeMds(filterInjectedMemoryFiles(await getMemoryFiles()))

  setCachedClaudeMdContent(claudeMd || null)  // 供自动模式分类器使用

  return {
    ...(claudeMd && { claudeMd }),
    currentDate: `Today's date is ${getLocalISODate()}.`,
  }
})
```

> 来源: `src/context.ts:155-189`

---

## 2. CLAUDE.md 加载系统

### 2.1 文件优先级

位置: `src/utils/claudemd.ts:1-26`

```
加载顺序（最后加载 = 最高优先级）:

1. 托管内存 (/etc/claude-code/CLAUDE.md) - 全局用户指令
2. 用户内存 (~/.claude/CLAUDE.md) - 私有全局指令
3. 项目内存 (CLAUDE.md, .claude/CLAUDE.md, .claude/rules/*.md) - 代码库指令
4. 本地内存 (CLAUDE.local.md) - 私有项目指令
```

### 2.2 @include 指令

位置: `src/utils/claudemd.ts:18-25`

内存文件支持 `@` 指令包含其他文件：
```markdown
# 主 CLAUDE.md
@./relative/path.md
@~/home/user/file.md
@/absolute/path.md
```

### 2.3 核心加载函数

位置: `src/utils/claudemd.ts:100+`

```typescript
export async function getClaudeMds(
  memoryFiles: MemoryFile[]
): Promise<string | null> {
  // 按优先级排序
  // 合并内容
  // 处理 @include 指令
  // 防止循环引用
}
```

---

## 3. 内存文件发现

### 3.1 getMemoryFiles

位置: `src/utils/claudemd.ts`

发现所有可能的 CLAUDE.md 文件：
- 项目根目录搜索
- 向上遍历目录树
- 全局用户目录
- 托管规则目录

### 3.2 过滤注入文件

```typescript
export function filterInjectedMemoryFiles(
  memoryFiles: MemoryFile[]
): MemoryFile[]
```

---

## 4. 系统提示词构建

### 4.1 getSystemPrompt

位置: `src/constants/prompts.ts` 或相关文件

构建发送给 API 的完整系统提示词。

### 4.2 增强系统提示

```typescript
export function enhanceSystemPromptWithEnvDetails(
  systemPrompt: SystemPrompt
): SystemPrompt
```

---

## 5. 上下文注入

### 5.1 缓存断裂器

位置: `src/context.ts:23-34`

```typescript
let systemPromptInjection: string | null = null

export function getSystemPromptInjection(): string | null {
  return systemPromptInjection
}

export function setSystemPromptInjection(value: string | null): void {
  systemPromptInjection = value
  // 清除缓存
  getUserContext.cache.clear?.()
  getSystemContext.cache.clear?.()
}
```

### 5.2 当前日期注入

位置: `src/context.ts:186`

```typescript
currentDate: `Today's date is ${getLocalISODate()}.`
```

---

## 6. 上下文与 API

### 6.1 API 客户端使用

位置: `src/services/api/claude.ts`

上下文在构建 API 请求时使用：

```typescript
const systemContext = await getSystemContext()
const userContext = await getUserContext()

const systemPrompt = buildSystemPrompt({
  ...systemContext,
  ...userContext,
})
```

### 6.2 缓存策略

- `getGitStatus` - Memoized，单会话缓存
- `getSystemContext` - Memoized，可被 injection 清除
- `getUserContext` - Memoized，可被 injection 清除

---

## 7. 特殊上下文模式

### 7.1 --bare 模式

在 `--bare` 模式下，CLAUDE.md 自动发现被禁用：

```typescript
const shouldDisableClaudeMd = isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_CLAUDE_MDS)
  || (isBareMode() && getAdditionalDirectoriesForClaudeMd().length === 0)
```

### 7.2 --remote 模式

在 `--remote` 模式下，Git 状态获取被跳过：

```typescript
const gitStatus = isEnvTruthy(process.env.CLAUDE_CODE_REMOTE) || !shouldIncludeGitInstructions()
  ? null
  : await getGitStatus()
```

---

## 8. 诊断日志

### 8.1 诊断点

```typescript
logForDiagnosticsNoPII('info', 'git_status_started')
logForDiagnosticsNoPII('info', 'git_is_git_check_completed', { duration_ms, is_git })
logForDiagnosticsNoPII('info', 'git_status_completed', { duration_ms, truncated })
logForDiagnosticsNoPII('info', 'system_context_started')
logForDiagnosticsNoPII('info', 'user_context_started')
```

> 来源: `src/context.ts:42-93` | `src/context.ts:121-139`

---

## 9. 文件路径索引

| 文件 | 功能 |
|------|------|
| `src/context.ts` | 核心上下文函数 (Git, System, User) |
| `src/utils/claudemd.ts` | CLAUDE.md 加载和解析 |
| `src/constants/prompts.ts` | 系统提示词构建 |
| `src/bootstrap/state.ts` | 引导状态（包括 cwd, sessionId） |
| `src/utils/git.ts` | Git 命令执行 |
| `src/utils/envUtils.ts` | 环境变量检查 |
