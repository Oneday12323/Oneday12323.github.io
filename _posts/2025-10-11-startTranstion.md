---
title: startTransition 详解
tags: React 性能优化 startTransition useTransition
article_header:
  type: cover
---
## 一、使用场景

### 场景 1：搜索过滤大量数据

```jsx
import { useState, startTransition } from 'react';

function SearchableList({ items }) {
  const [query, setQuery] = useState('');
  const [filteredItems, setFilteredItems] = useState(items);
  
  const handleSearch = (e) => {
    const value = e.target.value;
    
    // 高优先级：立即更新输入框，保持响应
    setQuery(value);
    
    // 低优先级：延迟更新列表，可以被打断
    startTransition(() => {
      const filtered = items.filter(item => 
        item.name.toLowerCase().includes(value.toLowerCase())
      );
      setFilteredItems(filtered);
    });
  };
  
  return (
    <div>
      <input 
        value={query} 
        onChange={handleSearch}
        placeholder="搜索..." 
      />
      <ul>
        {filteredItems.map(item => (
          <li key={item.id}>{item.name}</li>
        ))}
      </ul>
    </div>
  );
}
```

**为什么需要 startTransition？**
- 如果列表有 10000 项，过滤计算很耗时
- 不使用 transition，输入框会卡顿
- 使用 transition，输入框保持流畅，列表延迟更新

### 场景 2：Tab 切换

```jsx
function TabContainer() {
  const [tab, setTab] = useState('home');
  const [isPending, startTransition] = useTransition();
  
  const handleTabClick = (newTab) => {
    startTransition(() => {
      setTab(newTab);
    });
  };
  
  return (
    <div>
      <div className="tabs">
        <button onClick={() => handleTabClick('home')}>首页</button>
        <button onClick={() => handleTabClick('profile')}>个人</button>
        <button onClick={() => handleTabClick('settings')}>设置</button>
      </div>
      
      {isPending && <Spinner />}
      
      <div className="content">
        {tab === 'home' && <HomePage />}
        {tab === 'profile' && <ProfilePage />}
        {tab === 'settings' && <SettingsPage />}
      </div>
    </div>
  );
}
```

### 场景 3：实时数据可视化

```jsx
function Chart({ dataPoints }) {
  const [selectedRange, setSelectedRange] = useState('1M');
  const [chartData, setChartData] = useState([]);
  const [isPending, startTransition] = useTransition();
  
  const handleRangeChange = (range) => {
    setSelectedRange(range); // 立即更新按钮状态
    
    startTransition(() => {
      // 复杂的数据处理和图表渲染
      const processed = processChartData(dataPoints, range);
      setChartData(processed);
    });
  };
  
  return (
    <div>
      <RangeSelector 
        value={selectedRange} 
        onChange={handleRangeChange}
        disabled={isPending}
      />
      {isPending ? <ChartSkeleton /> : <ComplexChart data={chartData} />}
    </div>
  );
}
```

### 场景 4：路由切换

```jsx
function App() {
  const [page, setPage] = useState('home');
  const [isPending, startTransition] = useTransition();
  
  const navigate = (newPage) => {
    startTransition(() => {
      setPage(newPage);
    });
  };
  
  return (
    <div>
      <nav style={{ opacity: isPending ? 0.7 : 1 }}>
        <button onClick={() => navigate('home')}>Home</button>
        <button onClick={() => navigate('about')}>About</button>
      </nav>
      
      {isPending && <div>Loading...</div>}
      
      {page === 'home' && <HomePage />}
      {page === 'about' && <AboutPage />}
    </div>
  );
}
```

## 二、工作原理

### 1. 优先级系统

React 内部维护多个优先级车道（Lane）：

```
同步优先级 (SyncLane)          → 最高优先级，立即执行
输入连续优先级 (InputContinuousLane) → 用户输入
默认优先级 (DefaultLane)       → 普通更新
过渡优先级 (TransitionLane)    → startTransition 标记的更新
空闲优先级 (IdleLane)          → 最低优先级
```

### 2. 执行流程

```jsx
function SearchComponent() {
  const [input, setInput] = useState('');
  const [results, setResults] = useState([]);
  
  const handleChange = (e) => {
    const value = e.target.value;
    
    // Step 1: 高优先级更新被标记
    setInput(value);  // 优先级：DefaultLane
    
    // Step 2: 低优先级更新被标记
    startTransition(() => {
      setResults(search(value));  // 优先级：TransitionLane
    });
    
    // Step 3: React 调度器处理
    // - 先处理高优先级更新（input）
    // - 渲染输入框
    // - 再处理低优先级更新（results）
    // - 如果有新的高优先级更新进来，会中断低优先级更新
  };
  
  return (
    <>
      <input value={input} onChange={handleChange} />
      <ResultList items={results} />
    </>
  );
}
```

### 3. 可中断渲染

```jsx
// 模拟 React 内部调度逻辑
function scheduleUpdate(update) {
  if (update.lane === TransitionLane) {
    // 过渡更新：可中断
    scheduleInterruptibleWork(() => {
      // 开始渲染
      while (hasMoreWork() && !shouldYield()) {
        performUnitOfWork();
      }
      
      // 如果有更高优先级的更新
      if (hasHigherPriorityWork()) {
        // 中断当前工作，处理高优先级更新
        interruptCurrentWork();
        scheduleHighPriorityWork();
      }
    });
  } else {
    // 非过渡更新：不可中断
    scheduleUninterruptibleWork(() => {
      performAllWork();
    });
  }
}
```

### 4. 时间切片（Time Slicing）

```jsx
// React 内部实现简化版
const FRAME_BUDGET = 5; // 每帧 5ms

function workLoop() {
  while (workInProgress !== null && !shouldYield()) {
    workInProgress = performUnitOfWork(workInProgress);
  }
  
  if (workInProgress !== null) {
    // 还有工作未完成，让出控制权
    return true; // 继续调度
  }
  
  return false; // 工作完成
}

function shouldYield() {
  // 当前帧时间用完了吗？
  return getCurrentTime() >= deadline;
}

// 调度器
function scheduler() {
  const shouldContinue = workLoop();
  
  if (shouldContinue) {
    // 下一帧继续
    requestIdleCallback(scheduler);
  }
}
```

## 三、startTransition vs useTransition

### startTransition（函数）

```jsx
import { startTransition } from 'react';

function Component() {
  const handleClick = () => {
    startTransition(() => {
      setTab('next');
    });
    // 无法知道是否 pending
  };
}
```

**特点：**
- 只是标记更新优先级
- 没有 pending 状态
- 更轻量

### useTransition（Hook）

```jsx
import { useTransition } from 'react';

function Component() {
  const [isPending, startTransition] = useTransition();
  
  const handleClick = () => {
    startTransition(() => {
      setTab('next');
    });
  };
  
  return (
    <div>
      {isPending && <Spinner />}
      <button disabled={isPending} onClick={handleClick}>
        切换
      </button>
    </div>
  );
}
```

**特点：**
- 提供 isPending 状态
- 可以显示加载指示器
- 可以禁用按钮防止重复点击

## 四、性能对比

### 不使用 startTransition

```jsx
function SlowComponent() {
  const [input, setInput] = useState('');
  const [list, setList] = useState(generateList(10000));
  
  const handleChange = (e) => {
    const value = e.target.value;
    setInput(value);
    setList(filterList(value)); // 同步更新，阻塞 UI
  };
  
  // 结果：输入框卡顿，用户体验差
}
```

**时间轴：**
```
用户输入 'a' → 等待 200ms（过滤 + 渲染）→ 输入框显示 'a'
用户输入 'ab' → 等待 200ms → 输入框显示 'ab'
```

### 使用 startTransition

```jsx
function FastComponent() {
  const [input, setInput] = useState('');
  const [list, setList] = useState(generateList(10000));
  const [isPending, startTransition] = useTransition();
  
  const handleChange = (e) => {
    const value = e.target.value;
    setInput(value); // 立即更新
    
    startTransition(() => {
      setList(filterList(value)); // 可中断更新
    });
  };
  
  // 结果：输入框流畅，列表延迟更新
}
```

**时间轴：**
```
用户输入 'a' → 立即显示 'a' → 后台更新列表
用户输入 'ab' → 立即显示 'ab' → 中断上次更新 → 重新更新列表
```

## 五、最佳实践

### ✅ 适合使用 startTransition 的场景

```jsx
// 1. 大量 DOM 更新
startTransition(() => {
  setItems(largeArray);
});

// 2. 复杂计算
startTransition(() => {
  setResult(complexCalculation());
});

// 3. 非紧急 UI 更新
startTransition(() => {
  setTab('settings');
});
```

### ❌ 不适合使用 startTransition 的场景

```jsx
// 1. 用户输入（需要立即响应）
setInput(e.target.value); // 不要用 startTransition

// 2. 悬停效果
setHovered(true); // 不要用 startTransition

// 3. 表单验证
setError('Invalid email'); // 不要用 startTransition

// 4. 控制受控组件
setValue(newValue); // 不要用 startTransition
```

### 组合使用示例

```jsx
function SmartSearch() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  const [isPending, startTransition] = useTransition();
  
  // 使用 useDeferredValue 作为替代方案
  const deferredQuery = useDeferredValue(query);
  
  useEffect(() => {
    setResults(search(deferredQuery));
  }, [deferredQuery]);
  
  return (
    <div>
      <input 
        value={query}
        onChange={(e) => setQuery(e.target.value)}
      />
      {isPending ? <Skeleton /> : <Results items={results} />}
    </div>
  );
}
```

## 六、与其他 API 的对比

| 特性 | startTransition | setTimeout | useDeferredValue |
|------|----------------|------------|------------------|
| 可中断 | ✅ | ❌ | ✅ |
| 优先级感知 | ✅ | ❌ | ✅ |
| 固定延迟 | ❌ | ✅ | ❌ |
| 使用复杂度 | 低 | 低 | 中 |
| 性能优化 | 最优 | 次优 | 最优 |

**总结：** startTransition 是 React 18 提供的强大工具，用于优化非紧急更新，通过优先级调度和可中断渲染，让 UI 保持响应性和流畅性。