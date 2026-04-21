# Claude Code 项目概述

## 项目基本信息
- 项目名称: Claude Code Best V3 (CCB)
- 类型: CLI 工具 / AI 编程助手
- 规模: 2782 TS/TSX 文件，~100 万行代码
- 描述: Anthropic 官方 Claude Code CLI 的反向工程实现

## 技术栈
| 类别 | 技术 | 版本 |
|------|------|------|
| 运行时 | Bun | >=1.2.0 |
| 语言 | TypeScript | - |
| 构建 | bun build | code splitting |
| UI 框架 | React + Ink (自定义) | React Reconciler |
| 布局引擎 | Yoga | Flexbox |
| 状态管理 | Zustand 风格 | 自定义实现 |
| API SDK | @anthropic-ai/sdk | ^0.80.0 |

## 目录结构
```
src/
├── entrypoints/        # CLI 入口点
├── main.tsx           # Commander.js CLI 定义
├── query.ts          # 核心查询引擎 (1730行)
├── QueryEngine.ts    # 会话管理器 (1320行)
├── context.ts        # 上下文构建
├── Tool.ts           # 工具接口定义
├── tools.ts          # 工具注册表
├── tools/            # 50+ 工具实现
├── screens/          # 界面屏幕
│   └── REPL.tsx      # 主交互界面 (5000+行)
├── ink/              # 自定义 Ink 渲染框架
├── components/       # React 组件
├── state/            # 状态管理
├── services/api/     # API 客户端
├── utils/permissions/# 权限系统
├── coordinator/      # 协调器模式
├── tasks.ts          # 任务注册表
├── Task.ts          # Task 类型定义
└── types/            # TypeScript 类型定义
```

## 架构特点
1. **自定义 Ink 框架**: 基于 React Reconciler，为终端优化的渲染引擎
2. **双缓冲渲染**: frontFrame/backFrame 交换，避免闪烁
3. **模块化工具系统**: 标准化 Tool 接口，条件加载
4. **多 Provider API**: 抽象 API 层，支持 4 种后端
5. **Feature Flag 系统**: `feature()` 统一管理所有特性开关
6. **状态管理**: Zustand 风格，immutable state，useSyncExternalStore 集成
