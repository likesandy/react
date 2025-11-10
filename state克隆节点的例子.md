// 所有更新都是高优先级
setCount(1); // SyncLane
setCount(2); // SyncLane
setCount(3); // SyncLane

// 渲染时 (renderLanes = SyncLane):
// update1(1): 处理，newState = 1，不克隆 ✓
// update2(2): 处理，newState = 2，不克隆 ✓
// update3(3): 处理，newState = 3，不克隆 ✓
// 
// 结果:
// memoizedState = 3
// baseState = 3
// baseQueue = null  ← 没有克隆任何更新


<!-- 需要克隆 -->
startTransition(() => {
    setCount(1); // TransitionLane
  });
  setCount(2);   // SyncLane
  setCount(3);   // SyncLane

// 第一次渲染 (renderLanes = SyncLane):
  // update1(1, Trans): 跳过 ← newBaseQueueLast 被设置了！
  //   → 克隆到 newBaseQueue
  //   → newBaseState = 0
  // 
  // update2(2, Sync): 处理
  //   → 检查: newBaseQueueLast !== null ✓
  //   → 克隆 update2! (即使要处理它)
  //   → newState = 2
  // 
  // update3(3, Sync): 处理
  //   → 检查: newBaseQueueLast !== null ✓
  //   → 克隆 update3! (即使要处理它)
  //   → newState = 3
  //
  // 结果:
  // memoizedState = 3
  // baseState = 0
  // baseQueue = [update1克隆, update2克隆, update3克隆]



  // 第二次渲染 (renderLanes = TransitionLane):
  // 从 baseState = 0 开始
  // 
  // update1克隆(1): 处理
  //   → newState = 1
  // 
  // update2克隆(2): 处理
  //   → newState = 2
  // 
  // update3克隆(3): 处理
  //   → newState = 3
  //
  // 最终 memoizedState = 3 ✓ 与第一次渲染结果一致


<!-- 乐观更新 -->

