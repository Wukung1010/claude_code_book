# Ink 渲染框架详细分析

## 概述

`src/ink/` 是 Claude Code 自定义的终端渲染框架，基于 React Reconciler 构建，提供类 React 开发体验但在终端环境中运行。

## 核心文件

### ink.ts - 入口和导出

> 来源: `/Users/sunxianfeng/Desktop/n-true/claude_code_book/reference/claude-code/src/ink.ts`

```typescript
// 主题包装器 - 所有渲染调用都包裹在 ThemeProvider 中
function withTheme(node: ReactNode): ReactNode {
  return createElement(ThemeProvider, null, node);
}

export async function render(
  node: ReactNode,
  options?: NodeJS.WriteStream | RenderOptions,
): Promise<Instance> {
  return inkRender(withTheme(node), options);
}

export async function createRoot(options?: RenderOptions): Promise<Root> {
  const root = await inkCreateRoot(options);
  return {
    ...root,
    render: node => root.render(withTheme(node)),
  };
}
```

导出组件：Box, Text, Button, Link, Newline, Spacer, RawAnsi, NoSelect
导出钩子：useInput, useStdin, useApp, useTerminalFocus, useTerminalSize, useTabStatus, useTerminalTitle, useTerminalViewport
导出工具：color, Ansi, FocusManager, measureElement, wrapText

### ink.tsx - Ink 主类

> 来源: `/Users/sunxianfeng/Desktop/n-true/claude_code_book/reference/claude-code/src/ink/ink.tsx:76-1633`

`Ink` 类是渲染引擎核心，管理终端会话的完整生命周期。

#### 核心字段

```typescript
private readonly log: LogUpdate;           // 日志更新器
private readonly terminal: Terminal;       // 终端接口
private scheduleRender: () => void;        // 渲染调度
private container: FiberRoot;              // React 容器
private rootNode: dom.DOMElement;          // 根 DOM 节点
readonly focusManager: FocusManager;        // 焦点管理
private renderer: Renderer;                // 渲染器
private stylePool: StylePool;              // 样式池
private charPool: CharPool;                // 字符池
private hyperlinkPool: HyperlinkPool;      // 超链接池
private frontFrame: Frame;                 // 前缓冲区
private backFrame: Frame;                  // 后缓冲区
readonly selection: SelectionState;         // 选择状态
```

#### 关键方法

**render()** - 渲染 React 节点
> 来源: `/Users/sunxianfeng/Desktop/n-true/claude_code_book/reference/claude-code/src/ink/ink.tsx:1439-1449`

```typescript
render(node: ReactNode): void {
  this.currentNode = node;
  const tree = <App stdin={...} stdout={...} ...>
    <TerminalWriteProvider value={this.writeRaw}>
      {node}
    </TerminalWriteProvider>
  </App>;
  reconciler.updateContainerSync(tree, this.container, null, noop);
}
```

**onRender()** - 每帧渲染回调
> 来源: `/Users/sunxianfeng/Desktop/n-true/claude_code_book/reference/claude-code/src/ink/ink.tsx:418-787`

主要职责：
1. 计算布局 (Yoga)
2. 渲染到屏幕
3. 应用选择/搜索高亮
4. 计算 diff
5. 写入终端

**enterAlternateScreen() / exitAlternateScreen()** - 外部编辑器支持
> 来源: `/Users/sunxianfeng/Desktop/n-true/claude_code_book/reference/claude-code/src/ink/ink.tsx:355-417`

用于在外部编辑器（如 vim）运行时暂停 Ink。

**setAltScreenActive()** - 切换 Alt Screen 模式
> 来源: `/Users/sunxianfeng/Desktop/n-true/claude_code_book/reference/claude-code/src/ink/ink.tsx:858-867`

控制光标位置限制和 Alt Screen 相关行为。

## ink/root.ts - React DOM 风格 API

> 来源: `/Users/sunxianfeng/Desktop/n-true/claude_code_book/reference/claude-code/src/ink/root.ts`

```typescript
export async function createRoot({ ... }: RenderOptions = {}): Promise<Root> {
  await Promise.resolve();
  const instance = new Ink({ ... });
  instances.set(stdout, instance);
  return {
    render: node => instance.render(node),
    unmount: () => instance.unmount(),
    waitUntilExit: () => instance.waitUntilExit(),
  };
}
```

## ink/screen.ts - 屏幕管理

> 来源: `/Users/sunxianfeng/Desktop/n-true/claude_code_book/reference/claude-code/src/ink/screen.ts`

### 双缓冲架构

```typescript
// frontFrame: 当前显示的帧
// backFrame: 下一帧的缓冲区
this.frontFrame = emptyFrame(this.terminalRows, this.terminalColumns, ...);
this.backFrame = emptyFrame(this.terminalRows, this.terminalColumns, ...);

// 每帧渲染后交换
this.backFrame = this.frontFrame;
this.frontFrame = frame;
```

### diff 算法

> 来源: `/Users/sunxianfeng/Desktop/n-true/claude_code_book/reference/claude-code/src/ink/screen.ts:1126-1154`

```typescript
export function diff(prev: Screen, next: Screen): Patch[] {
  // 计算两帧之间的差异，生成最小补丁
}
```

## ink/selection.ts - 文本选择

> 来源: `/Users/sunxianfeng/Desktop/n-true/claude_code_book/reference/claude-code/src/ink/selection.ts`

`SelectionState` 管理终端文本选择：
- `anchor`: 选择起点
- `focus`: 选择终点
- `isDragging`: 是否正在拖拽
- `anchorSpan`: 选择类型 (char/word/line)

支持操作：
- `startSelection()`: 开始选择
- `updateSelection()`: 更新选择
- `extendSelection()`: 扩展选择
- `shiftSelection*()`: 滚动时移动选择
- `clearSelection()`: 清除选择

## ink/focus.ts - 焦点管理

> 来源: `/Users/sunxianfeng/Desktop/n-true/claude_code_book/reference/claude-code/src/ink/focus.ts`

管理终端中的焦点切换：
- Tab/Shift+Tab 循环
- 焦点事件分发
- 焦点状态订阅

## ink/hooks/use-input.ts - 输入处理

> 来源: `/Users/sunxianfeng/Desktop/n-true/claude_code_book/reference/claude-code/src/ink/hooks/use-input.ts`

```typescript
const useInput = (inputHandler: Handler, options: Options = {}) => {
  const { setRawMode, internal_exitOnCtrlC, internal_eventEmitter } = useStdin();
  
  useLayoutEffect(() => {
    if (options.isActive === false) return;
    setRawMode(true);  // 同步启用 raw 模式
    return () => setRawMode(false);
  }, [...]);
  
  // 注册输入事件监听器
  useEffect(() => {
    internal_eventEmitter?.on('input', handleData);
    return () => { ... };
  }, [...]);
};
```

## ink/termio/ - 终端协议

### osc.ts - OSC (Operating System Commands)

> 来源: `/Users/sunxianfeng/Desktop/n-true/claude_code_book/reference/claude-code/src/ink/termio/osc.ts`

处理终端操作系统的命令，如：
- `setClipboard()`: OSC 52 剪贴板操作
- `supportsTabStatus()`: OSC 21337 标签状态
- `wrapForMultiplexer()`: tmux 包装

### sgr.ts - SGR (Select Graphic Rendition)

> 来源: `/Users/sunxianfeng/Desktop/n-true/claude_code_book/reference/claude-code/src/ink/termio/sgr.ts`

处理终端文本样式（颜色、粗体、下划线等）。

### csi.ts - CSI (Control Sequence Introducer)

> 来源: `/Users/sunxianfeng/Desktop/n-true/claude_code_book/reference/claude-code/src/ink/termio/csi.ts`

处理光标移动、屏幕清除等控制序列。

## ink/layout/ - Yoga 布局

### yoga.ts

> 来源: `/Users/sunxianfeng/Desktop/n-true/claude_code_book/reference/claude-code/src/ink/layout/yoga.ts`

```typescript
export class YogaLayoutNode implements LayoutNode {
  readonly yoga: YogaNode;
  
  calculateLayout(width?: number, _height?: number): void {
    this.yoga.calculateLayout(width, undefined, Direction.LTR);
  }
  
  setMeasureFunc(fn: LayoutMeasureFunc): void {
    this.yoga.setMeasureFunc((w, wMode) => {
      // 转换为 Yoga MeasureMode 并调用测量函数
    });
  }
}
```

## ink/components/ - Ink 内置组件

| 组件 | 文件 | 用途 |
|------|------|------|
| Box | Box.tsx | Flexbox 布局容器 |
| Text | Text.tsx | 文本渲染 |
| Button | Button.tsx | 按钮组件 |
| Link | Link.tsx | 超链接 |
| Newline | Newline.tsx | 换行符 |
| Spacer | Spacer.tsx | 弹性空间 |
| RawAnsi | RawAnsi.tsx | 原始 ANSI 转义序列 |
| NoSelect | NoSelect.tsx | 不可选择文本 |
| AlternateScreen | AlternateScreen.tsx | Alt Screen 包装器 |
| ScrollBox | ScrollBox.tsx | 滚动容器 |
| App | App.tsx | Ink 根 App 组件 |
| StdinContext | StdinContext.ts | stdin Context |
| TerminalFocusContext | TerminalFocusContext.tsx | 焦点 Context |
| TerminalSizeContext | TerminalSizeContext.tsx | 终端尺寸 Context |

## 事件系统

### ink/events/

- `input-event.ts`: 输入事件 (按键、粘贴)
- `keyboard-event.ts`: 键盘事件
- `click-event.ts`: 点击事件
- `focus-event.ts`: 焦点事件
- `resize-event.ts`: 调整大小事件
