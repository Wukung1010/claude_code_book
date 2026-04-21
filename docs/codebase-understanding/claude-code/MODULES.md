# Claude Code 模块划分

## 模块总览
| 模块 | 路径 | 职责 |
|------|------|------|
| core-engine | query.ts, QueryEngine.ts, Task.ts, coordinator/ | API 调用、工具循环、会话管理、任务协调 |
| repl-ui | screens/, ink/, components/, replLauncher.tsx | 终端 UI 渲染、用户交互、消息显示 |
| system-support | tools/, state/, permissions/, context.ts, services/api/ | 工具实现、状态管理、权限控制、上下文构建 |

## 核心引擎 (core-engine)

### 核心文件
- `src/query.ts` (1730行): 核心查询引擎，while(true) 主循环处理 API 调用和工具执行
- `src/QueryEngine.ts` (1320行): 会话生命周期管理，submitMessage() 入口
- `src/Task.ts`: Task 类型定义，7 种任务类型
- `src/tasks.ts`: 任务注册表
- `src/coordinator/`: 协调器模式，支持并行 Workers

### 核心 API
| API | 职责 |
|-----|------|
| query() | API 调用、工具循环、上下文压缩 |
| QueryEngine.submitMessage() | 会话提交入口 |
| getAllTasks() | 返回所有注册任务 |
| isCoordinatorMode() | 检查协调器模式 |

## REPL/UI (repl-ui)

### 核心文件
- `src/screens/REPL.tsx` (5000+行): 主交互界面
- `src/ink/`: 自定义终端渲染框架
- `src/components/`: React 组件
- `src/replLauncher.tsx`: REPL 启动器

### 核心 API
| API | 职责 |
|-----|------|
| Ink class | 渲染引擎核心 |
| REPL.tsx | 主界面状态和交互 |
| VirtualMessageList | 虚拟滚动消息列表 |

## 系统支持 (system-support)

### 核心文件
- `src/tools/`: 50+ 工具实现
- `src/Tool.ts`: 工具接口定义
- `src/state/`: Zustand 风格状态管理
- `src/utils/permissions/`: 权限系统
- `src/context.ts`: 上下文构建
- `src/services/api/`: API 客户端

### 核心 API
| API | 职责 |
|-----|------|
| getAllBaseTools() | 返回所有可用工具 |
| createStore() | 创建状态 store |
| AppStateProvider | React Context 状态提供 |
| getSystemContext() | 获取 Git 状态 |
| getAnthropicClient() | 获取 API 客户端 |
