# Yoga 布局引擎应用原理深度解析

## 概述

Ink 使用 Facebook 的 Yoga 布局引擎来在终端环境中实现 Flexbox 布局。这是一个革命性的设计决策，让开发者能够使用熟悉的 CSS Flexbox 概念来构建终端界面。本文将深入探讨 Yoga 在 Ink 中的集成和应用原理。

## Yoga 布局引擎简介

### 什么是 Yoga？

Yoga 是 Facebook 开发的跨平台布局引擎，主要特点：
- 实现了完整的 Flexbox 规范
- 纯 C++ 实现，提供多语言绑定
- 高性能，适合移动设备
- 独立于渲染层，可适配任何渲染目标

### 为什么选择 Yoga？

1. **标准化**：Flexbox 是 Web 开发的标准布局方案
2. **跨平台**：同样的布局代码可以在不同平台运行
3. **性能**：高效的布局算法，适合频繁更新
4. **灵活性**：支持复杂的布局需求

## Yoga 节点的创建与管理

### 1. 节点创建

在 `src/dom.ts` 中，每个 DOM 元素都关联一个 Yoga 节点：

```typescript
export const createNode = (nodeName: ElementNames): DOMElement => {
  const node: DOMElement = {
    nodeName,
    style: {},
    attributes: {},
    childNodes: [],
    parentNode: undefined,
    yogaNode: nodeName === 'ink-virtual-text' ? undefined : Yoga.Node.create()
  };

  // 设置默认的 Flexbox 属性
  if (node.yogaNode) {
    node.yogaNode.setFlexDirection(Yoga.FLEX_DIRECTION_ROW);
    node.yogaNode.setFlexWrap(Yoga.WRAP_NO_WRAP);
  }

  return node;
};
```

### 2. 样式应用

在 `src/styles.ts` 中，Ink 将样式属性映射到 Yoga API：

```typescript
const applyStyles = (yogaNode: YogaNode, style: Styles) => {
  // 尺寸属性
  if (style.width !== undefined) {
    if (typeof style.width === 'number') {
      yogaNode.setWidth(style.width);
    } else if (typeof style.width === 'string' && style.width.endsWith('%')) {
      yogaNode.setWidthPercent(parseInt(style.width));
    }
  }

  if (style.height !== undefined) {
    if (typeof style.height === 'number') {
      yogaNode.setHeight(style.height);
    } else if (typeof style.height === 'string' && style.height.endsWith('%')) {
      yogaNode.setHeightPercent(parseInt(style.height));
    }
  }

  // Flexbox 属性
  if (style.flexDirection !== undefined) {
    yogaNode.setFlexDirection(
      style.flexDirection === 'row' ? Yoga.FLEX_DIRECTION_ROW :
      style.flexDirection === 'column' ? Yoga.FLEX_DIRECTION_COLUMN :
      style.flexDirection === 'row-reverse' ? Yoga.FLEX_DIRECTION_ROW_REVERSE :
      Yoga.FLEX_DIRECTION_COLUMN_REVERSE
    );
  }

  if (style.flexGrow !== undefined) {
    yogaNode.setFlexGrow(style.flexGrow);
  }

  if (style.flexShrink !== undefined) {
    yogaNode.setFlexShrink(style.flexShrink);
  }

  if (style.flexBasis !== undefined) {
    if (typeof style.flexBasis === 'number') {
      yogaNode.setFlexBasis(style.flexBasis);
    } else if (style.flexBasis.endsWith('%')) {
      yogaNode.setFlexBasisPercent(parseInt(style.flexBasis));
    }
  }

  // 对齐属性
  if (style.alignItems !== undefined) {
    yogaNode.setAlignItems(
      style.alignItems === 'flex-start' ? Yoga.ALIGN_FLEX_START :
      style.alignItems === 'center' ? Yoga.ALIGN_CENTER :
      Yoga.ALIGN_FLEX_END
    );
  }

  if (style.justifyContent !== undefined) {
    yogaNode.setJustifyContent(
      style.justifyContent === 'flex-start' ? Yoga.JUSTIFY_FLEX_START :
      style.justifyContent === 'center' ? Yoga.JUSTIFY_CENTER :
      style.justifyContent === 'flex-end' ? Yoga.JUSTIFY_FLEX_END :
      style.justifyContent === 'space-between' ? Yoga.JUSTIFY_SPACE_BETWEEN :
      style.justifyContent === 'space-around' ? Yoga.JUSTIFY_SPACE_AROUND :
      Yoga.JUSTIFY_SPACE_EVENLY
    );
  }

  // 间距属性
  applySpacing(yogaNode, style);
};
```

### 3. 间距处理

```typescript
const applySpacing = (yogaNode: YogaNode, style: Styles) => {
  // Padding
  if (style.paddingTop !== undefined) {
    yogaNode.setPadding(Yoga.EDGE_TOP, style.paddingTop);
  }
  if (style.paddingBottom !== undefined) {
    yogaNode.setPadding(Yoga.EDGE_BOTTOM, style.paddingBottom);
  }
  if (style.paddingLeft !== undefined) {
    yogaNode.setPadding(Yoga.EDGE_LEFT, style.paddingLeft);
  }
  if (style.paddingRight !== undefined) {
    yogaNode.setPadding(Yoga.EDGE_RIGHT, style.paddingRight);
  }

  // 简写属性
  if (style.paddingX !== undefined) {
    yogaNode.setPadding(Yoga.EDGE_HORIZONTAL, style.paddingX);
  }
  if (style.paddingY !== undefined) {
    yogaNode.setPadding(Yoga.EDGE_VERTICAL, style.paddingY);
  }
  if (style.padding !== undefined) {
    yogaNode.setPadding(Yoga.EDGE_ALL, style.padding);
  }

  // Margin (类似处理)
  // ...

  // Gap (Flexbox gap)
  if (style.gap !== undefined) {
    yogaNode.setGap(Yoga.GUTTER_ALL, style.gap);
  }
  if (style.columnGap !== undefined) {
    yogaNode.setGap(Yoga.GUTTER_COLUMN, style.columnGap);
  }
  if (style.rowGap !== undefined) {
    yogaNode.setGap(Yoga.GUTTER_ROW, style.rowGap);
  }
};
```

## 布局计算流程

### 1. 触发布局计算

在 `src/ink.tsx` 中：

```typescript
calculateLayout = () => {
  // 获取终端宽度
  const terminalWidth = this.options.stdout.columns || 80;
  
  // 设置根节点宽度
  this.rootNode.yogaNode!.setWidth(terminalWidth);
  
  // 执行布局计算
  this.rootNode.yogaNode!.calculateLayout(
    undefined,  // 宽度约束
    undefined,  // 高度约束
    Yoga.DIRECTION_LTR  // 布局方向
  );
};
```

### 2. 文本测量

对于文本节点，需要提供测量函数：

```typescript
// 在 src/reconciler.ts 中
if (node.nodeName === 'ink-text') {
  node.yogaNode.setMeasureFunc((width, widthMode, height, heightMode) => {
    const text = squashTextNodes(node);
    const dimensions = measureText(text, width);
    
    return {
      width: dimensions.width,
      height: dimensions.height
    };
  });
}
```

文本测量的实现（`src/measure-text.ts`）：

```typescript
const measureText = (text: string, maxWidth?: number) => {
  if (!text || text.length === 0) {
    return { width: 0, height: 0 };
  }

  // 计算文本宽度（考虑 ANSI 转义序列）
  const lines = text.split('\n');
  let maxLineWidth = 0;
  
  for (const line of lines) {
    const lineWidth = stringWidth(line);  // 使用 string-width 库
    maxLineWidth = Math.max(maxLineWidth, lineWidth);
  }

  // 如果有宽度限制，需要考虑换行
  if (maxWidth && maxLineWidth > maxWidth) {
    const wrappedText = wrapText(text, maxWidth, 'wrap');
    const wrappedLines = wrappedText.split('\n');
    
    return {
      width: Math.min(maxLineWidth, maxWidth),
      height: wrappedLines.length
    };
  }

  return {
    width: maxLineWidth,
    height: lines.length
  };
};
```

### 3. 布局结果应用

布局计算完成后，每个节点都有了确定的位置和尺寸：

```typescript
// 获取计算后的布局信息
const x = yogaNode.getComputedLeft();
const y = yogaNode.getComputedTop();
const width = yogaNode.getComputedWidth();
const height = yogaNode.getComputedHeight();

// 获取边框和内边距
const borderLeft = yogaNode.getComputedBorder(Yoga.EDGE_LEFT);
const paddingTop = yogaNode.getComputedPadding(Yoga.EDGE_TOP);
```

## 终端适配策略

### 1. 宽度限制处理

终端有固定的列数限制：

```typescript
// 监听终端大小变化
constructor(options: Options) {
  if (!isInCi) {
    options.stdout.on('resize', this.resized);
  }
}

resized = () => {
  this.calculateLayout();  // 重新计算布局
  this.onRender();        // 重新渲染
};
```

### 2. 坐标系统转换

Yoga 使用的是标准的 2D 坐标系统，需要转换为终端的行列系统：

```typescript
// 在 src/render-node-to-output.ts 中
const renderNodeToOutput = (node: DOMElement, output: Output, options) => {
  const { offsetX = 0, offsetY = 0 } = options;
  const { yogaNode } = node;
  
  if (yogaNode) {
    // Yoga 坐标是相对于父节点的
    const x = offsetX + yogaNode.getComputedLeft();
    const y = offsetY + yogaNode.getComputedTop();
    
    // 渲染内容到指定位置
    output.write(x, y, content);
    
    // 递归渲染子节点
    for (const child of node.childNodes) {
      renderNodeToOutput(child, output, {
        offsetX: x,
        offsetY: y,
        ...options
      });
    }
  }
};
```

## 高级布局特性

### 1. 百分比支持

```typescript
// 宽度百分比
<Box width="50%">...</Box>

// 实现
if (style.width.endsWith('%')) {
  const percent = parseInt(style.width);
  yogaNode.setWidthPercent(percent);
}
```

### 2. 最小/最大尺寸

```typescript
if (style.minWidth !== undefined) {
  yogaNode.setMinWidth(style.minWidth);
}
if (style.maxWidth !== undefined) {
  yogaNode.setMaxWidth(style.maxWidth);
}
```

### 3. Aspect Ratio（宽高比）

```typescript
if (style.aspectRatio !== undefined) {
  yogaNode.setAspectRatio(style.aspectRatio);
}
```

### 4. 边框处理

边框占用空间，Yoga 需要考虑：

```typescript
// 设置边框宽度
if (style.borderStyle) {
  yogaNode.setBorder(Yoga.EDGE_ALL, 1);  // 终端边框通常是 1 个字符宽
}

// 渲染时考虑边框
const contentX = x + yogaNode.getComputedBorder(Yoga.EDGE_LEFT);
const contentY = y + yogaNode.getComputedBorder(Yoga.EDGE_TOP);
```

## 性能优化

### 1. 布局缓存

Yoga 内部实现了布局缓存，只有在必要时才重新计算：

```typescript
// 标记需要重新布局
yogaNode.markDirty();

// 检查是否需要重新布局
if (yogaNode.isDirty()) {
  calculateLayout();
}
```

### 2. 批量更新

```typescript
// 批量应用样式变更
const applyStyleUpdates = (yogaNode: YogaNode, updates: Partial<Styles>) => {
  // 暂停布局计算
  yogaNode.setMeasureFunc(null);
  
  // 应用所有更新
  for (const [key, value] of Object.entries(updates)) {
    applyStyleProp(yogaNode, key, value);
  }
  
  // 恢复测量函数
  if (needsMeasurement) {
    yogaNode.setMeasureFunc(measureFunc);
  }
  
  // 标记需要重新布局
  yogaNode.markDirty();
};
```

### 3. 节点池化

对于频繁创建销毁的节点，可以使用对象池：

```typescript
class YogaNodePool {
  private pool: YogaNode[] = [];
  
  acquire(): YogaNode {
    return this.pool.pop() || Yoga.Node.create();
  }
  
  release(node: YogaNode) {
    node.reset();  // 重置所有属性
    this.pool.push(node);
  }
}
```

## 调试工具

### 1. 布局可视化

```typescript
const debugLayout = (node: DOMElement, depth = 0) => {
  const indent = '  '.repeat(depth);
  const { yogaNode } = node;
  
  if (yogaNode) {
    console.log(`${indent}${node.nodeName}: {`);
    console.log(`${indent}  position: (${yogaNode.getComputedLeft()}, ${yogaNode.getComputedTop()})`);
    console.log(`${indent}  size: ${yogaNode.getComputedWidth()}x${yogaNode.getComputedHeight()}`);
    console.log(`${indent}}`);
  }
  
  for (const child of node.childNodes) {
    debugLayout(child, depth + 1);
  }
};
```

### 2. 性能分析

```typescript
const profileLayout = () => {
  const start = performance.now();
  
  calculateLayout();
  
  const duration = performance.now() - start;
  console.log(`Layout calculation took ${duration}ms`);
  
  // 分析节点数量
  const nodeCount = countNodes(rootNode);
  console.log(`Total nodes: ${nodeCount}`);
};
```

## 常见布局模式

### 1. 居中布局

```jsx
<Box
  height={10}
  alignItems="center"
  justifyContent="center"
>
  <Text>Centered Content</Text>
</Box>
```

### 2. 响应式布局

```jsx
<Box flexDirection="row" flexWrap="wrap">
  <Box flexBasis="50%" minWidth={20}>
    <Text>Column 1</Text>
  </Box>
  <Box flexBasis="50%" minWidth={20}>
    <Text>Column 2</Text>
  </Box>
</Box>
```

### 3. 固定头部/底部

```jsx
<Box flexDirection="column" height="100%">
  <Box height={3}>
    <Text>Header</Text>
  </Box>
  <Box flexGrow={1}>
    <Text>Content</Text>
  </Box>
  <Box height={3}>
    <Text>Footer</Text>
  </Box>
</Box>
```

## 总结

Yoga 布局引擎的集成是 Ink 成功的关键因素之一。通过巧妙地将 Flexbox 概念映射到终端环境，Ink 实现了：

1. **统一的布局模型**：Web 开发者可以直接应用已有知识
2. **强大的布局能力**：支持复杂的响应式布局
3. **高性能**：利用 Yoga 的优化算法
4. **跨平台一致性**：相同的布局代码在不同终端表现一致

这种设计让终端 UI 开发从传统的字符定位模式升级到了现代的盒模型布局，极大地提升了开发效率和用户体验。