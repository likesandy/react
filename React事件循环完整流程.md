# React 事件循环完整流程

> 基于 React 源码分析，详细说明 React 在浏览器事件循环中的工作机制

## 📌 核心概念

### 两个不同的时间周期

| 周期类型 | 时长 | 用途 | 源码位置 |
|---------|------|------|---------|
| **React 时间切片** | **5ms** | Render 阶段让出控制权 | `packages/scheduler/src/SchedulerFeatureFlags.js:11` |
| **浏览器刷新率** | **16.6ms** (60Hz) | 屏幕渲染周期 | 浏览器原生机制 |

### 调度机制

```javascript
// 源码：packages/scheduler/src/forks/Scheduler.js:532-540
if (typeof MessageChannel !== 'undefined') {
  const channel = new MessageChannel();
  const port = channel.port2;
  channel.port1.onmessage = performWorkUntilDeadline;
  schedulePerformWorkUntilDeadline = () => {
    port.postMessage(null);
  };
}
```

**优先级顺序**：
1. `MessageChannel`（浏览器环境，优先使用）
2. `setImmediate`（Node.js 和旧版 IE）
3. `setTimeout`（兜底方案）

**原因**：避免 `setTimeout` 的 4ms 最小延迟限制

---

## 🔄 完整执行流程

### 1️⃣ 事件循环开始（宏任务）

```
用户交互（点击、输入）
    ↓
触发事件处理器
    ↓
setState() / dispatch()
    ↓
注册更新到 Scheduler 任务队列
    ↓
通过 MessageChannel 调度（创建宏任务，不是微任务！）
```

**关键代码**：

```javascript
// 源码：packages/react-reconciler/src/ReactFiberWorkLoop.js:3641
scheduleCallback(NormalSchedulerPriority, () => {
  flushPassiveEffects();
});
```

---

### 2️⃣ Render 阶段（可中断）

#### 工作循环实现

React 有两种并发工作循环：

##### 方式 1：基于 Scheduler（默认，生产环境）

```javascript
// 源码：packages/react-reconciler/src/ReactFiberWorkLoop.js:2992-2998
function workLoopConcurrentByScheduler() {
  // 通过 Scheduler 的 shouldYield() 检查是否需要让出控制权
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);
  }
}
```

##### 方式 2：固定时间间隔（实验性功能）

```javascript
// 源码：packages/react-reconciler/src/ReactFiberWorkLoop.js:2975-2989
function workLoopConcurrent(nonIdle: boolean) {
  // We yield every other "frame" when rendering Transition or Retries.
  // For Idle work we yield every 5ms to keep animations going smooth.
  if (workInProgress !== null) {
    const yieldAfter = now() + (nonIdle ? 25 : 5);
    do {
      performUnitOfWork(workInProgress);
    } while (workInProgress !== null && now() < yieldAfter);
  }
}
```

**时间切片检查**：

```javascript
// 源码：packages/scheduler/src/forks/Scheduler.js:447-460
function shouldYieldToHost(): boolean {
  if (!enableAlwaysYieldScheduler && enableRequestPaint && needsPaint) {
    return true;  // 需要绘制，立即让出
  }
  const timeElapsed = getCurrentTime() - startTime;
  if (timeElapsed < frameInterval) {  // frameInterval = 5ms
    return false;  // 还没超时，继续执行
  }
  return true;  // 超时，让出控制权
}
```

#### 时间切片流程

```
开始 Render
    ↓
执行约 5ms 的 Fiber 构建工作
    ↓
检查 shouldYield()
    ↓
如果返回 true：
    - 保存当前进度
    - 通过 MessageChannel.postMessage() 调度下一个宏任务
    - 让出主线程
    ↓
如果返回 false：
    - 继续执行
```

---

### 3️⃣ 清空微任务队列

```
宏任务执行完毕
    ↓
浏览器自动执行所有微任务
    - Promise.then()
    - queueMicrotask()
    - MutationObserver
    ↓
微任务队列清空
```

**注意**：React 的调度**不依赖微任务**，而是使用宏任务（MessageChannel）

---

### 4️⃣ Commit 阶段（不可中断，同步执行）

当 Render 工作完成后，React 进入 Commit 阶段：

```javascript
// 源码：packages/react-reconciler/src/ReactFiberWorkLoop.js:3706-3757
function commitRoot() {
  // 1. Before Mutation 阶段
  commitBeforeMutationEffects(root, finishedWork, lanes);

  // 2. Mutation 阶段（更新真实 DOM）
  pendingEffectsStatus = PENDING_MUTATION_PHASE;
  flushMutationEffects();

  // 3. Layout 阶段（执行 useLayoutEffect）
  flushLayoutEffects();

  // 4. 调度 Passive Effects（useEffect）
  if (enableYieldingBeforePassive) {
    // 新版本：依赖 ensureRootIsScheduled 调度
  } else {
    scheduleCallback(NormalSchedulerPriority, () => {
      flushPassiveEffects();  // useEffect 在下一个宏任务执行
    });
  }
}
```

#### Commit 阶段详解

| 子阶段 | 作用 | 是否同步 | 示例 |
|--------|------|---------|------|
| **Before Mutation** | 读取 DOM 状态 | ✅ 同步 | `getSnapshotBeforeUpdate` |
| **Mutation** | 更新真实 DOM | ✅ 同步 | DOM 插入/删除/更新 |
| **Layout** | DOM 更新后同步执行 | ✅ 同步 | `useLayoutEffect` |
| **Passive** | 异步调度 | ❌ 异步 | `useEffect`（下一个宏任务） |

#### useLayoutEffect vs useEffect

```javascript
// useLayoutEffect：Commit 阶段同步执行
// 源码：ReactFiberWorkLoop.js:3754
flushLayoutEffects();  // ← 阻塞浏览器渲染

// useEffect：下一个宏任务异步执行
// 源码：ReactFiberWorkLoop.js:3641-3655
scheduleCallback(NormalSchedulerPriority, () => {
  flushPassiveEffects();  // ← 不阻塞浏览器渲染
});
```

**使用建议**：

```javascript
// ❌ 避免在 useLayoutEffect 中执行昂贵操作
useLayoutEffect(() => {
  heavyCalculation();  // 会阻塞浏览器渲染，导致卡顿
}, []);

// ✅ 使用 useEffect
useEffect(() => {
  heavyCalculation();  // 异步执行，不阻塞渲染
}, []);

// ✅ useLayoutEffect 适用场景
useLayoutEffect(() => {
  // 需要在浏览器绘制前同步读取/修改 DOM
  const height = ref.current.offsetHeight;
  ref.current.style.top = `${height}px`;
}, []);
```

---

### 5️⃣ 浏览器渲染判断

微任务清空后，浏览器决定是否渲染：

```
检查渲染时机
    ↓
是否到达刷新率周期（约 16.6ms）？
    ↓
是否有 DOM 变化需要绘制？
    ↓
如果需要渲染：
    ├─ 执行 requestAnimationFrame 回调
    ├─ 样式计算（Recalculate Style）
    ├─ 布局计算（Layout）
    ├─ 绘制（Paint）
    └─ 合成（Composite Layers）
    ↓
显示到屏幕
```

**重要**：浏览器渲染与 React 更新是**独立的**！

---

### 6️⃣ 下一轮事件循环

```
下一个宏任务：
    - MessageChannel 回调（React 下一个时间切片）
    - useEffect 回调执行
    - 用户交互事件
    - 定时器回调
    - 网络请求响应
    ↓
循环继续...
```

---

## 📊 完整时间线示例

```
时间轴：0ms ───────────────────────────────────────→ 50ms

0ms     ─→ 【宏任务】用户点击按钮
           ↓ onClick 事件处理器
           ↓ setState() 注册更新

1ms     ─→ 【微任务队列清空】

2ms     ─→ 【宏任务】Render 时间切片 #1
           ↓ 构建 Fiber 树（约 5ms）

7ms     ─→ shouldYield() = true
           ↓ MessageChannel.postMessage()

        ─→ 【微任务队列清空】

        ─→ 浏览器可以：
           - 处理用户输入
           - 执行动画

10ms    ─→ 【宏任务】Render 时间切片 #2
           ↓ 继续构建 Fiber 树（约 5ms）

15ms    ─→ Render 阶段完成
           ↓ 开始 Commit 阶段（同步，不可中断）

        ─→ Before Mutation 阶段
           ↓ getSnapshotBeforeUpdate

        ─→ Mutation 阶段
           ↓ 更新真实 DOM

16ms    ─→ Layout 阶段
           ↓ useLayoutEffect 同步执行

17ms    ─→ Commit 阶段结束
           ↓ 调度 useEffect（注册到下一个宏任务）

        ─→ 【微任务队列清空】

18ms    ─→ 【浏览器渲染】
           ├─ requestAnimationFrame
           ├─ 样式计算
           ├─ 布局
           ├─ 绘制
           └─ 合成

33ms    ─→ 【宏任务】执行 useEffect 回调
           ↓ flushPassiveEffects()

        ... 继续下一轮循环
```

---

## ⚠️ Commit 阶段超过 1 帧的影响

### 问题描述

Commit 阶段是**同步执行**，无法中断。如果超过 16.6ms（1 帧）：

```
0ms    ─→ Commit 开始
       ↓ flushMutationEffects (更新 1000 个 DOM 节点)
       ↓ flushLayoutEffects (执行 useLayoutEffect)
       ↓
50ms   ─→ Commit 结束 ← 超过 3 帧！
       ↓
       浏览器渲染 ← 本应在 16.6ms 时渲染

结果：
✗ 掉了 3 帧
✗ 主线程阻塞 50ms
✗ 无法响应用户输入
```

### 性能影响

| Commit 时长 | 帧率 | 用户感受 | 后果 |
|------------|------|---------|------|
| < 16.6ms | 60 FPS | 流畅 ✅ | 无影响 |
| 16.6-33ms | 30 FPS | 可感知卡顿 ⚠️ | 轻微掉帧 |
| 33-50ms | 20 FPS | 明显卡顿 ❌ | 明显掉帧 |
| 50-100ms | 10 FPS | 严重卡顿 🔥 | 应用几乎冻结 |
| > 100ms | < 10 FPS | 应用无响应 💥 | 用户认为崩溃 |

### 源码证据

React 团队明确知道这个问题：

```javascript
// 源码：packages/react-reconciler/src/ReactFiberWorkLoop.js:506
// The most recent time we either committed a fallback, or when a fallback was
// filled in with the resolved UI. This lets us throttle the appearance of new
// content as it streams in, to minimize jank.
//                                           ↑
//                                    最小化"卡顿"
let globalMostRecentFallbackTime: number = 0;
```

```javascript
// 源码：packages/react-reconciler/src/ReactFiberWorkLoop.js:2975-2981
function workLoopConcurrent(nonIdle: boolean) {
  // We yield every other "frame" when rendering Transition or Retries.
  // ...intentionally block any frequently occurring other main
  // thread work like animations from starving our work. In other words,
  // the purpose of this is to reduce the framerate of animations to 30 FPS.
  //                                                                   ↑
  //                                            刻意降低帧率以减少卡顿
}
```

### 应对策略

#### 1. 减少单次更新的 DOM 数量

```javascript
// ❌ 一次性更新 10000 个节点
<div>
  {hugeArray.map(item => <Item key={item.id} {...item} />)}
</div>

// ✅ 使用虚拟化
import { useVirtualizer } from '@tanstack/react-virtual';

function VirtualList() {
  const virtualizer = useVirtualizer({
    count: hugeArray.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 50,
  });

  return (
    <div ref={parentRef}>
      {virtualizer.getVirtualItems().map(virtualRow => (
        <Item key={virtualRow.key} {...hugeArray[virtualRow.index]} />
      ))}
    </div>
  );
}
```

#### 2. 使用 Transition API 降低优先级

```javascript
import { useTransition } from 'react';

function SearchResults() {
  const [isPending, startTransition] = useTransition();
  const [results, setResults] = useState([]);

  const handleSearch = (query) => {
    // 低优先级更新，不会阻塞紧急更新（如用户输入）
    startTransition(() => {
      const filtered = expensiveSearch(query);
      setResults(filtered);
    });
  };

  return (
    <div>
      <input onChange={e => handleSearch(e.target.value)} />
      {isPending ? <Spinner /> : <Results data={results} />}
    </div>
  );
}
```

#### 3. 避免在 useLayoutEffect 中执行昂贵操作

```javascript
// ❌ 阻塞 Commit 阶段
useLayoutEffect(() => {
  // 同步执行，阻塞浏览器渲染
  for (let i = 0; i < 1000000; i++) {
    heavyCalculation();
  }
}, []);

// ✅ 异步执行
useEffect(() => {
  // 在下一个宏任务执行，不阻塞渲染
  for (let i = 0; i < 1000000; i++) {
    heavyCalculation();
  }
}, []);
```

#### 4. 使用 React DevTools Profiler

```bash
# 步骤
1. 打开 React DevTools
2. 切换到 Profiler 标签
3. 点击录制按钮
4. 执行操作
5. 停止录制
6. 查看 Flamegraph 和 Ranked Chart

# 关注指标
- Commit duration（Commit 持续时间）
- Render duration（Render 持续时间）
- 找出耗时最长的组件
```

#### 5. 性能监控

```javascript
// 监控长任务（Long Task）
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (entry.duration > 16.6) {
      console.warn(`⚠️ Long task detected: ${entry.duration.toFixed(2)}ms`);
      // 上报到监控系统
      reportToAnalytics({
        type: 'long-task',
        duration: entry.duration,
        startTime: entry.startTime,
      });
    }
  }
});
observer.observe({ entryTypes: ['longtask'] });

// 监控首次输入延迟（FID）
const observer2 = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    console.log(`FID: ${entry.processingStart - entry.startTime}ms`);
  }
});
observer2.observe({ entryTypes: ['first-input'] });
```

---

## 🎯 关键要点总结

| 项目 | 说明 | 源码位置 |
|------|------|---------|
| **时间切片** | 5ms | `SchedulerFeatureFlags.js:11` |
| **调度方式** | MessageChannel | `Scheduler.js:532-540` |
| **让出控制权** | `shouldYield()` | `Scheduler.js:447-460` |
| **Render 阶段** | 可中断，时间切片 | `ReactFiberWorkLoop.js:2992-2998` |
| **Commit 阶段** | 不可中断，同步执行 | `ReactFiberWorkLoop.js:3706-3757` |
| **useLayoutEffect** | Commit 阶段同步 | `ReactFiberWorkLoop.js:3754` |
| **useEffect** | 下一个宏任务异步 | `ReactFiberWorkLoop.js:3641-3655` |
| **浏览器渲染** | 约 16.6ms/帧（60Hz） | 浏览器原生机制 |

---

## 🔍 常见误解澄清

### ❌ 误解 1：React 使用微任务调度

**真相**：React 使用 **MessageChannel（宏任务）** 调度，不是微任务。

```javascript
// 源码证据：packages/scheduler/src/forks/Scheduler.js:532-540
if (typeof MessageChannel !== 'undefined') {
  const channel = new MessageChannel();
  channel.port1.onmessage = performWorkUntilDeadline;
  schedulePerformWorkUntilDeadline = () => {
    port.postMessage(null);  // ← 创建宏任务
  };
}
```

【微任务执行】processRootScheduleInMicrotask()
      ↓
      ├─ 同步更新（SyncLane）
      │  └─ 在微任务中立即执行 render + commit
      │
      └─ 并发更新（其他 Lanes）
         └─ 通过 Scheduler.scheduleCallback
            └─ 使用 MessageChannel 调度到宏任务
               └─ 时间切片执行 render

### ❌ 误解 2：useEffect 在 Commit 阶段执行

**真相**：`useEffect` 在 **Commit 之后的下一个宏任务** 执行。

```javascript
// Commit 阶段结束后
scheduleCallback(NormalSchedulerPriority, () => {
  flushPassiveEffects();  // ← useEffect 在这里执行
});
```

### ❌ 误解 3：时间切片是 16.6ms

**真相**：React 时间切片是 **5ms**，浏览器渲染周期是 16.6ms。

```javascript
// 源码：packages/scheduler/src/SchedulerFeatureFlags.js:11
export const frameYieldMs = 5;  // ← 5ms，不是 16.6ms
```

### ❌ 误解 4：Commit 阶段可以中断

**真相**：Commit 阶段 **完全同步，不可中断**。

```javascript
// 源码：ReactFiberWorkLoop.js:3752
// Flush synchronously.  ← 明确说明是同步的
flushMutationEffects();
flushLayoutEffects();
```

---

## 💡 最佳实践

### 1. 性能优化清单

```javascript
// ✅ 使用 memo 避免不必要的重渲染
const MemoizedComponent = React.memo(ExpensiveComponent);

// ✅ 使用 useMemo 缓存计算结果
const expensiveResult = useMemo(() => {
  return computeExpensiveValue(a, b);
}, [a, b]);

// ✅ 使用 useCallback 缓存函数
const memoizedCallback = useCallback(() => {
  doSomething(a, b);
}, [a, b]);

// ✅ 使用 Transition 降低非紧急更新的优先级
startTransition(() => {
  setLowPriorityState(newValue);
});

// ✅ 使用虚拟化处理长列表
import { FixedSizeList } from 'react-window';

// ✅ 懒加载组件
const LazyComponent = React.lazy(() => import('./Component'));
```

### 2. 避免常见陷阱

```javascript
// ❌ 在 render 中创建新对象/数组
function Component() {
  return <Child data={{ value: 1 }} />;  // 每次渲染创建新对象
}

// ✅ 使用 useMemo
function Component() {
  const data = useMemo(() => ({ value: 1 }), []);
  return <Child data={data} />;
}

// ❌ 在 useLayoutEffect 中执行异步操作
useLayoutEffect(() => {
  fetchData();  // 阻塞渲染
}, []);

// ✅ 使用 useEffect
useEffect(() => {
  fetchData();  // 不阻塞渲染
}, []);

// ❌ 过度使用 Context 导致大量重渲染
<Context.Provider value={{ user, theme, config }}>
  {children}
</Context.Provider>

// ✅ 拆分 Context
<UserContext.Provider value={user}>
  <ThemeContext.Provider value={theme}>
    <ConfigContext.Provider value={config}>
      {children}
    </ConfigContext.Provider>
  </ThemeContext.Provider>
</UserContext.Provider>
```

---

## 📚 相关源码文件

| 文件路径 | 作用 |
|---------|------|
| `packages/scheduler/src/forks/Scheduler.js` | Scheduler 调度器核心实现 |
| `packages/scheduler/src/SchedulerFeatureFlags.js` | 时间切片配置（5ms） |
| `packages/react-reconciler/src/ReactFiberWorkLoop.js` | Fiber 工作循环、Render 和 Commit 阶段 |
| `packages/react-reconciler/src/ReactFiberHooks.js` | Hooks 实现（包括 useEffect、useLayoutEffect） |
| `packages/react-reconciler/src/ReactFiberCommitWork.js` | Commit 阶段的具体实现 |

---

## 🎓 总结

### React 事件循环的设计哲学

1. **Render 阶段可中断** → 通过时间切片保持应用响应
2. **Commit 阶段不可中断** → 保证 DOM 状态一致性
3. **useEffect 异步执行** → 不阻塞浏览器渲染
4. **useLayoutEffect 同步执行** → 允许在绘制前同步读取/修改 DOM

### 为什么这样设计？

```
Render 阶段（可中断）
    ↓
    目的：计算新的 UI 状态
    特点：纯计算，无副作用
    策略：可以随时暂停和恢复

Commit 阶段（不可中断）
    ↓
    目的：将变化应用到 DOM
    特点：有副作用，操作真实 DOM
    策略：必须一次性完成，保证一致性
```

### 最终目标

- ✅ 保持应用流畅（60 FPS）
- ✅ 快速响应用户输入
- ✅ 保证 UI 一致性
- ✅ 高效批量更新

---

**🎉 恭喜！你已经完全理解 React 的事件循环机制！**

---

*本文档基于 React 源码分析，所有结论均有源码支持。*
*最后更新时间：2025-11-07*
