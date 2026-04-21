# codebase-understanding skill 优化任务

## 任务列表

| # | 任务 | 描述 | 状态 | 待优化内容 |
|---|------|------|------|------------|
| 2 | 精简输出格式模板 | 将30行模板改为 checklist | pending | 改为关键问题列表，更实用 |
| 3 | 增加理解验证步骤 | 确认理解正确性的机制 | pending | 理解完成后验证机制 |
| 4 | 区分不同规模项目的时间预算 | 小/中/大型项目指导 | pending | 不同规模项目的时间估算 |
| 5 | 优化探索命令可操作性 | 修复 zsh 转义问题 | pending | 修复 zsh 转义，补充实用命令 |
| 6 | 增加 Agent 协同指导 | Explore agent vs Grep/Glob/Read | pending | 何时用 Explore vs Grep/Glob/Read |
| 7 | 补充触发条件（When to Use） | 明确使用/不使用场景 | pending | 增加反例场景 |

## 优化方向汇总

### 1. 触发条件 (When to Use)
- 明确何时应该使用此 skill
- 增加反例：哪些场景不应该使用

### 2. 输出格式
- 当前：30行完整文档模板
- 目标：改为 checklist 式的关键问题列表

### 3. Agent 协同
- 何时用 Explore agent（复杂探索）
- 何时用 Grep/Glob/Read（简单任务）

### 4. 验证步骤
- 增加理解验证机制
- 确认理解正确性

### 5. 命令优化
- 修复 zsh 转义问题（如 `|` 需要引号）
- 补充更实用的命令示例

### 6. 时间预算
- 小型项目（< 1万行）
- 中型项目（1-10万行）
- 大型项目（> 10万行）

## 进度记录

- 2026-04-16: 任务列表创建
- 2026-04-16: Task #7 完成 - 触发条件优化，明确使用 Explore subagent 执行，增加反例场景
- 2026-04-16: Task #4 完成 - 执行流程细化为3阶段9步骤，去除时间限制，改为完成标志定义边界
- 2026-04-16: Task #2 完成 - 输出格式改为三层报告结构（总报告/模块报告/文件报告），目录 `docs/codebase-understanding/[monorepo]/`
- 2026-04-16: Task #3 完成 - 验证步骤明确为"交叉验证"，测试文件可读则用，无则跳，不执行测试
- 2026-04-16: Task #5 完成 - Skill frontmatter 增加 allowed-tools: Read, Write, Glob, Grep（路径权限限制暂不支持）
- 2026-04-16: Task #6 完成 - When to Use 中明确使用 Explore subagent 执行
- 2026-04-17: Agent 协同优化 - 增加任务分发机制（并发控制N=3）、TASK_QUEUE.md 模板、subagent prompt 模板
- 2026-04-17: 代码审查修复 - 删除重复的 Step 9，在约束部分增加"不执行测试"，移除未使用的 progress.md
