# query.ts

## 基本信息
- 路径: src/query.ts
- 模块: core-engine
- 复杂度: 高
- 行数: ~1730 行

## 职责
核心查询引擎，负责与 Claude API 通信、处理工具调用循环、管理上下文压缩和错误恢复。

## 关键内容
- 导出: `query()` 函数
- 依赖: `@anthropic-ai/sdk`, `src/types/`, `src/tools.ts`

## 核心逻辑

### 主循环结构 (L307-1731)
```typescript
while(true) {
  → 上下文准备 (microcompact, snip, context collapse)
  → API 调用 (deps.callModel)
  → 流式处理
  → 工具执行
  → 递归继续或终止
}
```

### 工具执行模式 (L1367-1411)
- `StreamingToolExecutor`：流式执行，API流式响应期间并行执行工具
- `runTools`：批量执行

### 错误恢复机制
| 错误类型 | 恢复策略 | 位置 |
|---------|---------|------|
| prompt-too-long | Context Collapse → Reactive Compact | L1073-1178 |
| max-output-tokens | 升级到 64k 或注入恢复消息 | L1188-1259 |
| 媒体大小错误 | Reactive Compact strip-retry | L1118-1178 |

### Feature Flags
- `HISTORY_SNIP`: 历史裁剪
- `CONTEXT_COLLAPSE`: 上下文折叠
- `REACTIVE_COMPACT`: 响应式压缩
- `TOKEN_BUDGET`: Token 预算管理

## 与其他模块的联系
- **调用**: `QueryEngine.ts` 调用 `query()`
- **工具**: 调用 `StreamingToolExecutor` / `runTools` 执行工具
- **API**: 通过 `deps.callModel` 调用 SDK

> 来源: src/query.ts:307, 401, 440, 615, 1311, 1367
