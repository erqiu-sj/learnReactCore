# setState

# setState 是同步 or 异步？

**_不同模式下的 React,效果也不同_**

- **_legacy_** 模式： `ReactDOM.render(<App />, rootNode)`。这是当前 React app 使用的方式

- **_blocking_** 模式： `ReactDOM.createBlockingRoot(rootNode).render(<App />)`。目前正在实验中。作为迁移到 concurrent 模式的第一个步骤。

- **_concurrent_** 模式： `ReactDOM.createRoot(rootNode).render(<App />)`。目前在实验中，未来稳定之后，打算作为 React 的默认开发模式。这个模式开启了所有的新功能。**_但我们使用该模式，我们触发的更新就会有不同的优先级,并且更新的过程也是可以打断的_**

```javascript
class App extends Components {
  state = {
    num: 0,
  };
  update = () => {
    const { num } = this.state;
    console.log("before", num);
    this.setState({ num: num + 1 });
    console.log("after", num);
  };
  render() {
    const { num } = this.state;
    console.log("render", num);
    return <div onClick={this.update.bind(this)}>num:{num}</div>;
  }
}
```

点击`div`标签会`log`出

```javascript
before 0
after 0
render 1
// 可以看出这是异步的，react中有一个batchedUpdates(批处理)，当我们触发一个或多个setState，只会触发1次render，react将它们合并，提高性能
```

```javascript
export function batchedUpdates<A, R>(fn: (A) => R, a: A): R {
  const prevExecutionContext = executionContext;
  // 在执行这个函数之前会将一个全局变量executionContext附加上BatchedContext这个flag
  executionContext |= BatchedContext;
  try {
    // 执行该函数，这里的fn也就是update更新的这个函数
    // 这就是说当我们执行update这个函数时，会有一个全局变量executionContext包含了BatchedContext
    // react会判断如果这次更新包含了BatchedContext，他就会认为这是一次批处理，会被合并为一次更新，并且异步执行
    // 如果跳出BatchedContext呢，可以看到如果我们的函数出发了fn(a)异步执行的话，等setState执行时全局的executionContext中已经不包含了BatchedContext,只需要将setState放入定时器即可
    return fn(a);
  } finally {
    // 执行完后会将 executionContext 中的BatchedContext去除
    executionContext = prevExecutionContext;
    if (executionContext === NoContext) {
      // Flush the immediate callbacks that were scheduled during this batch
      resetRenderTimer();
      flushSyncCallbackQueue();
    }
  }
}
```

```javascript
// 当我们setState放入定时器中
update = () => {
  const { num } = this.state;
  console.log("before", num);
  // this.setState({ num: num + 1 });
  // console.log("after", num);
  setTimeOut(() => {
    this.setState({ num: num + 1 });
    console.log("after", num);
  }, 0);
};
// 点击执行后
before 0
render 1
after 1
// 为什么会这样？因为我们加上了setTimeout，此时的setState在全局上下文中已经不存在BatchedContext,但不存在这个flag时,每次调度更新时会调用scheduleUpdateOnFiber
```

```javascript
export function scheduleUpdateOnFiber(
  fiber: Fiber,
  lane: Lane,
  eventTime: number
): FiberRoot | null {
  checkForNestedUpdates();
  warnAboutRenderPhaseUpdatesInDEV(fiber);

  const root = markUpdateLaneFromFiberToRoot(fiber, lane);
  if (root === null) {
    warnAboutUpdateOnUnmountedFiberInDEV(fiber);
    return null;
  }
  // Mark that the root has a pending update.
  markRootUpdated(root, lane, eventTime);

  if (enableProfilerTimer && enableProfilerNestedUpdateScheduledHook) {
    if (
      (executionContext & CommitContext) !== NoContext &&
      root === rootCommittingMutationOrLayoutEffects
    ) {
      if (fiber.mode & ProfileMode) {
        let current = fiber;
        while (current !== null) {
          if (current.tag === Profiler) {
            const { id, onNestedUpdateScheduled } = current.memoizedProps;
            if (typeof onNestedUpdateScheduled === "function") {
              if (enableSchedulerTracing) {
                onNestedUpdateScheduled(id, root.memoizedInteractions);
              } else {
                onNestedUpdateScheduled(id);
              }
            }
          }
          current = current.return;
        }
      }
    }
  }

  if (root === workInProgressRoot) {
    // Received an update to a tree that's in the middle of rendering. Mark
    // that there was an interleaved update work on this root. Unless the
    // `deferRenderPhaseUpdateToNextBatch` flag is off and this is a render
    // phase update. In that case, we don't treat render phase updates as if
    // they were interleaved, for backwards compat reasons.
    if (
      deferRenderPhaseUpdateToNextBatch ||
      (executionContext & RenderContext) === NoContext
    ) {
      workInProgressRootUpdatedLanes = mergeLanes(
        workInProgressRootUpdatedLanes,
        lane
      );
    }
    if (workInProgressRootExitStatus === RootSuspendedWithDelay) {
      // The root already suspended with a delay, which means this render
      // definitely won't finish. Since we have a new update, let's mark it as
      // suspended now, right before marking the incoming update. This has the
      // effect of interrupting the current render and switching to the update.
      // TODO: Make sure this doesn't override pings that happen while we've
      // already started rendering.
      markRootSuspended(root, workInProgressRootRenderLanes);
    }
  }
  // TODO: requestUpdateLanePriority also reads the priority. Pass the
  // priority as an argument to that function and this one.
  const priorityLevel = getCurrentPriorityLevel();

  if (lane === SyncLane) {
    if (
      // Check if we're inside unbatchedUpdates
      (executionContext & LegacyUnbatchedContext) !== NoContext &&
      // Check if we're not already rendering
      // 在这如果我们的上下文什么都没有的话就会同步的执行这次更新，当我们调用包裹在setTimeout中的setState时候，就会进入同步的更新，那么我们就能在定时器中同步的获得更新后的值吗？答案是不一定。
      // 要进入该函数的前提是lane === SyncLane,也就是说当前更新的优先级是同步的优先级，那么什么情况下的优先级是同步的优先级呢？使用React.render 创建的应用都是同步的优先级，而Concurrent模式，我们的更新就拥有了不同的优先级所以不会进入该函数会进入else条件
      // 所以在Concurrent模式下setState被setTimeOut包裹了执行的也是异步更新
      (executionContext & (RenderContext | CommitContext)) === NoContext
    ) {
      // Register pending interactions on the root to avoid losing traced interaction data.
      schedulePendingInteractions(root, lane);

      // This is a legacy edge case. The initial mount of a ReactDOM.render-ed
      // root inside of batchedUpdates should be synchronous, but layout updates
      // should be deferred until the end of the batch.
      performSyncWorkOnRoot(root);
    } else {
      ensureRootIsScheduled(root, eventTime);
      schedulePendingInteractions(root, lane);
      if (executionContext === NoContext) {
        // Flush the synchronous work now, unless we're already working or inside
        // a batch. This is intentionally inside scheduleUpdateOnFiber instead of
        // scheduleCallbackForFiber to preserve the ability to schedule a callback
        // without immediately flushing it. We only do this for user-initiated
        // updates, to preserve historical behavior of legacy mode.
        resetRenderTimer();
        flushSyncCallbackQueue();
      }
    }
  } else {
    // Schedule a discrete update but only if it's not Sync.
    if (
      (executionContext & DiscreteEventContext) !== NoContext &&
      // Only updates at user-blocking priority or greater are considered
      // discrete, even inside a discrete event.
      (priorityLevel === UserBlockingSchedulerPriority ||
        priorityLevel === ImmediateSchedulerPriority)
    ) {
      // This is the result of a discrete event. Track the lowest priority
      // discrete update per root so we can flush them early, if needed.
      if (rootsWithPendingDiscreteUpdates === null) {
        rootsWithPendingDiscreteUpdates = new Set([root]);
      } else {
        rootsWithPendingDiscreteUpdates.add(root);
      }
    }
    // Schedule other updates after in case the callback is sync.
    ensureRootIsScheduled(root, eventTime);
    schedulePendingInteractions(root, lane);
  }
  // We use this when assigning a lane for a transition inside
  // `requestUpdateLane`. We assume it's the same as the root being updated,
  // since in the common case of a single root app it probably is. If it's not
  // the same root, then it's not a huge deal, we just might batch more stuff
  // together more than necessary.
  mostRecentlyUpdatedRoot = root;

  return root;
}
```

```javascript
// 当我们开启Concurrent模式
React.unstable_createRoot(root).render(<App />);
// 没有被setTimeOut包裹的setState打印出来的是,这和legacy模式没有区别
before 0
after 0
render 1
// 当如果被setTimeOut包裹的setState，执行
before 0
after 0
render 1
// 这是为什么呢？go to line
```

# 总结

legacy 模式下命中了 batchedUpdates 时候是异步的。否则则是同步
Concurrent 模式下都是异步
