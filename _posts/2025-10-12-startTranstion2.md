---
title: startTranstion中断更新的底层原理
tags: React 性能优化 startTransition useTransition
article_header:
  type: cover
---
## 一、Fiber 架构基础

在理解中断机制前，需要先了解 React 的 Fiber 架构。

### 1. Fiber 节点结构

```javascript
// 简化的 Fiber 节点结构
class FiberNode {
  constructor() {
    // 节点信息
    this.type = null;           // 组件类型
    this.key = null;
    this.stateNode = null;      // 真实 DOM 节点
    
    // Fiber 树结构
    this.return = null;         // 父节点
    this.child = null;          // 第一个子节点
    this.sibling = null;        // 下一个兄弟节点
    
    // 工作相关
    this.alternate = null;      // 双缓冲：指向另一棵树的对应节点
    this.flags = NoFlags;       // 副作用标记
    this.lanes = NoLanes;       // 优先级车道
    
    // 更新队列
    this.updateQueue = null;
  }
}
```

### 2. 双缓冲树

React 维护两棵 Fiber 树：

```javascript
// current 树：当前屏幕显示的内容
const currentFiber = {
  tag: 'div',
  child: { tag: 'span', props: { children: 'Hello' } }
};

// workInProgress 树：正在构建的新树
const workInProgressFiber = {
  tag: 'div',
  child: { tag: 'span', props: { children: 'World' } },
  alternate: currentFiber  // 指向 current 树
};
```

## 二、优先级车道系统

### 1. Lane 模型

```javascript
// Lane 使用二进制位表示优先级
const NoLanes = 0b0000000000000000000000000000000;
const SyncLane = 0b0000000000000000000000000000001;  // 同步，最高优先级
const InputContinuousLane = 0b0000000000000000000000000000100;  // 连续输入
const DefaultLane = 0b0000000000000000000000000010000;  // 默认优先级
const TransitionLane1 = 0b0000000000000000000000001000000;  // 过渡优先级
const TransitionLane2 = 0b0000000000000000000000010000000;
// ... 更多 transition lanes

const IdleLane = 0b0100000000000000000000000000000;  // 空闲优先级
```

### 2. 优先级判断

```javascript
function isHigherPriority(lane1, lane2) {
  // 数值越小，优先级越高
  return lane1 < lane2;
}

// 示例
isHigherPriority(SyncLane, TransitionLane1);  // true
isHigherPriority(DefaultLane, TransitionLane1);  // true
isHigherPriority(TransitionLane1, DefaultLane);  // false
```

## 三、中断更新的核心机制

### 1. 工作循环（Work Loop）

```javascript
// React 的核心工作循环（简化版）
let workInProgress = null;  // 当前正在处理的 Fiber 节点
let currentEventTime = NoTimestamp;
let currentEventPriority = DefaultLane;

function workLoopConcurrent() {
  // 并发模式的工作循环
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);
  }
}

function performUnitOfWork(unitOfWork) {
  const current = unitOfWork.alternate;
  
  // 1. 开始处理这个 Fiber 节点
  let next = beginWork(current, unitOfWork, renderLanes);
  
  if (next === null) {
    // 2. 没有子节点，完成这个节点
    completeUnitOfWork(unitOfWork);
  } else {
    // 3. 有子节点，继续处理子节点
    workInProgress = next;
  }
}
```

### 2. shouldYield - 决定是否中断

```javascript
// 判断是否应该让出执行权
function shouldYield() {
  const currentTime = getCurrentTime();
  
  // 1. 时间片用完了？
  if (currentTime >= deadline) {
    return true;
  }
  
  // 2. 有更高优先级的任务？
  if (hasHigherPriorityWork()) {
    return true;
  }
  
  return false;
}

// 检查是否有更高优先级的工作
function hasHigherPriorityWork() {
  const nextLanes = getNextLanes(root, NoLanes);
  const currentLane = getCurrentRenderLane();
  
  // 比较优先级
  return nextLanes !== NoLanes && nextLanes < currentLane;
}
```

### 3. 时间切片

```javascript
// 每帧的时间预算（毫秒）
const frameYieldMs = 5;
let deadline = 0;

function schedulerCallback() {
  const currentTime = getCurrentTime();
  deadline = currentTime + frameYieldMs;
  
  // 开始工作
  const hasMoreWork = workLoopConcurrent();
  
  if (hasMoreWork) {
    // 还有工作，但时间片用完了，继续调度
    return schedulerCallback;
  } else {
    // 工作完成
    return null;
  }
}

// 使用浏览器 API 调度
scheduler.scheduleCallback(schedulerCallback);
```

## 四、中断更新的完整流程

### 场景模拟

```jsx
function App() {
  const [input, setInput] = useState('');
  const [list, setList] = useState(generateItems(10000));
  
  const handleChange = (e) => {
    const value = e.target.value;
    
    // 高优先级更新
    setInput(value);  // Lane: DefaultLane (优先级 16)
    
    // 低优先级更新
    startTransition(() => {
      setList(filterItems(value));  // Lane: TransitionLane1 (优先级 64)
    });
  };
  
  return (
    <>
      <input value={input} onChange={handleChange} />
      <List items={list} />  {/* 假设渲染 10000 个项目很慢 */}
    </>
  );
}
```

### 执行流程详解

```javascript
// ============ 第 1 步：用户输入 ============
// 用户在 t=0ms 时输入 'a'

// 创建两个更新
const update1 = {
  lane: DefaultLane,        // 优先级 16
  action: () => 'a'
};

const update2 = {
  lane: TransitionLane1,    // 优先级 64
  action: () => filterItems('a')
};

// 调度更新
scheduleUpdateOnFiber(inputFiber, DefaultLane);
scheduleUpdateOnFiber(listFiber, TransitionLane1);

// ============ 第 2 步：开始渲染高优先级更新 ============
// t=1ms: 调度器选择最高优先级的 lane
const renderLanes = getNextLanes(root);  // 返回 DefaultLane

// 开始工作循环
workLoopConcurrent();

// 处理 Fiber 树
// App Fiber (处理中...)
//   ├─ input Fiber (更新 value='a') ✅
//   └─ List Fiber (跳过，优先级不够)

// t=3ms: 完成高优先级渲染
// 提交更新，input 显示 'a'

// ============ 第 3 步：开始渲染低优先级更新 ============
// t=4ms: 调度器调度 TransitionLane1 更新

renderLanes = TransitionLane1;
workInProgress = rootFiber;
deadline = 4 + 5 = 9ms;  // 5ms 时间片

// 开始处理 List 组件（10000 个项目）
workLoopConcurrent();

// t=5ms: 处理第 1000 个项目
performUnitOfWork(listItemFiber_1000);
shouldYield();  // false，继续

// t=6ms: 处理第 2000 个项目
performUnitOfWork(listItemFiber_2000);
shouldYield();  // false，继续

// t=7ms: 处理第 3000 个项目
performUnitOfWork(listItemFiber_3000);
shouldYield();  // false，继续

// t=8ms: 处理第 4000 个项目
performUnitOfWork(listItemFiber_4000);
shouldYield();  // false，继续

// ============ 第 4 步：用户再次输入！============
// t=9ms: 用户输入 'ab'

// 创建新的高优先级更新
const update3 = {
  lane: DefaultLane,        // 优先级 16
  action: () => 'ab'
};

scheduleUpdateOnFiber(inputFiber, DefaultLane);

// ============ 第 5 步：中断发生！============
// t=9ms: 继续工作循环

performUnitOfWork(listItemFiber_5000);

// 检查是否应该让出
shouldYield();
  ↓
hasHigherPriorityWork();
  ↓
getNextLanes(root);  // 返回 DefaultLane (16)
getCurrentRenderLane();  // 返回 TransitionLane1 (64)
  ↓
DefaultLane < TransitionLane1  // true!

// 返回 true，中断当前工作！

// ============ 第 6 步：保存中断状态 ============
// 保存当前进度
const interruptedWork = workInProgress;  // listItemFiber_5000

// 标记为未完成
workInProgress = null;

// 保存未完成的 lanes
root.pendingLanes |= TransitionLane1;

// ============ 第 7 步：处理高优先级更新 ============
// t=10ms: 重新开始，处理 DefaultLane

renderLanes = DefaultLane;
workInProgress = rootFiber;

workLoopConcurrent();
// 快速更新 input，显示 'ab'

// t=12ms: 提交更新

// ============ 第 8 步：重新开始低优先级更新 ============
// t=13ms: 再次调度 TransitionLane1

renderLanes = TransitionLane1;
workInProgress = rootFiber;  // 从头开始！

// 注意：不是从 listItemFiber_5000 继续，而是从头开始
// 因为 props 可能已经变了（input 从 'a' 变成 'ab'）

workLoopConcurrent();
// 处理完整的 filterItems('ab')

// t=25ms: 完成并提交
```

## 五、关键代码实现

### 1. 调度更新

```javascript
function scheduleUpdateOnFiber(fiber, lane) {
  // 1. 标记 fiber 到 root 路径上的所有节点
  const root = markUpdateLaneFromFiberToRoot(fiber, lane);
  
  // 2. 标记 root 有待处理的 lanes
  markRootUpdated(root, lane);
  
  // 3. 调度根节点的更新
  ensureRootIsScheduled(root);
}

function ensureRootIsScheduled(root) {
  // 获取下一个要处理的 lane
  const nextLanes = getNextLanes(root, NoLanes);
  
  if (nextLanes === NoLanes) {
    // 没有工作要做
    return;
  }
  
  // 获取优先级
  const newCallbackPriority = getHighestPriorityLane(nextLanes);
  
  // 取消旧的调度
  if (existingCallbackPriority !== newCallbackPriority) {
    if (existingCallbackNode !== null) {
      cancelCallback(existingCallbackNode);
    }
  }
  
  // 调度新的工作
  if (newCallbackPriority === SyncLane) {
    // 同步优先级，立即执行
    scheduleSyncCallback(performSyncWorkOnRoot.bind(null, root));
  } else {
    // 并发优先级，可中断
    const schedulerPriority = lanesToSchedulerPriority(newCallbackPriority);
    newCallbackNode = scheduleCallback(
      schedulerPriority,
      performConcurrentWorkOnRoot.bind(null, root)
    );
  }
}
```

### 2. 渲染阶段

```javascript
function renderRootConcurrent(root, lanes) {
  // 准备新的工作栈
  prepareFreshStack(root, lanes);
  
  do {
    try {
      workLoopConcurrent();
      break;
    } catch (thrownValue) {
      handleError(root, thrownValue);
    }
  } while (true);
  
  // 检查是否完成
  if (workInProgress !== null) {
    // 未完成，被中断了
    return RootInProgress;
  } else {
    // 完成了
    return workInProgressRootExitStatus;
  }
}

function workLoopConcurrent() {
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);
  }
}
```

### 3. beginWork - 处理 Fiber 节点

```javascript
function beginWork(current, workInProgress, renderLanes) {
  // 检查这个 fiber 是否需要更新
  if (current !== null) {
    const oldProps = current.memoizedProps;
    const newProps = workInProgress.pendingProps;
    
    // props 没变，且优先级不够
    if (
      oldProps === newProps &&
      !includesSomeLane(renderLanes, workInProgress.lanes)
    ) {
      // 跳过这个节点
      return bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes);
    }
  }
  
  // 需要更新，清除优先级
  workInProgress.lanes = NoLanes;
  
  // 根据 tag 类型处理
  switch (workInProgress.tag) {
    case FunctionComponent:
      return updateFunctionComponent(current, workInProgress, renderLanes);
    case ClassComponent:
      return updateClassComponent(current, workInProgress, renderLanes);
    case HostComponent:  // div, span 等
      return updateHostComponent(current, workInProgress, renderLanes);
    // ... 其他类型
  }
}
```

### 4. 优先级比较

```javascript
function includesSomeLane(a, b) {
  // 使用位运算检查是否有交集
  return (a & b) !== NoLanes;
}

function isSubsetOfLanes(set, subset) {
  // subset 是否是 set 的子集
  return (set & subset) === subset;
}

function mergeLanes(a, b) {
  // 合并 lanes
  return a | b;
}

function removeLanes(set, subset) {
  // 从 set 中移除 subset
  return set & ~subset;
}
```

## 六、实际案例分析

### 案例：快速输入场景

```jsx
function SearchBox() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  
  const handleChange = (e) => {
    const value = e.target.value;
    setQuery(value);
    
    startTransition(() => {
      setResults(expensiveSearch(value));
    });
  };
  
  return (
    <>
      <input value={query} onChange={handleChange} />
      <ResultsList items={results} />
    </>
  );
}
```

**用户快速输入 "react" 的时间线：**

```
t=0ms   用户输入 'r'
        → setQuery('r') [DefaultLane]
        → startTransition(() => setResults(search('r'))) [TransitionLane]

t=1ms   开始渲染 DefaultLane
        → 更新 input.value = 'r' ✅

t=2ms   提交 DefaultLane，input 显示 'r'

t=3ms   开始渲染 TransitionLane (search('r'))
        → 处理中... (耗时约 20ms)

t=8ms   用户输入 'e' (输入过程中)
        → setQuery('re') [DefaultLane]
        → startTransition(() => setResults(search('re'))) [TransitionLane]

t=9ms   ⚠️ 检测到高优先级更新
        → shouldYield() 返回 true
        → 中断 search('r') 的渲染
        → 保存进度，丢弃未完成的工作

t=10ms  开始渲染新的 DefaultLane
        → 更新 input.value = 're' ✅

t=11ms  提交，input 显示 're'

t=12ms  开始渲染新的 TransitionLane (search('re'))
        → 从头开始处理 (不是继续 search('r'))

t=15ms  用户输入 'a'
        → 再次中断...

... (重复过程)

t=100ms 用户停止输入，最后输入完 "react"
t=101ms 开始渲染 search('react')
t=120ms 完成并显示结果
```

## 七、总结

**中断更新的核心要素：**

1. **Fiber 架构** - 将渲染工作拆分成小单元
2. **优先级车道** - 用二进制位表示不同优先级
3. **时间切片** - 每 5ms 检查一次是否需要让出
4. **shouldYield** - 判断是否有更高优先级工作
5. **workInProgress** - 追踪当前工作进度
6. **从头开始** - 中断后重新开始，保证一致性

**为什么要从头开始而不是继续？**
- props 可能已经改变
- 保证 UI 一致性
- 避免基于过期数据的计算
- 简化实现复杂度