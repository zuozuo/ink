# React Reconciler 自定义实现原理深度解析

## 概述

Ink 的核心创新在于将 React 的声明式编程模型带入终端环境。这个过程的关键是实现了一个自定义的 React Reconciler，它能够将 React 组件树转换为终端可以理解的输出。本文将深入分析这个实现的底层原理。

## React Reconciler 基础概念

### 什么是 React Reconciler？

React Reconciler 是 React 的核心算法实现，负责：
- 管理组件树的创建、更新和删除
- 执行 diff 算法，计算最小更新集
- 调度更新任务
- 与具体的渲染平台（Host）进行交互

React 的设计允许通过提供不同的 Host Config 来支持不同的渲染目标，如：
- React DOM → 浏览器 DOM
- React Native → 原生移动组件
- **Ink → 终端输出**

### 什么是 Fiber？

Fiber 是 React 16+ 中引入的核心概念，它是 React 内部的工作单元，代表了组件树中的一个节点。理解 Fiber 对于理解 React Reconciler 的工作原理至关重要。

#### Fiber 的核心概念

1. **Fiber 是一个 JavaScript 对象**，包含了组件的信息和工作进度：

```typescript
interface Fiber {
  // 组件类型信息
  type: any;                    // 函数组件、类组件或原生元素类型
  key: null | string;           // React key
  elementType: any;             // 元素类型
  
  // 组件实例和 props
  stateNode: any;               // 对应的真实 DOM 节点或组件实例
  pendingProps: any;            // 新的 props
  memoizedProps: any;           // 上一次渲染的 props
  memoizedState: any;           // 上一次渲染的 state
  
  // Fiber 树结构
  return: Fiber | null;         // 父 Fiber
  child: Fiber | null;          // 第一个子 Fiber
  sibling: Fiber | null;        // 下一个兄弟 Fiber
  index: number;                // 在兄弟中的位置
  
  // 副作用
  flags: Flags;                 // 副作用标记（更新、删除、新增等）
  subtreeFlags: Flags;          // 子树的副作用标记
  deletions: Fiber[] | null;    // 需要删除的子 Fiber
  
  // 调度优先级
  lanes: Lanes;                 // 优先级
  childLanes: Lanes;            // 子树的优先级
  
  // 双缓冲
  alternate: Fiber | null;      // 指向另一个版本的 Fiber（current ↔ workInProgress）
}
```

2. **Fiber 的双缓冲机制**：React 维护两棵 Fiber 树
   - **Current Fiber Tree**：当前显示在屏幕上的组件树
   - **Work-in-Progress Fiber Tree**：正在内存中构建的新树

```typescript
// 示例：更新时的双缓冲
function beginWork(current: Fiber | null, workInProgress: Fiber) {
  // current 是当前屏幕上的 Fiber
  // workInProgress 是正在构建的新 Fiber
  
  if (current !== null) {
    // 更新现有组件
    if (current.memoizedProps !== workInProgress.pendingProps) {
      // props 发生变化，需要更新
    }
  } else {
    // 挂载新组件
  }
}
```

3. **Fiber 如何实现可中断渲染**：

```typescript
// 传统的递归渲染（不可中断）
function renderRecursive(element) {
  if (element.children) {
    element.children.forEach(child => {
      renderRecursive(child);  // 递归调用，无法中断
    });
  }
}

// Fiber 的链表遍历（可中断）
function performUnitOfWork(fiber: Fiber) {
  // 处理当前 Fiber
  beginWork(fiber);
  
  // 返回下一个要处理的 Fiber
  if (fiber.child) {
    return fiber.child;
  }
  
  while (fiber) {
    completeWork(fiber);
    
    if (fiber.sibling) {
      return fiber.sibling;
    }
    
    fiber = fiber.return;  // 回到父节点
  }
  
  return null;
}

// 工作循环（可以在任何时候中断）
function workLoop(deadline) {
  while (nextUnitOfWork && deadline.timeRemaining() > 0) {
    nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
  }
  
  if (nextUnitOfWork) {
    // 时间用完了，请求下一次调度
    requestIdleCallback(workLoop);
  }
}
```

#### Fiber 在 Ink 中的应用

在 Ink 的上下文中，Fiber 工作流程如下：

1. **创建 Fiber 节点**：当 React 处理 `<Text color="green">Hello</Text>` 时

```typescript
// React 为 Text 组件创建 Fiber
const textComponentFiber = {
  type: Text,                           // Text 函数组件
  pendingProps: { color: "green" },     // 新 props
  stateNode: null,                      // 函数组件没有实例
  return: parentFiber,                  // 父 Fiber
  child: null,                          // 将在 beginWork 中创建
  flags: NoFlags,                       // 初始无副作用
};

// 执行 Text 组件，返回 <ink-text>
const inkTextElement = Text({ color: "green", children: "Hello" });

// 为 ink-text 创建 Fiber
const inkTextFiber = {
  type: "ink-text",                     // Host 组件类型
  pendingProps: {
    style: { flexGrow: 0, ... },
    internal_transform: colorizeFunction,
  },
  stateNode: null,                      // 将在 createInstance 中创建
  return: textComponentFiber,           // 父 Fiber
  flags: Placement,                     // 需要插入 DOM
};
```

2. **Fiber 与 Host 实例的关联**：

```typescript
// 在 createInstance 中创建 DOM 节点并关联到 Fiber
createInstance(type, props, rootContainer, hostContext) {
  const domElement = createNode(type);
  
  // ... 设置属性
  
  // 这个 DOM 元素会被保存到 Fiber.stateNode
  return domElement;
}

// 关联后
inkTextFiber.stateNode = {
  nodeName: 'ink-text',
  yogaNode: YogaNode,
  childNodes: [],
  internal_transform: colorizeFunction,
};
```

3. **Fiber 树的遍历和更新**：

```typescript
// 当状态更新时，React 会：
// 1. 克隆 Fiber 节点创建 workInProgress 树
// 2. 在 workInProgress 树上执行更新
// 3. 比较新旧 props 决定是否需要更新 Host 实例

function updateHostComponent(current, workInProgress) {
  const oldProps = current.memoizedProps;
  const newProps = workInProgress.pendingProps;
  
  if (oldProps !== newProps) {
    // 标记需要更新
    workInProgress.flags |= Update;
    
    // 准备更新数据
    const updatePayload = prepareUpdate(
      workInProgress.stateNode,
      type,
      oldProps,
      newProps
    );
    
    workInProgress.updateQueue = updatePayload;
  }
}
```

4. **副作用的收集和执行**：

```typescript
// Fiber 使用标记来追踪需要执行的操作
const Placement = 0b000001;      // 插入
const Update = 0b000010;         // 更新  
const Deletion = 0b000100;       // 删除

// 在 commit 阶段，根据标记执行实际操作
function commitWork(fiber) {
  if (fiber.flags & Placement) {
    // 调用 appendChild 插入节点
    appendChildToContainer(fiber.stateNode);
  }
  
  if (fiber.flags & Update) {
    // 调用 commitUpdate 更新属性
    commitUpdate(fiber.stateNode, fiber.updateQueue);
  }
  
  if (fiber.flags & Deletion) {
    // 调用 removeChild 删除节点
    removeChildFromContainer(fiber.stateNode);
  }
}
```

#### Fiber 的优势

1. **可中断和恢复**：渲染工作可以被分割成小块，允许浏览器处理高优先级任务
2. **优先级调度**：不同的更新可以有不同的优先级
3. **双缓冲**：在内存中准备新树，减少 UI 闪烁
4. **增量渲染**：可以逐步完成渲染，而不是一次性完成

在 Ink 的场景中，虽然终端渲染通常很快，但 Fiber 架构仍然带来了：
- 统一的 React 开发体验
- 支持 Suspense、Concurrent Features 等高级特性
- 更好的错误边界处理
- 未来的性能优化空间

## Ink 的 Host Config 实现

### 1. 节点类型定义

Ink 定义了自己的节点系统，位于 `src/dom.ts`：

```typescript
// 支持的元素类型
type ElementNames = 
  | 'ink-root'      // 根节点
  | 'ink-box'       // 容器节点（类似 div）
  | 'ink-text'      // 文本容器节点
  | 'ink-virtual-text'; // 虚拟文本节点

// DOM 元素结构
interface DOMElement {
  nodeName: ElementNames;
  attributes: Record<string, DOMNodeAttribute>;
  childNodes: DOMNode[];
  parentNode?: DOMElement;
  yogaNode?: YogaNode;  // 布局信息
  style: Styles;        // 样式信息
  
  // 渲染相关回调
  onComputeLayout?: () => void;
  onRender?: () => void;
  onImmediateRender?: () => void;
}
```

### 2. Reconciler 配置详解

在 `src/reconciler.ts` 中，Ink 实现了完整的 Host Config：

```typescript
export default createReconciler<...>({
  // 1. 创建实例
  createInstance(type, props, rootContainer, hostContext) {
    const node = createNode(type);
    
    // 应用样式
    if (type === 'ink-box') {
      applyStyles(node.yogaNode!, props);
    }
    
    // 设置属性
    for (const [key, value] of Object.entries(props)) {
      if (key !== 'children' && key !== 'style') {
        setAttribute(node, key, value);
      }
    }
    
    return node;
  },

  // 2. 创建文本实例
  createTextInstance(text, rootContainer, hostContext) {
    // 根据上下文决定创建哪种文本节点
    const node = hostContext.isInsideText 
      ? createTextNode(text)
      : createNode('ink-virtual-text');
    
    if (!hostContext.isInsideText) {
      setTextNodeValue(node, text);
    }
    
    return node;
  },

  // 3. 准备更新
  prepareUpdate(instance, type, oldProps, newProps) {
    const updatePayload: any = {};
    let needsUpdate = false;
    
    // 计算样式差异
    if (type === 'ink-box') {
      const diff = diffStyles(oldProps, newProps);
      if (diff) {
        updatePayload.style = diff;
        needsUpdate = true;
      }
    }
    
    // 计算属性差异
    const attrDiff = diffAttributes(oldProps, newProps);
    if (attrDiff) {
      updatePayload.attributes = attrDiff;
      needsUpdate = true;
    }
    
    return needsUpdate ? updatePayload : null;
  },

  // 4. 提交更新
  commitUpdate(instance, updatePayload) {
    if (updatePayload.style) {
      applyStyles(instance.yogaNode!, updatePayload.style);
    }
    
    if (updatePayload.attributes) {
      for (const [key, value] of Object.entries(updatePayload.attributes)) {
        setAttribute(instance, key, value);
      }
    }
  }
});
```

### 3. 上下文管理

Ink 使用 Host Context 来跟踪渲染上下文：

```typescript
type HostContext = {
  isInsideText: boolean;  // 是否在文本组件内部
};

// 获取根上下文
getRootHostContext: () => ({
  isInsideText: false,
}),

// 获取子上下文
getChildHostContext(parentHostContext, type) {
  const isInsideText = type === 'ink-text' || type === 'ink-virtual-text';
  
  if (parentHostContext.isInsideText === isInsideText) {
    return parentHostContext;
  }
  
  return { isInsideText };
}
```

## 渲染流程详解

### 1. 组件挂载流程：从 JSX 到终端输出

让我们通过一个具体例子 `<Text color="green">Hello</Text>` 来详细追踪整个渲染流程。

#### 步骤 1：JSX 转换和组件创建

```jsx
// 用户代码
<Text color="green">Hello</Text>

// Babel 转换后
React.createElement(Text, { color: "green" }, "Hello")

// Text 组件内部实现 (src/components/Text.tsx)
export default function Text({color, children, ...props}) {
  const transform = (text: string): string => {
    if (color) {
      return colorize(text, color, 'foreground');
    }
    return text;
  };
  
  return (
    <ink-text
      style={{flexGrow: 0, flexShrink: 1, flexDirection: 'row', textWrap: wrap}}
      internal_transform={transform}
    >
      {children}
    </ink-text>
  );
}
```

#### 步骤 2：Reconciler 处理组件树

React Reconciler 开始工作，创建 Fiber 节点：

```typescript
// 1. 处理 Text 组件，返回 ink-text 元素
// 2. Reconciler 看到 ink-text，调用 createInstance

createInstance(originalType, newProps, rootNode, hostContext) {
  // originalType = 'ink-text'
  // newProps = { 
  //   style: {flexGrow: 0, flexShrink: 1, flexDirection: 'row', textWrap: 'wrap'},
  //   internal_transform: [Function],
  //   children: 'Hello'
  // }
  // hostContext = { isInsideText: false }
  
  const type = originalType === 'ink-text' && hostContext.isInsideText
    ? 'ink-virtual-text'
    : originalType; // 结果是 'ink-text'
  
  // 创建 DOM 节点
  const node = createNode(type);
  // node = {
  //   nodeName: 'ink-text',
  //   style: {},
  //   attributes: {},
  //   childNodes: [],
  //   yogaNode: Yoga.Node.create()
  // }
  
  // 处理 props
  for (const [key, value] of Object.entries(newProps)) {
    if (key === 'style') {
      setStyle(node, value);
      applyStyles(node.yogaNode, value); // 应用 Flexbox 样式
    }
    
    if (key === 'internal_transform') {
      node.internal_transform = value; // 保存文本转换函数
    }
  }
  
  return node;
}
```

#### 步骤 3：处理文本内容

Reconciler 发现 "Hello" 是文本内容，需要创建文本节点：

```typescript
// 更新 hostContext
getChildHostContext(parentHostContext, type) {
  // type = 'ink-text'
  const isInsideText = type === 'ink-text' || type === 'ink-virtual-text';
  // isInsideText = true
  
  return { isInsideText: true };
}

// 创建文本节点
createTextInstance(text, _root, hostContext) {
  // text = "Hello"
  // hostContext = { isInsideText: true }
  
  if (!hostContext.isInsideText) {
    throw new Error(`Text string "${text}" must be rendered inside <Text> component`);
  }
  
  return createTextNode(text);
  // 返回 { nodeName: '#text', nodeValue: 'Hello' }
}
```

#### 步骤 4：构建节点树

```typescript
// 将文本节点添加到 ink-text 节点
appendInitialChild(parentInstance, child) {
  // parentInstance = ink-text 节点
  // child = 文本节点
  
  appendChildNode(parentInstance, child);
  // ink-text.childNodes = [{ nodeName: '#text', nodeValue: 'Hello' }]
}
```

#### 步骤 5：布局计算

当组件树构建完成后，触发布局计算：

```typescript
// src/ink.tsx - calculateLayout
calculateLayout = () => {
  const terminalWidth = this.options.stdout.columns || 80;
  
  // 设置根节点宽度
  this.rootNode.yogaNode!.setWidth(terminalWidth);
  
  // Yoga 计算布局
  this.rootNode.yogaNode!.calculateLayout(
    undefined,
    undefined,
    Yoga.DIRECTION_LTR
  );
  
  // 此时每个节点都有了计算后的位置和尺寸
  // ink-text 节点: x=0, y=0, width=5, height=1
};
```

#### 步骤 6：渲染到 Output

```typescript
// src/render-node-to-output.ts
const renderNodeToOutput = (node: DOMElement, output: Output, options) => {
  const {yogaNode} = node;
  
  if (node.nodeName === 'ink-text') {
    // 获取所有文本内容
    let text = squashTextNodes(node); // "Hello"
    
    // 应用转换（颜色）
    if (node.internal_transform) {
      text = node.internal_transform(text);
      // 结果: "\x1B[32mHello\x1B[39m" (带绿色 ANSI 代码的文本)
    }
    
    // 获取位置
    const x = yogaNode.getComputedLeft(); // 0
    const y = yogaNode.getComputedTop();  // 0
    
    // 写入到输出缓冲区
    output.write(x, y, text);
  }
};
```

#### 步骤 7：生成最终输出

```typescript
// src/output.ts
class Output {
  private canvas: string[][] = [];
  
  write(x: number, y: number, text: string) {
    // 将文本写入虚拟画布的指定位置
    const lines = text.split('\n');
    
    for (let i = 0; i < lines.length; i++) {
      const line = lines[i];
      const row = y + i;
      
      if (!this.canvas[row]) {
        this.canvas[row] = [];
      }
      
      // 逐字符写入（处理 ANSI 序列）
      let column = x;
      for (const char of line) {
        this.canvas[row][column] = char;
        column++;
      }
    }
  }
  
  get() {
    // 将虚拟画布转换为字符串
    const lines = this.canvas.map(row => row.join(''));
    return {
      output: lines.join('\n'), // "\x1B[32mHello\x1B[39m"
      height: lines.length       // 1
    };
  }
}
```

#### 步骤 8：输出到终端

```typescript
// src/ink.tsx - onRender
onRender = () => {
  const {output, outputHeight} = render(this.rootNode);
  
  if (output !== this.lastOutput) {
    // 使用 log-update 更新终端
    this.throttledLog(output);
    // 终端显示绿色的 "Hello"
    
    this.lastOutput = output;
    this.lastOutputHeight = outputHeight;
  }
};
```

#### 完整流程图

```
<Text color="green">Hello</Text>
         ↓
React.createElement(Text, {color: "green"}, "Hello")
         ↓
Text 组件返回 <ink-text internal_transform={colorize}>Hello</ink-text>
         ↓
Reconciler.createInstance('ink-text', props)
    - 创建 DOMElement { nodeName: 'ink-text', yogaNode: ... }
    - 保存 internal_transform 函数
         ↓
Reconciler.createTextInstance('Hello')
    - 创建 TextNode { nodeName: '#text', nodeValue: 'Hello' }
         ↓
appendChild(ink-text, textNode)
    - 构建父子关系
         ↓
Yoga 布局计算
    - 计算位置 (x=0, y=0) 和尺寸 (width=5, height=1)
         ↓
renderNodeToOutput()
    - 提取文本: "Hello"
    - 应用转换: "\x1B[32mHello\x1B[39m"
    - 写入 Output 对象的 (0, 0) 位置
         ↓
Output.get()
    - 生成最终字符串: "\x1B[32mHello\x1B[39m"
         ↓
log-update(output)
    - 输出到终端
    - 用户看到绿色的 "Hello"
```

### 2. 更新流程

```typescript
// 在 resetAfterCommit 中触发渲染
resetAfterCommit(rootNode) {
  // 1. 计算布局
  if (typeof rootNode.onComputeLayout === 'function') {
    rootNode.onComputeLayout();
  }
  
  // 2. 处理静态内容的立即渲染
  if (rootNode.isStaticDirty) {
    rootNode.isStaticDirty = false;
    if (typeof rootNode.onImmediateRender === 'function') {
      rootNode.onImmediateRender();
    }
    return;
  }
  
  // 3. 常规渲染
  if (typeof rootNode.onRender === 'function') {
    rootNode.onRender();
  }
}
```

### 3. 优先级调度

Ink 实现了 React 的优先级调度机制：

```typescript
let currentUpdatePriority = NoEventPriority;

getCurrentEventPriority() {
  return currentUpdatePriority;
},

setCurrentUpdatePriority(newPriority) {
  currentUpdatePriority = newPriority;
},

// 不同类型的事件有不同优先级
resolveUpdatePriority() {
  if (currentUpdatePriority !== NoEventPriority) {
    return currentUpdatePriority;
  }
  return DefaultEventPriority;
}
```

## 与终端的集成

### 1. 根节点创建

在 `src/ink.tsx` 中：

```typescript
constructor(options: Options) {
  // 创建根 DOM 节点
  this.rootNode = dom.createNode('ink-root');
  
  // 设置布局计算回调
  this.rootNode.onComputeLayout = this.calculateLayout;
  
  // 设置渲染回调（带节流）
  this.rootNode.onRender = options.debug
    ? this.onRender
    : throttle(this.onRender, 32, {
        leading: true,
        trailing: true,
      });
  
  // 创建 Reconciler 容器
  this.container = reconciler.createContainer(
    this.rootNode,
    LegacyRoot,
    null,
    false,
    null,
    'id',
    () => {},
    () => {}
  );
}
```

### 2. 渲染触发机制

```typescript
render(node: ReactNode) {
  reconciler.updateContainer(
    <App {...props}>{node}</App>,
    this.container,
    null,
    () => {}
  );
}
```

## 性能优化策略

### 1. 批量更新

Reconciler 自动批处理多个状态更新，减少实际的渲染次数。

### 2. 静态内容优化

对于 `<Static>` 组件，使用特殊标记避免重复渲染：

```typescript
if (skipStaticElements && node.internal_static) {
  return;  // 跳过静态元素的渲染
}
```

### 3. 差异计算优化

```typescript
const diff = (before: AnyObject, after: AnyObject): AnyObject | undefined => {
  if (before === after) {
    return;  // 引用相等，无需更新
  }
  
  // 只计算实际变化的属性
  const changed: AnyObject = {};
  let isChanged = false;
  
  // ... 差异计算逻辑
  
  return isChanged ? changed : undefined;
};
```

## React Reconciliation 算法中的 Diff 算法深度解析

### Diff 算法的核心原理

React 的 Reconciliation（协调）算法是 React 用来确定如何高效更新 UI 以匹配最新状态的过程。其核心是通过 diff 算法比较两棵虚拟 DOM 树的差异，并将这些差异应用到实际 DOM 上。

#### 传统 Diff 算法的问题

传统的树 diff 算法时间复杂度为 O(n³)，对于大型应用来说性能开销过大。React 通过三个关键假设将复杂度降低到 O(n)：

1. **不同类型的元素会产生不同的树**
2. **可以通过 key 属性来标识哪些子元素在不同的渲染中保持稳定**
3. **只对同一层级的节点进行比较**

### Ink 中的 Diff 算法实现

#### 1. 属性 Diff 实现 (reconciler.ts:53-84)

Ink 实现了一个简洁高效的属性 diff 函数：

```typescript
const diff = (before: AnyObject, after: AnyObject): AnyObject | undefined => {
    // 快速路径：引用相等检查
    if (before === after) {
        return;
    }
    
    // 边界情况：before 不存在
    if (!before) {
        return after;
    }
    
    const changed: AnyObject = {};
    let isChanged = false;
    
    // 阶段1：检测被删除的属性
    for (const key of Object.keys(before)) {
        const isDeleted = after ? !Object.hasOwn(after, key) : true;
        
        if (isDeleted) {
            changed[key] = undefined;  // undefined 表示删除
            isChanged = true;
        }
    }
    
    // 阶段2：检测新增或修改的属性
    if (after) {
        for (const key of Object.keys(after)) {
            if (after[key] !== before[key]) {
                changed[key] = after[key];
                isChanged = true;
            }
        }
    }
    
    return isChanged ? changed : undefined;
};
```

这个 diff 函数的特点：
- **高效**：使用引用相等检查快速跳过无变化的对象
- **完整**：同时处理属性的新增、修改和删除
- **精确**：只返回实际变化的属性，减少后续处理

#### 2. 节点树的 Diff 操作

Ink 通过实现 React Reconciler 的标准接口来处理节点树的变化：

##### 2.1 节点创建 (createInstance)
当 React 发现需要创建新节点时：
```typescript
createInstance(originalType, newProps, rootNode, hostContext) {
    const node = createNode(type);
    
    // 应用所有属性
    for (const [key, value] of Object.entries(newProps)) {
        if (key === 'style') {
            setStyle(node, value as Styles);
            applyStyles(node.yogaNode, value as Styles);
        } else if (key === 'internal_transform') {
            node.internal_transform = value as OutputTransformer;
        } else {
            setAttribute(node, key, value as DOMNodeAttribute);
        }
    }
    
    return node;
}
```

##### 2.2 节点更新 (commitUpdate)
当属性发生变化时：
```typescript
commitUpdate(node, _type, oldProps, newProps) {
    // 使用 diff 函数计算变化
    const props = diff(oldProps, newProps);
    const style = diff(
        oldProps['style'] as Styles,
        newProps['style'] as Styles,
    );
    
    // 只有存在变化时才更新
    if (!props && !style) {
        return;
    }
    
    // 应用属性变化
    if (props) {
        for (const [key, value] of Object.entries(props)) {
            if (key === 'style') {
                setStyle(node, value as Styles);
            } else {
                setAttribute(node, key, value as DOMNodeAttribute);
            }
        }
    }
    
    // 应用样式变化
    if (style && node.yogaNode) {
        applyStyles(node.yogaNode, style);
    }
}
```

##### 2.3 子节点操作
Ink 实现了完整的子节点操作接口：

```typescript
// 添加子节点
appendChildNode(node: DOMElement, childNode: DOMElement): void {
    if (childNode.parentNode) {
        removeChildNode(childNode.parentNode, childNode);
    }
    
    childNode.parentNode = node;
    node.childNodes.push(childNode);
    
    // 同步 Yoga 布局树
    if (childNode.yogaNode) {
        node.yogaNode?.insertChild(
            childNode.yogaNode,
            node.yogaNode.getChildCount(),
        );
    }
}

// 插入子节点
insertBeforeNode(
    node: DOMElement,
    newChildNode: DOMNode,
    beforeChildNode: DOMNode,
): void {
    const index = node.childNodes.indexOf(beforeChildNode);
    if (index >= 0) {
        node.childNodes.splice(index, 0, newChildNode);
        if (newChildNode.yogaNode) {
            node.yogaNode?.insertChild(newChildNode.yogaNode, index);
        }
    }
}

// 移除子节点
removeChildNode(node: DOMElement, removeNode: DOMNode): void {
    if (removeNode.yogaNode) {
        removeNode.parentNode?.yogaNode?.removeChild(removeNode.yogaNode);
    }
    
    removeNode.parentNode = undefined;
    
    const index = node.childNodes.indexOf(removeNode);
    if (index >= 0) {
        node.childNodes.splice(index, 1);
    }
}
```

#### 3. Key 的处理和列表 Diff

从测试用例可以看到 Ink 对 key 的支持：

```typescript
// 测试：使用 key 重排序子元素 (reconciler.tsx:248-304)
function Test({reorder}: {readonly reorder?: boolean}) {
    if (reorder) {
        return (
            <Box flexDirection="column">
                <Text key="b">B</Text>
                <Text key="a">A</Text>
            </Box>
        );
    }
    
    return (
        <Box flexDirection="column">
            <Text key="a">A</Text>
            <Text key="b">B</Text>
        </Box>
    );
}
```

当组件从第一种状态更新到第二种状态时：
1. React 通过 key 识别出两个 Text 组件是相同的，只是位置变了
2. 不会销毁和重建组件，而是调用 `insertBeforeNode` 重新排序
3. 保留了组件的内部状态和 DOM 节点

### React Reconciliation 的完整工作流程

#### 1. 更新触发
- 组件调用 `setState` 或接收新的 props
- React 将更新任务加入更新队列
- 调度器决定何时开始处理更新

#### 2. Render 阶段（可中断）

##### 2.1 构建 Work-in-Progress 树
```typescript
function beginWork(current: Fiber | null, workInProgress: Fiber) {
    if (current !== null) {
        // 更新现有组件
        const oldProps = current.memoizedProps;
        const newProps = workInProgress.pendingProps;
        
        if (oldProps !== newProps || hasLegacyContextChanged()) {
            // 需要更新
            didReceiveUpdate = true;
        } else {
            // 可以复用
            return bailoutOnAlreadyFinishedWork(current, workInProgress);
        }
    }
    
    // 根据组件类型处理
    switch (workInProgress.tag) {
        case FunctionComponent:
            return updateFunctionComponent(current, workInProgress);
        case ClassComponent:
            return updateClassComponent(current, workInProgress);
        case HostComponent:
            return updateHostComponent(current, workInProgress);
    }
}
```

##### 2.2 Diff 算法核心流程
```typescript
function reconcileChildren(current, workInProgress, nextChildren) {
    if (current === null) {
        // 挂载新组件
        workInProgress.child = mountChildFibers(workInProgress, null, nextChildren);
    } else {
        // 更新现有组件
        workInProgress.child = reconcileChildFibers(
            workInProgress,
            current.child,
            nextChildren
        );
    }
}

function reconcileChildFibers(returnFiber, currentFirstChild, newChild) {
    // 处理不同类型的子节点
    
    // 单个元素
    if (typeof newChild === 'object' && newChild !== null) {
        return placeSingleChild(
            reconcileSingleElement(returnFiber, currentFirstChild, newChild)
        );
    }
    
    // 文本节点
    if (typeof newChild === 'string' || typeof newChild === 'number') {
        return placeSingleChild(
            reconcileSingleTextNode(returnFiber, currentFirstChild, newChild)
        );
    }
    
    // 数组（列表）
    if (isArray(newChild)) {
        return reconcileChildrenArray(returnFiber, currentFirstChild, newChild);
    }
}
```

##### 2.3 列表 Diff 算法
```typescript
function reconcileChildrenArray(returnFiber, currentFirstChild, newChildren) {
    // 第一轮遍历：处理更新的节点
    let oldFiber = currentFirstChild;
    let newIdx = 0;
    
    for (; oldFiber !== null && newIdx < newChildren.length; newIdx++) {
        if (oldFiber.key !== newChildren[newIdx].key) {
            break;  // key 不匹配，跳出第一轮
        }
        
        // 更新节点
        const newFiber = updateElement(oldFiber, newChildren[newIdx]);
        lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);
        
        oldFiber = oldFiber.sibling;
    }
    
    // 第二轮遍历：处理剩余情况
    if (newIdx === newChildren.length) {
        // 新列表遍历完，删除剩余旧节点
        deleteRemainingChildren(returnFiber, oldFiber);
        return resultingFirstChild;
    }
    
    if (oldFiber === null) {
        // 旧列表遍历完，插入剩余新节点
        for (; newIdx < newChildren.length; newIdx++) {
            const newFiber = createChild(returnFiber, newChildren[newIdx]);
            lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);
        }
        return resultingFirstChild;
    }
    
    // 第三轮遍历：处理节点移动
    // 构建 key -> oldFiber 的映射
    const existingChildren = mapRemainingChildren(returnFiber, oldFiber);
    
    for (; newIdx < newChildren.length; newIdx++) {
        const newFiber = updateFromMap(existingChildren, newChildren[newIdx]);
        if (newFiber !== null) {
            existingChildren.delete(newFiber.key);
            lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);
        }
    }
    
    // 删除未使用的节点
    existingChildren.forEach(child => deleteChild(returnFiber, child));
}
```

#### 3. Commit 阶段（不可中断）

##### 3.1 Before Mutation 阶段
- 调用 `getSnapshotBeforeUpdate` 生命周期
- 保存当前 DOM 状态供后续使用

##### 3.2 Mutation 阶段
```typescript
function commitMutationEffects(fiber: Fiber) {
    switch (fiber.tag) {
        case HostComponent: {
            const instance = fiber.stateNode;
            
            if (fiber.flags & Placement) {
                // 插入新节点
                appendChildToContainer(instance);
            }
            
            if (fiber.flags & Update) {
                // 更新属性
                const updatePayload = fiber.updateQueue;
                commitUpdate(instance, updatePayload);
            }
            
            if (fiber.flags & Deletion) {
                // 删除节点
                removeChildFromContainer(instance);
            }
            
            break;
        }
    }
}
```

##### 3.3 Layout 阶段
- 调用 `componentDidMount`、`componentDidUpdate` 等生命周期
- 在 Ink 中，触发布局计算和渲染

### 性能优化策略

#### 1. 时间切片（Time Slicing）
React 通过 Fiber 架构实现了可中断的渲染：
```typescript
function workLoop(deadline) {
    while (nextUnitOfWork && deadline.timeRemaining() > 0) {
        nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
    }
    
    if (nextUnitOfWork) {
        // 还有工作未完成，请求下一次调度
        requestIdleCallback(workLoop);
    } else {
        // 渲染阶段完成，进入提交阶段
        commitRoot();
    }
}
```

#### 2. 批量更新
多个状态更新会被合并成一次渲染：
```typescript
this.setState({a: 1});
this.setState({b: 2});
this.setState({c: 3});
// 只会触发一次重新渲染
```

#### 3. Bailout 优化
当组件的 props 和 state 都没有变化时，React 会跳过该组件的渲染：
```typescript
if (oldProps === newProps && oldState === newState) {
    return bailoutOnAlreadyFinishedWork(current, workInProgress);
}
```

### 总结

React Reconciliation 算法通过巧妙的设计实现了高效的 UI 更新：

1. **Diff 算法优化**：通过三个假设将 O(n³) 降至 O(n)
2. **Fiber 架构**：实现可中断渲染和优先级调度
3. **双缓冲机制**：平滑的 UI 更新体验
4. **Key 的使用**：高效处理列表更新
5. **批量更新**：减少不必要的渲染

Ink 作为一个 React 自定义渲染器的实现，完美展示了这些概念如何应用到实际场景中，将 React 的声明式编程模型带入了终端环境。

## 内存管理

### 1. Yoga 节点清理

```typescript
const cleanupYogaNode = (node?: YogaNode): void => {
  node?.unsetMeasureFunc();
  node?.freeRecursive();  // 递归释放所有子节点
};

// 在组件卸载时调用
removeChild(parentInstance, child) {
  removeChildNode(parentInstance, child);
  cleanupYogaNode(child.yogaNode);
}
```

### 2. 实例管理

使用 WeakMap 管理多个 Ink 实例，避免内存泄漏：

```typescript
// src/instances.ts
const instances = new WeakMap<NodeJS.WriteStream, Ink>();
```

## 调试支持

### React DevTools 集成

```typescript
if (process.env['DEV'] === 'true') {
  reconciler.injectIntoDevTools({
    bundleType: 0,
    version: '16.13.1',
    rendererPackageName: 'ink',
  });
}
```

这允许开发者使用 React DevTools 调试 Ink 应用，查看组件树和状态。

## 总结

Ink 的 React Reconciler 实现展示了 React 架构的强大扩展性。通过实现自定义的 Host Config，Ink 成功地：

1. **抽象了终端渲染**：将终端输出抽象为类 DOM 的节点树
2. **复用了 React 生态**：开发者可以使用熟悉的 React 模式和工具
3. **优化了性能**：通过批处理、节流和静态内容优化等策略
4. **保持了灵活性**：支持样式、布局、交互等丰富功能

这种设计不仅让 CLI 开发变得更加直观，也为其他非传统渲染目标提供了很好的参考实现。