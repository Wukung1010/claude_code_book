# QueryEngine.ts

## 基本信息
- 路径: src/QueryEngine.ts
- 模块: core-engine
- 复杂度: 高
- 行数: ~1320 行

## 职责
会话生命周期管理器，封装 `query()` 函数，管理会话状态、transcript、归因和权限拒绝追踪。

## 关键导出
- `QueryEngine` 类

## 关键方法

### submitMessage() 流程 (L211-1181)
1. 初始化阶段 - 构建 `ProcessUserInputContext`
2. 用户输入处理 - 调用 `processUserInput()`
3. Transcript 预记录 - 确保 kill-mid-request 可恢复
4. API 调用循环 - 调用 `query()` 并处理输出
5. 结果分类 - 检查成功/错误状态

### Permission Denial 追踪 (L246-274)
```typescript
const wrappedCanUseTool: CanUseToolFn = async (tool, input, ...) => {
  const result = await canUseTool(...)
  if (result.behavior !== 'allow') {
    this.permissionDenials.push({
      type: 'permission_denial',
      tool_name: sdkCompatToolName(tool.name),
      tool_use_id: toolUseID,
      tool_input: input,
    })
  }
  return result
}
```

### 结果类型 (L622-641)
| subtype | 含义 |
|---------|------|
| `success` | 成功完成 |
| `error_max_turns` | 达到最大轮数 |
| `error_max_budget_usd` | 超出 USD 预算 |
| `error_max_structured_output_retries` | 结构化输出重试超限 |
| `error_during_execution` | 执行期间错误 |

## 核心属性
- `mutableMessages`: 可变消息列表
- `totalUsage`: Token 使用统计
- `permissionDenials`: 权限拒绝记录

## 与其他模块的联系
- **调用**: 内部调用 `query()` 函数
- **状态**: 使用 `AppState` 管理状态
- **输出**: 产生 `SDKMessage` 输出

> 来源: src/QueryEngine.ts:211, 246, 622, 1006, 1049, 1107
