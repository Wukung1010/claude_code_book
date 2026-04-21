# Claude Code 代码理解总览

## 一句话描述
Claude Code 是 Anthropic 官方 Claude Code CLI 的反向工程实现，一个终端 AI 编程助手，支持流式对话、50+ 工具调用、复杂权限管理和多 Provider API（Anthropic/Bedrock/Vertex/Foundry）。

## 报告索引
- [项目概述](PROJECT_OVERVIEW.md)
- [模块划分](MODULES.md)
- [数据流](DATA_FLOW.md)
- [关键发现](KEYFINDINGS.md)

## 快速结论
1. **入口**: `src/entrypoints/cli.tsx` → `src/main.tsx` → `src/screens/REPL.tsx`
2. **核心引擎**: query.ts (1730行) 处理 API 调用/工具循环/上下文压缩，QueryEngine.ts (1320行) 管理会话状态
3. **工具系统**: 50+ 工具实现（BashTool/FileEditTool/GrepTool/AgentTool 等），Tool 接口标准化
4. **UI 渲染**: 自定义 Ink 框架（基于 React Reconciler + Yoga），双缓冲渲染
5. **权限系统**: 6300+ 行，支持 plan/auto/manual/yolo 模式，路径验证和规则匹配
6. **状态管理**: Zustand 风格 store，AppState 包含 settings/tasks/MCP/plugins 等
7. **多 Provider**: 支持 Anthropic Direct、AWS Bedrock、Google Vertex、Azure Foundry
8. **上下文**: Git 状态、CLAUDE.md、memory 文件自动加载
9. **Task 系统**: 7 种任务类型（local_bash/agent/remote_agent/workflow 等）
10. **Feature Flags**: `feature()` 统一管理，bundled build 时注入

## 项目统计
- 总文件数: 2782 TS/TSX 文件
- 总模块数: 3 大模块（core-engine, repl-ui, system-support）
- 入口文件: src/entrypoints/cli.tsx
- 核心依赖: @anthropic-ai/sdk, react, ink (custom), zustand
