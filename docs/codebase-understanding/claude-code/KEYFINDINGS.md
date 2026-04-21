# Claude Code 关键发现

## 发现 1: 自定义 Ink 渲染框架

Claude Code 使用自定义的 Ink 框架（不是 npm 包），基于 React Reconciler 构建，专门为终端环境优化。它使用 Yoga 布局引擎实现 Flexbox 布局，支持双缓冲渲染（frontFrame/backFrame）避免闪烁，支持文本选择、搜索高亮、外部编辑器暂停等高级功能。

> 来源: `src/ink/ink.tsx:96-105` | `src/ink.ts`

## 发现 2: query.ts 主循环采用 while(true) 结构

核心查询引擎 query.ts (1730行) 使用 while(true) 主循环结构，交替执行 API 调用和工具处理。工具执行支持两种模式：StreamingToolExecutor（流式并行）和 runTools（批量），通过 feature flag 控制。

> 来源: `src/query.ts:307-1731` | `src/query.ts:1367-1411`

## 发现 3: 完善的错误恢复机制

query.ts 实现了多层错误恢复：
- `prompt-too-long` → Context Collapse → Reactive Compact
- `max-output-tokens` → 升级到 64k 或注入恢复消息
- 媒体大小错误 → Reactive Compact strip-retry

> 来源: `src/query.ts:1073-1259`

## 发现 4: Task 系统支持 7 种任务类型

Task.ts 定义了 7 种任务类型：local_bash, local_agent, remote_agent, in_process_teammate, local_workflow, monitor_mcp, dream。Task ID 使用 crypto.randomBytes(8) 生成，提供 2.8 万亿种组合。

> 来源: `src/Task.ts:6-76` | `src/Task.ts:98-106`

## 发现 5: 权限系统 6300+ 行

权限系统非常复杂，支持多种模式（default/plan/yolo/bypass），包含路径验证、危险文件保护、shell 命令语义分析。BashTool 的 bashPermissions.ts 就有 98,830 行。

> 来源: `src/utils/permissions/PermissionMode.ts` | `src/tools/BashTool/bashPermissions.ts`

## 发现 6: 多 Provider API 抽象

services/api/ 支持 4 种 API Provider：Anthropic Direct、AWS Bedrock、Google Vertex、Azure Foundry。通过环境变量（CLAUDE_CODE_USE_BEDROCK/VERTEX/FOUNDRY）切换认证方式。

> 来源: `src/services/api/client.ts:88-316`

## 发现 7: Feature Flag 系统统一管理

所有 feature flags 通过 `feature()` 函数统一管理，来自 `bun:bundle`。当前所有 flags 返回 false（特性未启用），任何依赖 feature flag 的代码都是 dead code。

> 来源: `src/entrypoints/cli.tsx` | `src/types/internal-modules.d.ts`

## 发现 8: Zustand 风格状态管理

state/ 实现了 Zustand 风格的状态管理，createStore<T> 创建 store，支持 getState/setState/subscribe。AppState 是核心状态容器，包含 settings/tasks/MCP/plugins 等。

> 来源: `src/state/store.ts:1-34` | `src/state/AppStateStore.ts:89-452`

## 发现 9: Coordinator 协调器模式

coordinatorMode.ts 实现了协调器系统提示，定义 Workers 并行研究 → Coordinator 制定规格 → 实现 → 验证的工作流程。使用 TEAM_CREATE_TOOL_NAME 等工具协调。

> 来源: `src/coordinator/coordinatorMode.ts:111-369`

## 发现 10: 虚拟滚动处理大会话

VirtualMessageList 组件实现虚拟滚动，仅渲染可见区域消息，支持大会话（数千条消息）的高效渲染和滚动位置记忆。

> 来源: `src/components/VirtualMessageList.ts`
