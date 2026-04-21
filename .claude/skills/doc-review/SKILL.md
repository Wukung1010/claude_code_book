---
name: doc-review
description: Use after codebase-understanding completes to review and iteratively improve generated documentation based on user feedback
allowed-tools: Read, Write, Glob, Grep, Bash
---

# Documentation Review

## Overview

在代码理解文档生成完毕后，系统化审查文档质量，识别缺失，以新人视角评估文档实用性。通过迭代式提问和修正，直到文档满足维护者需求。

**核心原则：文档是给未来的自己看的，要解决"我怎么用"而不是"代码怎么写"的问题。**

## When to Use

**执行时机：codebase-understanding 完成并生成文档后自动触发**

适用场景：
- 文档生成完毕，需要最终审查
- 用户要求改进文档质量
- 项目交接，需要完善文档

## 执行流程

```
文档生成完毕
    │
    ▼
Phase 1: 文档审查（新人视角）
    │
    ▼
Phase 2: 识别问题并提问
    │
    ▼
用户回答问题
    │
    ▼
根据回答修正文档
    │
    ▼
再次审查（循环）
    │
    ▼
用户确认无问题 → 完成
```

## Phase 1: 文档审查

**以新人维护者视角审查，评估维度：**

| 维度 | 权重 | 审查问题 |
|------|------|----------|
| 实用性 | 30% | 文档能告诉我"怎么用"吗？ |
| 完整性 | 25% | 缺少哪些关键内容？ |
| 准确性 | 20% | 描述是否与代码一致？ |
| 可读性 | 15% | 结构清晰，容易理解吗？ |
| 可维护性 | 10% | 未来如何更新文档？ |

**新人必问清单（按优先级）：**

### P0 - 必须有（缺失则文档不合格）
1. **Quick Start 示例** - 有没有 3-5 行代码创建表格的示例？
2. **API 文档** - NCell 类的 public 方法、事件、配置项有没有说明？
3. **packages/ 和 src/ 的关系** - 为什么有两套目录？以哪个为准？

### P1 - 应该有（缺失会影响使用）
4. **配置项说明** - NCellOptions 的每个配置项含义和默认值？
5. **插件开发指南** - 如何写一个自定义插件？
6. **常见问题/调试** - 单元格不显示怎么排查？

### P2 - 最好有（提升体验）
7. **设计意图** - 为什么不直接用现成库，要自研？
8. **目录结构说明** - 为什么这样组织代码？
9. **测试/发布流程** - 怎么运行测试？怎么发布？

## Phase 2: 提问与修正

### 提问策略

**不要一次问所有问题**，按优先级分轮提问：

**第一轮（必问）：**
```
作为接手这个项目的新人，我最关心的是：
1. 文档有没有 Quick Start 示例？（创建表格、设值、监听事件）
2. NCell 类的 public API 是否有说明？
3. packages/ 和 src/ 的关系是什么？

请针对以上三点回答，我将根据你的回答补充到文档中。
```

**第二轮（根据回答再问）：**
- 如果缺少 Quick Start → 询问"你希望示例覆盖哪些场景？"
- 如果缺少 API 文档 → 询问"你希望 API 文档详细到什么程度？"
- 如果 packages/src 关系不清 → 询问"这两套目录的实际关系是什么？"

### 修正文档

根据用户回答，使用 Write/Edit 工具补充或修正文档：

**补充 Quick Start 示例：**
```markdown
## Quick Start

### 创建第一个表格

```javascript
import NCell, { buildCellSheet } from 'n-cell'

// 创建容器
const container = document.querySelector('#app')

// 构建表样（可选）
const sheet = buildCellSheet()

// 创建实例
const cell = new NCell(container, sheet)

// 设置单元格值
cell.setData(0, 0, 'Hello NCell')

// 监听选择变更
cell.on('selectionchange', (selections) => {
  console.log('选中了:', selections)
})
```
```

**补充 API 文档：**
```markdown
## API 文档

### NCell 类

#### 构造函数
```typescript
new NCell(element: HTMLElement, sheet?: SheetConfig)
```

#### 核心方法
| 方法 | 说明 | 示例 |
|------|------|------|
| setData(row, col, value) | 设置单元格值 | cell.setData(0, 0, 'text') |
| getData(row, col) | 获取单元格值 | cell.getData(0, 0) |
| on(event, handler) | 订阅事件 | cell.on('selectionchange', fn) |

#### 事件
| 事件 | 触发时机 |
|------|----------|
| selectionchange | 选择区域变更 |
| somethingchange | 任意数据变更 |
```
```

## 审查报告模板

```markdown
# 文档审查报告

## 审查信息
- 审查时间: [时间]
- 审查者视角: 新人维护者
- 项目: [项目名]

## 维度评分

| 维度 | 分数 | 说明 |
|------|------|------|
| 实用性 | X/10 | [说明] |
| 完整性 | X/10 | [说明] |
| 准确性 | X/10 | [说明] |
| 可读性 | X/10 | [说明] |
| 可维护性 | X/10 | [说明] |

## 缺失项（按优先级）

### P0 - 必须补充
- [ ] [缺失项1]
- [ ] [缺失项2]

### P1 - 应该补充
- [ ] [缺失项1]
- [ ] [缺失项2]

### P2 - 建议补充
- [ ] [缺失项1]
```

## 迭代流程

1. **第一轮审查** → 生成审查报告，识别 P0-P2 缺失项
2. **第一轮提问** → 向用户提出 P0 问题
3. **第一轮修正** → 根据回答补充文档
4. **第二轮审查** → 检查补充内容
5. **第二轮提问** → 向用户提出 P1-P2 问题
6. **第二轮修正** → 继续补充
7. **循环** → 直到用户确认无 P0-P1 问题

## 退出条件

文档满足以下条件时可退出迭代：
- ✅ P0 问题全部解决
- ✅ P1 问题大部分解决（≥80%）
- ✅ 用户确认无更多问题

## 约束

- **只修改文档，不修改代码**
- **每轮最多问 3 个问题**，避免信息过载
- **问题要具体**，不是泛泛而问
- **根据回答修正**，不要预设答案
