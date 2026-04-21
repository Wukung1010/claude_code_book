# 核心引擎模块总览

## 一句话描述
核心引擎模块包含 query.ts (1730行) 和 QueryEngine.ts (1320行)，负责 API 调用、工具执行循环、上下文压缩和会话状态管理。

## 快速结论
1. `query.ts` 是核心查询引擎，通过 `while(true)` 主循环处理 API 调用、工具执行和上下文压缩
2. `QueryEngine` 封装 `query()`，管理会话状态、transcript 和权限拒绝追踪
3. 工具执行支持两种模式：`StreamingToolExecutor`（流式）和 `runTools`（批量）
4. 错误恢复机制完善：prompt-too-long 触发 Context Collapse，max-output-tokens 升级到 64k
5. Task 系统支持 7 种任务类型（local_bash, local_agent, remote_agent 等）
6. Coordinator 协调器通过 `COORDINATOR_MODE` feature flag 控制，支持并行 Workers
7. 消息类型定义在 `src/types/message.ts`，包含 user/assistant/system/attachment/progress

## 核心 API
| API | 职责 | 文件 |
|-----|------|------|
| query() | 核心查询引擎，处理工具调用循环 | query.ts |
| QueryEngine.submitMessage() | 会话提交入口，管理生命周期 | QueryEngine.ts |
| getAllTasks() | 返回所有注册任务 | tasks.ts |
| isCoordinatorMode() | 检查协调器模式 | coordinatorMode.ts |

## 模块依赖
- 上游: tools.ts, services/api, types/message.ts
- 下游: screens/REPL.tsx, coordinator/

## 关键发现索引
- [query.ts 主循环结构](files/query.md)
- [QueryEngine submitMessage 流程](files/QueryEngine.md)
- [Task 类型系统](files/Task.md)
