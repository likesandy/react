完整的更新流程

  阶段 1: 挂载时建立订阅连接

  // 1. 组件首次渲染，调用 useSyncExternalStore
  function TodoList() {
    const todos = useSyncExternalStore(
      todosStore.subscribe,    // ← subscribe 函数
      todosStore.getSnapshot   // ← getSnapshot 函数
    );
    // ...
  }

  // 2. React 内部调用 mountSyncExternalStore (ReactFiberHooks.js:1634)
  function mountSyncExternalStore(subscribe, getSnapshot, getServerSnapshot) {
    const fiber = currentlyRenderingFiber;
    const hook = mountWorkInProgressHook();

    // 3. 获取初始快照
    const nextSnapshot = getSnapshot();  // ← todos = [{ id: 0, text: 'Todo #1' }]

    // 4. 创建实例对象
    const inst: StoreInstance<T> = {
      value: nextSnapshot,     // 保存当前快照
      getSnapshot,             // 保存获取器
    };
    hook.queue = inst;

    // 5. 🔑 关键：注册订阅 effect
    mountEffect(subscribeToStore.bind(null, fiber, inst, subscribe), [subscribe]);
    //           ↑ 这个 effect 会在提交后执行

    return nextSnapshot;
  }

  阶段 2: Effect 执行，建立订阅

  // 6. 提交阶段后，执行 subscribeToStore (ReactFiberHooks.js:1871)
  function subscribeToStore(fiber, inst, subscribe) {
    // 7. 创建变化处理函数
    const handleStoreChange = () => {
      // 9. 当外部 store 变化时，这个函数会被调用 ↓
      if (checkIfSnapshotChanged(inst)) {
        startUpdateTimerByLane(SyncLane, 'updateSyncExternalStore()', fiber);
        forceStoreRerender(fiber);  // ← 触发重新渲染
      }
    };

    // 8. 🔑 调用用户的 subscribe，注册 handleStoreChange
    return subscribe(handleStoreChange);
    //      ↓
    //      调用 todosStore.subscribe(handleStoreChange)
    //      ↓
    //      listeners = [...listeners, handleStoreChange]
    //      ↓
    //      返回取消订阅函数
  }

  此时，listeners 数组中包含了 React 的 handleStoreChange 函数！

  阶段 3: 用户触发更新

  // 10. 用户点击按钮，调用 addTodo
  todosStore.addTodo();

  // 11. 外部 store 更新数据
  export const todosStore = {
    addTodo() {
      todos = [...todos, { id: nextId++, text: 'Todo #' + nextId }];
      //      ↑ 数据已经变了！

      emitChange();  // 12. 触发所有监听器
    }
  }

  // 13. emitChange 遍历所有 listeners
  function emitChange() {
    for (let listener of listeners) {
      listener();  // ← 调用 React 注册的 handleStoreChange
    }
  }

  阶段 4: React 检测变化并触发渲染

  // 14. handleStoreChange 被调用
  const handleStoreChange = () => {
    // 15. 检查快照是否变化
    if (checkIfSnapshotChanged(inst)) {
      forceStoreRerender(fiber);
    }
  };

  // 16. checkIfSnapshotChanged 实现 (ReactFiberHooks.js:1889)
  function checkIfSnapshotChanged(inst) {
    const latestGetSnapshot = inst.getSnapshot;
    const prevValue = inst.value;  // 旧值: [{ id: 0, text: 'Todo #1' }]

    try {
      const nextValue = latestGetSnapshot();  // 新值: [{ id: 0, ... }, { id: 1, ... }]
      return !is(prevValue, nextValue);  // ← 比较，返回 true
    } catch (error) {
      return true;
    }
  }

  // 17. 触发强制重新渲染 (ReactFiberHooks.js:1900)
  function forceStoreRerender(fiber) {
    const root = enqueueConcurrentRenderForLane(fiber, SyncLane);
    if (root !== null) {
      scheduleUpdateOnFiber(root, fiber, SyncLane);  // ← 安排同步更新
    }
  }

  阶段 5: React 重新渲染

  // 18. scheduleUpdateOnFiber 触发调度
  // 19. React 开始新的渲染周期
  // 20. 调用 updateSyncExternalStore
  function updateSyncExternalStore(subscribe, getSnapshot, getServerSnapshot) {
    const fiber = currentlyRenderingFiber;
    const hook = updateWorkInProgressHook();

    // 21. 重新读取快照
    const nextSnapshot = getSnapshot();  // ← 获取最新数据

    // 22. 与上次快照比较
    const prevSnapshot = (currentHook || hook).memoizedState;
    const snapshotChanged = !is(prevSnapshot, nextSnapshot);

    if (snapshotChanged) {
      hook.memoizedState = nextSnapshot;
      markWorkInProgressReceivedUpdate();  // ← 标记需要更新
    }

    // 23. 更新订阅（如果 subscribe 变了）
    updateEffectImpl(
      PassiveEffect,
      HookPassive,
      subscribeToStore.bind(null, fiber, inst, subscribe),
      [subscribe],
    );

    return nextSnapshot;  // ← 返回新数据给组件
  }

  // 24. 组件重新渲染，显示新的 todos

  关键点总结

  🔑 为什么 subscribe 能触发更新？

  1. mountSyncExternalStore 通过 mountEffect 注册订阅
     ↓
  2. subscribeToStore 将 handleStoreChange 传给用户的 subscribe
     ↓
  3. 用户的 subscribe 将 handleStoreChange 添加到 listeners 数组
     ↓
  4. 当 store 变化时，emitChange() 调用所有 listeners
     ↓
  5. handleStoreChange 被触发
     ↓
  6. checkIfSnapshotChanged 检测快照是否变化
     ↓
  7. forceStoreRerender 触发同步重新渲染
     ↓
  8. React 重新调用 updateSyncExternalStore 获取最新数据

  🎯 核心机制

  1. 订阅注册 (ReactFiberHooks.js:1707)
  mountEffect(subscribeToStore.bind(null, fiber, inst, subscribe), [subscribe]);
  2. 变化回调 (ReactFiberHooks.js:1876-1886)
  const handleStoreChange = () => {
    if (checkIfSnapshotChanged(inst)) {
      forceStoreRerender(fiber);
    }
  };
  return subscribe(handleStoreChange);  // ← 用户的 subscribe 保存这个回调
  3. 强制渲染 (ReactFiberHooks.js:1900-1905)
  function forceStoreRerender(fiber) {
    const root = enqueueConcurrentRenderForLane(fiber, SyncLane);
    if (root !== null) {
      scheduleUpdateOnFiber(root, fiber, SyncLane);  // ← 同步更新
    }
  }

  💡 设计亮点

  1. 用户只需提供 subscribe 和 getSnapshot，React 自动处理订阅和取消订阅
  2. 同步更新（SyncLane），确保外部 store 变化立即反映到 UI
  3. 快照比较，只在数据真正变化时触发更新
  4. 自动清理，effect 返回的取消订阅函数会在组件卸载时自动调用

  这就是 useSyncExternalStore 的完整工作原理！外部 store 通过调用注册的回调函数，触发
  React 的调度系统，从而实现响应式更新。


并发渲染(一致性检查)

  特点：
  - 低优先级（Transition、Retry、Idle 等）
  - 可以被中断和暂停
  - 渲染可能持续很长时间
  - 在渲染期间，外部 store 可能已经更新多次

  问题：可能出现 tearing（撕裂）
  时间线：
  T1: 组件 A 读取 store，值为 1
  T2: 渲染被中断，外部 store 更新为 2
  T3: 组件 B 读取 store，值为 2
  T4: 提交 - UI 显示 A=1, B=2 ❌ 不一致！
