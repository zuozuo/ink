# 文本渲染和样式系统原理深度解析

## 概述

Ink 的文本渲染和样式系统是其核心功能之一，它将 Web 开发中熟悉的文本样式概念带入了终端环境。通过 ANSI 转义序列和智能的文本处理，Ink 能够在终端中实现丰富的视觉效果。本文将深入探讨这个系统的实现原理。

## ANSI 转义序列基础

### 1. 什么是 ANSI 转义序列？

ANSI 转义序列是控制终端显示的特殊字符序列：

```typescript
// 基本格式：ESC[参数m
const ESC = '\x1B';
const red = `${ESC}[31m`;      // 红色前景色
const bgBlue = `${ESC}[44m`;   // 蓝色背景色
const bold = `${ESC}[1m`;      // 粗体
const reset = `${ESC}[0m`;     // 重置所有样式

// 使用示例
console.log(`${red}${bold}Hello${reset} World`);
```

### 2. 常用 ANSI 代码

```typescript
const ANSI_CODES = {
  // 样式
  reset: [0, 0],
  bold: [1, 22],
  dim: [2, 22],
  italic: [3, 23],
  underline: [4, 24],
  strikethrough: [9, 29],
  inverse: [7, 27],
  
  // 前景色 (30-37)
  black: [30, 39],
  red: [31, 39],
  green: [32, 39],
  yellow: [33, 39],
  blue: [34, 39],
  magenta: [35, 39],
  cyan: [36, 39],
  white: [37, 39],
  
  // 背景色 (40-47)
  bgBlack: [40, 49],
  bgRed: [41, 49],
  bgGreen: [42, 49],
  bgYellow: [43, 49],
  bgBlue: [44, 49],
  bgMagenta: [45, 49],
  bgCyan: [46, 49],
  bgWhite: [47, 49],
  
  // 亮色 (90-97)
  gray: [90, 39],
  brightRed: [91, 39],
  brightGreen: [92, 39],
  brightYellow: [93, 39],
  brightBlue: [94, 39],
  brightMagenta: [95, 39],
  brightCyan: [96, 39],
  brightWhite: [97, 39]
};
```

## Chalk 集成

### 1. 颜色系统

Ink 使用 Chalk 库处理颜色，`src/colorize.ts` 实现了高级颜色支持：

```typescript
import chalk from 'chalk';

const colorize = (text: string, color: string, type: 'foreground' | 'background') => {
  // 处理标准颜色名
  if (type === 'foreground' && color in chalk) {
    return chalk[color](text);
  }
  
  if (type === 'background' && `bg${capitalize(color)}` in chalk) {
    return chalk[`bg${capitalize(color)}`](text);
  }
  
  // 处理 Hex 颜色
  if (color.startsWith('#')) {
    return type === 'foreground' 
      ? chalk.hex(color)(text)
      : chalk.bgHex(color)(text);
  }
  
  // 处理 RGB 颜色
  const rgbMatch = color.match(/rgb\((\d+),\s*(\d+),\s*(\d+)\)/);
  if (rgbMatch) {
    const [, r, g, b] = rgbMatch;
    return type === 'foreground'
      ? chalk.rgb(Number(r), Number(g), Number(b))(text)
      : chalk.bgRgb(Number(r), Number(g), Number(b))(text);
  }
  
  // 处理 HSL 颜色
  const hslMatch = color.match(/hsl\((\d+),\s*(\d+)%,\s*(\d+)%\)/);
  if (hslMatch) {
    const [, h, s, l] = hslMatch;
    return type === 'foreground'
      ? chalk.hsl(Number(h), Number(s), Number(l))(text)
      : chalk.bgHsl(Number(h), Number(s), Number(l))(text);
  }
  
  return text;
};
```

### 2. 24位真彩色支持

现代终端支持 24 位颜色：

```typescript
const supports24BitColor = () => {
  // 检查环境变量
  if (process.env.COLORTERM === 'truecolor') {
    return true;
  }
  
  // 检查终端类型
  const term = process.env.TERM || '';
  if (term.includes('256color') || term.includes('24bit')) {
    return true;
  }
  
  return false;
};

// 24 位颜色 ANSI 序列
const rgb24bit = (r: number, g: number, b: number, bg = false) => {
  const type = bg ? 48 : 38;
  return `\x1B[${type};2;${r};${g};${b}m`;
};
```

## Text 组件实现

### 1. 样式转换管道

`src/components/Text.tsx` 中的样式应用：

```typescript
const Text: FC<Props> = ({
  color,
  backgroundColor,
  dimColor,
  bold,
  italic,
  underline,
  strikethrough,
  inverse,
  wrap,
  children
}) => {
  const transform = (text: string): string => {
    // 应用样式的顺序很重要
    let styledText = text;
    
    // 1. 首先应用 dim（会影响颜色）
    if (dimColor) {
      styledText = chalk.dim(styledText);
    }
    
    // 2. 应用颜色
    if (color) {
      styledText = colorize(styledText, color, 'foreground');
    }
    
    if (backgroundColor) {
      styledText = colorize(styledText, backgroundColor, 'background');
    }
    
    // 3. 应用文本样式
    if (bold) {
      styledText = chalk.bold(styledText);
    }
    
    if (italic) {
      styledText = chalk.italic(styledText);
    }
    
    if (underline) {
      styledText = chalk.underline(styledText);
    }
    
    if (strikethrough) {
      styledText = chalk.strikethrough(styledText);
    }
    
    // 4. 最后应用 inverse（反转前景和背景）
    if (inverse) {
      styledText = chalk.inverse(styledText);
    }
    
    return styledText;
  };
  
  return (
    <ink-text
      style={{textWrap: wrap}}
      internal_transform={transform}
    >
      {children}
    </ink-text>
  );
};
```

### 2. 嵌套样式处理

处理嵌套的 Text 组件：

```typescript
// 样式会正确嵌套和组合
<Text color="blue">
  Blue text with <Text bold>bold part</Text> and
  <Text backgroundColor="yellow"> highlighted </Text> section
</Text>

// 实现原理：每个 Text 组件独立应用其变换
const renderNestedText = (node: DOMElement): string => {
  let result = '';
  
  for (const child of node.childNodes) {
    if (isTextNode(child)) {
      // 应用当前节点的变换
      result += node.internal_transform
        ? node.internal_transform(child.nodeValue)
        : child.nodeValue;
    } else if (child.nodeName === 'ink-text') {
      // 递归处理嵌套的 Text
      const childText = renderNestedText(child);
      result += node.internal_transform
        ? node.internal_transform(childText)
        : childText;
    }
  }
  
  return result;
};
```

## 文本换行和截断

### 1. 自动换行实现

`src/wrap-text.ts` 实现了智能的文本换行：

```typescript
const wrapText = (text: string, maxWidth: number, wrapType: TextWrap): string => {
  if (wrapType === 'wrap') {
    return wrapWords(text, maxWidth);
  }
  
  // 截断模式
  const lines = text.split('\n');
  return lines.map(line => truncateLine(line, maxWidth, wrapType)).join('\n');
};

const wrapWords = (text: string, maxWidth: number): string => {
  const words = text.split(' ');
  const lines: string[] = [];
  let currentLine = '';
  
  for (const word of words) {
    // 计算实际宽度（排除 ANSI 序列）
    const wordWidth = stringWidth(word);
    const lineWidth = stringWidth(currentLine);
    
    // 单词太长，需要断开
    if (wordWidth > maxWidth) {
      if (currentLine) {
        lines.push(currentLine);
        currentLine = '';
      }
      
      // 按字符断开长单词
      const brokenWord = breakLongWord(word, maxWidth);
      lines.push(...brokenWord.slice(0, -1));
      currentLine = brokenWord[brokenWord.length - 1];
    }
    // 加上这个词会超出宽度
    else if (lineWidth + (currentLine ? 1 : 0) + wordWidth > maxWidth) {
      lines.push(currentLine);
      currentLine = word;
    }
    // 正常添加词
    else {
      currentLine += (currentLine ? ' ' : '') + word;
    }
  }
  
  if (currentLine) {
    lines.push(currentLine);
  }
  
  return lines.join('\n');
};
```

### 2. 文本截断

```typescript
const truncateLine = (line: string, maxWidth: number, wrapType: TextWrap): string => {
  const width = stringWidth(line);
  
  if (width <= maxWidth) {
    return line;
  }
  
  switch (wrapType) {
    case 'truncate':
    case 'truncate-end':
      return truncateEnd(line, maxWidth);
      
    case 'truncate-start':
      return truncateStart(line, maxWidth);
      
    case 'truncate-middle':
      return truncateMiddle(line, maxWidth);
      
    default:
      return line;
  }
};

const truncateEnd = (str: string, maxWidth: number): string => {
  const ellipsis = '…';
  const ellipsisWidth = 1;
  
  if (maxWidth < ellipsisWidth) {
    return '';
  }
  
  let width = 0;
  let result = '';
  
  for (const char of str) {
    const charWidth = stringWidth(char);
    
    if (width + charWidth + ellipsisWidth > maxWidth) {
      return result + ellipsis;
    }
    
    result += char;
    width += charWidth;
  }
  
  return result;
};
```

### 3. ANSI 序列感知的宽度计算

```typescript
import stringWidth from 'string-width';
import stripAnsi from 'strip-ansi';

// string-width 库已经处理了 ANSI 序列
const measureText = (text: string): {width: number; height: number} => {
  const lines = text.split('\n');
  let maxWidth = 0;
  
  for (const line of lines) {
    // stringWidth 会自动剥离 ANSI 序列并正确计算宽度
    // 包括处理 emoji、中文等宽字符
    const width = stringWidth(line);
    maxWidth = Math.max(maxWidth, width);
  }
  
  return {
    width: maxWidth,
    height: lines.length
  };
};

// 处理包含 ANSI 的文本切片
const sliceAnsi = (str: string, start: number, end?: number): string => {
  // 使用 slice-ansi 库保持 ANSI 序列完整性
  return sliceAnsiString(str, start, end);
};
```

## Transform 组件

### 1. 高级文本转换

`src/components/Transform.tsx` 允许自定义文本转换：

```typescript
interface TransformProps {
  readonly transform: (output: string, index: number) => string;
  readonly children?: ReactNode;
}

const Transform: FC<TransformProps> = ({transform, children}) => {
  return (
    <ink-text internal_transform={transform}>
      {children}
    </ink-text>
  );
};

// 使用示例：渐变效果
const GradientText: FC = ({children}) => {
  const gradient = createGradient(['#FF0000', '#00FF00', '#0000FF']);
  
  return (
    <Transform
      transform={(output, index) => {
        return output.split('').map((char, i) => {
          const color = gradient.getColorAt(i / output.length);
          return chalk.hex(color)(char);
        }).join('');
      }}
    >
      {children}
    </Transform>
  );
};
```

### 2. 逐行转换

```typescript
// 实现悬挂缩进
const HangingIndent: FC<{indent?: number}> = ({indent = 4, children}) => {
  return (
    <Transform
      transform={(line, lineIndex) => {
        // 第一行不缩进，其他行缩进
        return lineIndex === 0 ? line : ' '.repeat(indent) + line;
      }}
    >
      {children}
    </Transform>
  );
};

// 行号显示
const LineNumbers: FC = ({children}) => {
  return (
    <Transform
      transform={(line, lineIndex) => {
        const lineNo = String(lineIndex + 1).padStart(3, ' ');
        return chalk.gray(`${lineNo} │ `) + line;
      }}
    >
      {children}
    </Transform>
  );
};
```

## 边框渲染

### 1. 边框样式系统

`src/render-border.ts` 实现了丰富的边框样式：

```typescript
import cliBoxes from 'cli-boxes';

const renderBorder = (x: number, y: number, node: DOMElement, output: Output) => {
  const {yogaNode, style} = node;
  
  if (!style.borderStyle || !yogaNode) {
    return;
  }
  
  const width = yogaNode.getComputedWidth();
  const height = yogaNode.getComputedHeight();
  
  // 获取边框字符
  const borderChars = typeof style.borderStyle === 'string'
    ? cliBoxes[style.borderStyle]
    : style.borderStyle;
  
  const {
    topLeft, top, topRight,
    left, right,
    bottomLeft, bottom, bottomRight
  } = borderChars;
  
  // 应用边框颜色
  const colorize = (char: string, edge: string) => {
    const color = style[`border${edge}Color`] || style.borderColor;
    const dimmed = style[`border${edge}DimColor`] || style.borderDimColor;
    
    if (color) {
      char = chalk[color](char);
    }
    
    if (dimmed) {
      char = chalk.dim(char);
    }
    
    return char;
  };
  
  // 渲染顶部边框
  if (style.borderTop !== false) {
    output.write(x, y, colorize(topLeft, 'Top'));
    output.write(x + 1, y, colorize(top.repeat(width - 2), 'Top'));
    output.write(x + width - 1, y, colorize(topRight, 'Top'));
  }
  
  // 渲染侧边
  for (let row = 1; row < height - 1; row++) {
    if (style.borderLeft !== false) {
      output.write(x, y + row, colorize(left, 'Left'));
    }
    
    if (style.borderRight !== false) {
      output.write(x + width - 1, y + row, colorize(right, 'Right'));
    }
  }
  
  // 渲染底部边框
  if (style.borderBottom !== false) {
    output.write(x, y + height - 1, colorize(bottomLeft, 'Bottom'));
    output.write(x + 1, y + height - 1, colorize(bottom.repeat(width - 2), 'Bottom'));
    output.write(x + width - 1, y + height - 1, colorize(bottomRight, 'Bottom'));
  }
};
```

### 2. 自定义边框样式

```typescript
// 预定义的边框样式
const borderStyles = {
  single: {
    topLeft: '┌', top: '─', topRight: '┐',
    left: '│', right: '│',
    bottomLeft: '└', bottom: '─', bottomRight: '┘'
  },
  double: {
    topLeft: '╔', top: '═', topRight: '╗',
    left: '║', right: '║',
    bottomLeft: '╚', bottom: '═', bottomRight: '╝'
  },
  round: {
    topLeft: '╭', top: '─', topRight: '╮',
    left: '│', right: '│',
    bottomLeft: '╰', bottom: '─', bottomRight: '╯'
  },
  bold: {
    topLeft: '┏', top: '━', topRight: '┓',
    left: '┃', right: '┃',
    bottomLeft: '┗', bottom: '━', bottomRight: '┛'
  }
};

// 创建自定义边框
const customBorder = {
  topLeft: '◢', top: '▔', topRight: '◣',
  left: '▏', right: '▕',
  bottomLeft: '◥', bottom: '▁', bottomRight: '◤'
};
```

## 特殊字符处理

### 1. Unicode 和 Emoji 支持

```typescript
const handleSpecialCharacters = (text: string) => {
  // 正确处理宽字符（CJK、Emoji）
  const chars = [...text];  // 使用扩展运算符正确分割 Unicode
  
  let width = 0;
  for (const char of chars) {
    const charWidth = getCharacterWidth(char);
    width += charWidth;
  }
  
  return width;
};

const getCharacterWidth = (char: string): number => {
  const code = char.codePointAt(0);
  
  if (!code) return 0;
  
  // Emoji 和其他宽字符
  if (
    // Emoji 范围
    (code >= 0x1F300 && code <= 0x1F9FF) ||
    // CJK 字符
    (code >= 0x4E00 && code <= 0x9FFF) ||
    // 全角字符
    (code >= 0xFF00 && code <= 0xFFEF)
  ) {
    return 2;
  }
  
  // 控制字符
  if (code < 0x20 || (code >= 0x7F && code <= 0x9F)) {
    return 0;
  }
  
  return 1;
};
```

### 2. 控制字符过滤

```typescript
const sanitizeText = (text: string): string => {
  // 移除危险的控制字符，保留必要的
  return text.replace(/[\x00-\x08\x0B-\x0C\x0E-\x1F\x7F]/g, '');
};

// 转义特殊字符
const escapeAnsi = (text: string): string => {
  // 转义现有的 ANSI 序列，防止注入
  return text.replace(/\x1B/g, '\\x1B');
};
```

## 性能优化

### 1. 样式缓存

```typescript
class StyleCache {
  private cache = new Map<string, string>();
  
  getStyledText(text: string, styles: TextStyles): string {
    const key = this.createKey(text, styles);
    
    if (this.cache.has(key)) {
      return this.cache.get(key)!;
    }
    
    const styledText = this.applyStyles(text, styles);
    this.cache.set(key, styledText);
    
    // 限制缓存大小
    if (this.cache.size > 1000) {
      const firstKey = this.cache.keys().next().value;
      this.cache.delete(firstKey);
    }
    
    return styledText;
  }
  
  private createKey(text: string, styles: TextStyles): string {
    return `${text}:${JSON.stringify(styles)}`;
  }
}
```

### 2. ANSI 序列优化

```typescript
class AnsiOptimizer {
  optimize(segments: TextSegment[]): string {
    let result = '';
    let lastStyle: TextStyles = {};
    
    for (const segment of segments) {
      const styleDiff = this.getStyleDiff(lastStyle, segment.style);
      
      if (Object.keys(styleDiff).length > 0) {
        // 只输出变化的样式
        result += this.generateAnsiCodes(styleDiff);
      }
      
      result += segment.text;
      lastStyle = segment.style;
    }
    
    // 最后重置样式
    if (Object.keys(lastStyle).length > 0) {
      result += '\x1B[0m';
    }
    
    return result;
  }
}
```

## 总结

Ink 的文本渲染和样式系统展现了如何将 Web 开发的概念优雅地映射到终端环境：

1. **完整的样式支持**：通过 Chalk 和 ANSI 序列支持丰富的文本样式
2. **智能文本处理**：自动换行、截断、Unicode 支持等
3. **灵活的转换系统**：Transform 组件允许自定义文本处理
4. **高性能实现**：通过缓存、优化等策略确保渲染效率
5. **边框系统**：提供丰富的边框样式，增强视觉效果

这个系统让开发者能够在终端中创建视觉丰富、布局灵活的用户界面，同时保持了终端应用的简洁和高效。