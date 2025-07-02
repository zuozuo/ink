# 终端交互能力原理深度解析

## 概述

Ink 提供了丰富的终端交互能力，包括键盘输入处理、焦点管理、鼠标支持等。这些功能让开发者能够构建真正交互式的命令行应用。本文将深入分析这些交互能力的实现原理。

## 输入处理架构

### 1. 输入流管理

Ink 通过 stdin 流捕获用户输入：

```typescript
// src/components/StdinContext.ts
export const StdinContext = createContext<Props>({
  stdin: undefined,
  setRawMode: () => {},
  isRawModeSupported: false,
  internal_exitOnCtrlC: true,
});

// 在 App 组件中提供 stdin
const App: FC<Props> = ({children, stdin, exitOnCtrlC}) => {
  const [rawMode, setRawMode] = useState(false);
  
  // 检查是否支持 raw mode
  const isRawModeSupported = stdin?.isTTY && typeof stdin?.setRawMode === 'function';
  
  useEffect(() => {
    if (!isRawModeSupported || !rawMode) {
      return;
    }
    
    stdin.setRawMode(true);
    
    return () => {
      stdin.setRawMode(false);
    };
  }, [stdin, rawMode, isRawModeSupported]);
  
  return (
    <StdinContext.Provider
      value={{
        stdin,
        setRawMode,
        isRawModeSupported,
        internal_exitOnCtrlC: exitOnCtrlC
      }}
    >
      {children}
    </StdinContext.Provider>
  );
};
```

### 2. Raw Mode 的重要性

Raw Mode 是终端交互的关键：

- **Normal Mode**：终端缓冲输入，只有按下回车才发送
- **Raw Mode**：每个按键立即发送，支持特殊键（箭头键、功能键等）

```typescript
// Raw Mode 设置
const enableRawMode = (stdin: NodeJS.ReadStream) => {
  if (stdin.isTTY && stdin.setRawMode) {
    stdin.setRawMode(true);
    
    // 配置终端行为
    stdin.resume();
    stdin.setEncoding('utf8');
    
    // 处理 Ctrl+C
    if (exitOnCtrlC) {
      stdin.on('data', data => {
        if (data === '\x03') {  // Ctrl+C
          process.exit(0);
        }
      });
    }
  }
};
```

## useInput Hook 实现

### 1. 核心实现

`src/hooks/use-input.ts` 提供了高级的输入处理：

```typescript
export interface Key {
  leftArrow: boolean;
  rightArrow: boolean;
  upArrow: boolean;
  downArrow: boolean;
  return: boolean;
  escape: boolean;
  ctrl: boolean;
  shift: boolean;
  tab: boolean;
  backspace: boolean;
  delete: boolean;
  pageUp: boolean;
  pageDown: boolean;
  meta: boolean;
}

const useInput = (
  inputHandler: (input: string, key: Key) => void,
  options: {isActive?: boolean} = {}
) => {
  const {stdin, setRawMode} = useStdin();
  const {isActive = true} = options;
  
  useEffect(() => {
    if (!stdin || !isActive) {
      return;
    }
    
    const handleData = (data: string) => {
      const key = parseKeypress(data);
      inputHandler(data, key);
    };
    
    // 启用 raw mode
    setRawMode(true);
    stdin.on('data', handleData);
    
    return () => {
      stdin.off('data', handleData);
      setRawMode(false);
    };
  }, [stdin, inputHandler, isActive, setRawMode]);
};
```

### 2. 按键解析

`src/parse-keypress.ts` 负责解析按键序列：

```typescript
const parseKeypress = (data: string): Key => {
  const key: Key = {
    leftArrow: false,
    rightArrow: false,
    upArrow: false,
    downArrow: false,
    return: false,
    escape: false,
    ctrl: false,
    shift: false,
    tab: false,
    backspace: false,
    delete: false,
    pageUp: false,
    pageDown: false,
    meta: false
  };
  
  // ANSI 转义序列映射
  const sequences: Record<string, Partial<Key>> = {
    '\x1B[A': {upArrow: true},
    '\x1B[B': {downArrow: true},
    '\x1B[C': {rightArrow: true},
    '\x1B[D': {leftArrow: true},
    '\x1B[5~': {pageUp: true},
    '\x1B[6~': {pageDown: true},
    '\r': {return: true},
    '\n': {return: true},
    '\x1B': {escape: true},
    '\t': {tab: true},
    '\x7F': {backspace: true},
    '\x1B[3~': {delete: true}
  };
  
  // 检查特殊序列
  if (sequences[data]) {
    return {...key, ...sequences[data]};
  }
  
  // 检查 Ctrl 组合键
  if (data.length === 1 && data.charCodeAt(0) < 32) {
    key.ctrl = true;
  }
  
  // 检查 Shift（大写字母）
  if (data.length === 1 && data >= 'A' && data <= 'Z') {
    key.shift = true;
  }
  
  return key;
};
```

### 3. 高级按键处理

支持组合键和特殊键：

```typescript
// 使用示例
const InteractiveComponent = () => {
  useInput((input, key) => {
    if (key.ctrl && input === 'c') {
      // Ctrl+C 处理
      handleQuit();
    } else if (key.ctrl && input === 's') {
      // Ctrl+S 保存
      handleSave();
    } else if (key.leftArrow) {
      // 左箭头
      moveCursorLeft();
    } else if (key.tab && key.shift) {
      // Shift+Tab
      focusPrevious();
    } else if (key.meta && input === 'v') {
      // Cmd+V (Mac) 粘贴
      handlePaste();
    }
  });
  
  return <Text>Interactive Component</Text>;
};
```

## 焦点管理系统

### 1. Focus Context

`src/components/FocusContext.ts` 实现了完整的焦点系统：

```typescript
interface FocusContext {
  activeId?: string;
  add: (id: string, options: {autoFocus?: boolean}) => void;
  remove: (id: string) => void;
  activate: (id: string) => void;
  deactivate: (id: string) => void;
  focus: (id: string) => void;
  focusNext: () => void;
  focusPrevious: () => void;
  disableFocus: () => void;
  enableFocus: () => void;
  focusedId?: string;
}

const FocusProvider: FC = ({children}) => {
  const [state, setState] = useState<FocusState>({
    isFocusEnabled: true,
    focusables: [],
    focusedId: undefined
  });
  
  const focusNext = useCallback(() => {
    setState(prevState => {
      const {focusables, focusedId} = prevState;
      const currentIndex = focusables.findIndex(f => f.id === focusedId);
      
      // 循环到下一个可聚焦元素
      let nextIndex = currentIndex + 1;
      if (nextIndex >= focusables.length) {
        nextIndex = 0;
      }
      
      // 跳过非激活的元素
      while (
        nextIndex !== currentIndex && 
        !focusables[nextIndex]?.isActive
      ) {
        nextIndex = (nextIndex + 1) % focusables.length;
      }
      
      return {
        ...prevState,
        focusedId: focusables[nextIndex]?.id
      };
    });
  }, []);
  
  // 监听 Tab 键
  useInput((input, key) => {
    if (!state.isFocusEnabled) return;
    
    if (key.tab && !key.shift) {
      focusNext();
    } else if (key.tab && key.shift) {
      focusPrevious();
    }
  });
  
  return (
    <FocusContext.Provider value={contextValue}>
      {children}
    </FocusContext.Provider>
  );
};
```

### 2. useFocus Hook

```typescript
interface UseFocusOptions {
  autoFocus?: boolean;
  isActive?: boolean;
  id?: string;
}

const useFocus = (options: UseFocusOptions = {}) => {
  const {autoFocus = false, isActive = true, id: customId} = options;
  const {add, remove, activate, deactivate, focusedId} = useFocusManager();
  
  // 生成唯一 ID
  const id = useMemo(() => customId || generateId(), [customId]);
  
  const isFocused = focusedId === id;
  
  useEffect(() => {
    add(id, {autoFocus});
    
    return () => {
      remove(id);
    };
  }, [id, autoFocus, add, remove]);
  
  useEffect(() => {
    if (isActive) {
      activate(id);
    } else {
      deactivate(id);
    }
  }, [id, isActive, activate, deactivate]);
  
  return {isFocused};
};
```

### 3. 焦点可视化

```typescript
const FocusableButton: FC<{label: string}> = ({label}) => {
  const {isFocused} = useFocus();
  
  return (
    <Box
      borderStyle="single"
      borderColor={isFocused ? 'blue' : 'gray'}
      paddingX={1}
    >
      <Text color={isFocused ? 'blue' : undefined}>
        {isFocused ? '▶ ' : '  '}{label}
      </Text>
    </Box>
  );
};
```

## 鼠标支持（实验性）

### 1. 鼠标事件处理

虽然不是核心功能，但 Ink 可以支持鼠标：

```typescript
const enableMouse = (stdin: NodeJS.ReadStream) => {
  // 启用鼠标报告
  process.stdout.write('\x1b[?1000h');  // 启用鼠标点击
  process.stdout.write('\x1b[?1002h');  // 启用鼠标移动
  
  return () => {
    // 禁用鼠标报告
    process.stdout.write('\x1b[?1000l');
    process.stdout.write('\x1b[?1002l');
  };
};

const parseMouseEvent = (data: string): MouseEvent | null => {
  // SGR 鼠标协议
  const match = data.match(/\x1b\[<(\d+);(\d+);(\d+)([mM])/);
  
  if (match) {
    const [, button, x, y, type] = match;
    
    return {
      button: parseInt(button),
      x: parseInt(x) - 1,  // 转为 0 索引
      y: parseInt(y) - 1,
      type: type === 'M' ? 'press' : 'release'
    };
  }
  
  return null;
};
```

### 2. 点击区域检测

```typescript
const useMouseClick = (onMouseClick: (x: number, y: number) => void) => {
  const ref = useRef<DOMElement>();
  
  useEffect(() => {
    const handleMouse = (event: MouseEvent) => {
      if (!ref.current || event.type !== 'press') return;
      
      const {yogaNode} = ref.current;
      if (!yogaNode) return;
      
      const elementX = yogaNode.getComputedLeft();
      const elementY = yogaNode.getComputedTop();
      const width = yogaNode.getComputedWidth();
      const height = yogaNode.getComputedHeight();
      
      // 检查点击是否在元素内
      if (
        event.x >= elementX &&
        event.x < elementX + width &&
        event.y >= elementY &&
        event.y < elementY + height
      ) {
        onMouseClick(event.x - elementX, event.y - elementY);
      }
    };
    
    mouseEvents.on('click', handleMouse);
    
    return () => {
      mouseEvents.off('click', handleMouse);
    };
  }, [onMouseClick]);
  
  return ref;
};
```

## 剪贴板集成

### 1. 复制到剪贴板

```typescript
const copyToClipboard = (text: string) => {
  // macOS
  if (process.platform === 'darwin') {
    execSync(`echo "${text}" | pbcopy`);
  }
  // Linux (需要 xclip)
  else if (process.platform === 'linux') {
    execSync(`echo "${text}" | xclip -selection clipboard`);
  }
  // Windows (PowerShell)
  else if (process.platform === 'win32') {
    execSync(`echo "${text}" | clip`);
  }
};
```

### 2. 从剪贴板粘贴

```typescript
const pasteFromClipboard = (): string => {
  try {
    if (process.platform === 'darwin') {
      return execSync('pbpaste').toString();
    } else if (process.platform === 'linux') {
      return execSync('xclip -selection clipboard -o').toString();
    } else if (process.platform === 'win32') {
      return execSync('powershell -command Get-Clipboard').toString();
    }
  } catch {
    return '';
  }
};
```

## 高级交互模式

### 1. 文本输入组件

```typescript
const TextInput: FC<{onSubmit: (value: string) => void}> = ({onSubmit}) => {
  const [value, setValue] = useState('');
  const [cursor, setCursor] = useState(0);
  
  useInput((input, key) => {
    if (key.return) {
      onSubmit(value);
    } else if (key.backspace) {
      setValue(prev => prev.slice(0, cursor - 1) + prev.slice(cursor));
      setCursor(prev => Math.max(0, prev - 1));
    } else if (key.delete) {
      setValue(prev => prev.slice(0, cursor) + prev.slice(cursor + 1));
    } else if (key.leftArrow) {
      setCursor(prev => Math.max(0, prev - 1));
    } else if (key.rightArrow) {
      setCursor(prev => Math.min(value.length, prev + 1));
    } else if (!key.ctrl && !key.meta && input) {
      setValue(prev => prev.slice(0, cursor) + input + prev.slice(cursor));
      setCursor(prev => prev + input.length);
    }
  });
  
  // 渲染输入框和光标
  const displayValue = 
    value.slice(0, cursor) + 
    '│' +  // 光标
    value.slice(cursor);
  
  return (
    <Box borderStyle="single">
      <Text>{displayValue}</Text>
    </Box>
  );
};
```

### 2. 选择列表

```typescript
const SelectList: FC<{items: string[]; onSelect: (item: string) => void}> = ({
  items,
  onSelect
}) => {
  const [selectedIndex, setSelectedIndex] = useState(0);
  
  useInput((input, key) => {
    if (key.upArrow) {
      setSelectedIndex(prev => Math.max(0, prev - 1));
    } else if (key.downArrow) {
      setSelectedIndex(prev => Math.min(items.length - 1, prev + 1));
    } else if (key.return) {
      onSelect(items[selectedIndex]);
    }
  });
  
  return (
    <Box flexDirection="column">
      {items.map((item, index) => (
        <Box key={index}>
          <Text color={index === selectedIndex ? 'blue' : undefined}>
            {index === selectedIndex ? '▶ ' : '  '}
            {item}
          </Text>
        </Box>
      ))}
    </Box>
  );
};
```

## 交互性能优化

### 1. 输入防抖

```typescript
const useDebouncedInput = (handler: InputHandler, delay = 100) => {
  const timeoutRef = useRef<NodeJS.Timeout>();
  const accumulatedInput = useRef('');
  
  useInput((input, key) => {
    if (key.return || key.escape) {
      // 立即处理特殊键
      handler(input, key);
      return;
    }
    
    accumulatedInput.current += input;
    
    if (timeoutRef.current) {
      clearTimeout(timeoutRef.current);
    }
    
    timeoutRef.current = setTimeout(() => {
      handler(accumulatedInput.current, key);
      accumulatedInput.current = '';
    }, delay);
  });
};
```

### 2. 输入缓冲

```typescript
class InputBuffer {
  private buffer: string[] = [];
  private maxSize = 100;
  
  push(input: string) {
    this.buffer.push(input);
    
    if (this.buffer.length > this.maxSize) {
      this.buffer.shift();
    }
  }
  
  getSequence(length: number): string {
    return this.buffer.slice(-length).join('');
  }
  
  clear() {
    this.buffer = [];
  }
}

// 用于检测输入序列
const useInputSequence = (sequences: Record<string, () => void>) => {
  const buffer = useRef(new InputBuffer());
  
  useInput((input) => {
    buffer.current.push(input);
    
    for (const [sequence, handler] of Object.entries(sequences)) {
      if (buffer.current.getSequence(sequence.length) === sequence) {
        handler();
        buffer.current.clear();
        break;
      }
    }
  });
};
```

## 无障碍支持

### 1. 屏幕阅读器集成

```typescript
const announceToScreenReader = (message: string) => {
  // 使用 ANSI 转义序列发送屏幕阅读器消息
  process.stdout.write(`\x1b]777;notify;${message}\x07`);
};

const AccessibleComponent: FC = () => {
  const {isFocused} = useFocus();
  
  useEffect(() => {
    if (isFocused) {
      announceToScreenReader('Button focused. Press Enter to activate.');
    }
  }, [isFocused]);
  
  return <Text>Accessible Button</Text>;
};
```

### 2. 键盘导航提示

```typescript
const KeyboardHints: FC = () => {
  const hints = [
    'Tab: Next item',
    'Shift+Tab: Previous item',
    'Enter: Select',
    'Esc: Cancel'
  ];
  
  return (
    <Box borderStyle="single" borderColor="gray">
      <Text dimColor>
        {hints.join(' | ')}
      </Text>
    </Box>
  );
};
```

## 总结

Ink 的终端交互能力展现了其作为现代 CLI 框架的强大之处：

1. **完整的输入系统**：从基础按键到组合键的全面支持
2. **智能焦点管理**：自动处理 Tab 导航和焦点切换
3. **高级交互组件**：文本输入、选择列表等开箱即用
4. **性能优化**：输入防抖、缓冲等机制确保流畅体验
5. **可扩展性**：支持鼠标、剪贴板等高级功能

这些交互能力让开发者能够构建不输于 GUI 应用的丰富交互体验，同时保持了终端应用的轻量和高效。