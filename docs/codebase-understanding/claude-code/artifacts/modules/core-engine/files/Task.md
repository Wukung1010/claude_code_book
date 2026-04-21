# Task.ts / tasks.ts

## 基本信息
- 路径: src/Task.ts, src/tasks.ts
- 模块: core-engine
- 复杂度: 中
- 行数: ~125 + ~40 行

## 职责
定义任务类型系统和任务注册表。任务系统支持本地/远程 agent、工作流、monitor 等多种执行模式。

## Task 类型定义 (Task.ts)

### TaskType (L6-76)
```typescript
export type TaskType =
  | 'local_bash'    // 'b' prefix
  | 'local_agent'   // 'a' prefix
  | 'remote_agent'  // 'r' prefix
  | 'in_process_teammate'  // 't' prefix
  | 'local_workflow'       // 'w' prefix
  | 'monitor_mcp'         // 'm' prefix
  | 'dream'               // 'd' prefix
```

### Task 接口 (L72-76)
```typescript
export type Task = {
  name: string
  type: TaskType
  kill(taskId: string, setAppState: SetAppState): Promise<void>
}
```
**注意**: `spawn` 和 `render` 方法已被移除（#22546），现在只保留 `kill` 方法。

### Task ID 生成 (L98-106)
- 使用 `crypto.randomBytes(8)` 生成 8 字节随机数据
- 36^8 ≈ 2.8 万亿种组合，防暴力破解

## Task 注册 (tasks.ts)

### getAllTasks() (L22-39)
```typescript
export function getAllTasks(): Task[] {
  return [
    LocalShellTask,   // local_bash
    LocalAgentTask,   // local_agent
    RemoteAgentTask,  // remote_agent
    DreamTask,        // dream
    ...(LocalWorkflowTask if WORKFLOW_SCRIPTS enabled)
    ...(MonitorMcpTask if MONITOR_TOOL enabled)
  ]
}
```

## 与其他模块的联系
- **使用**: 被 `Task.ts` 中的任务系统使用
- **注册**: `tasks.ts` 聚合所有任务
- **协调器**: `coordinator/` 中的任务使用 Task 类型

> 来源: src/Task.ts:6, 72, 98 | src/tasks.ts:22
