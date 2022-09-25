# React源码学习进阶（四）render流程的入口逻辑详解

> 本文采用React v16.13.1版本源码进行分析

### 源码解析

接下来我们分析一下`updateContainer`的逻辑，它的入口在`packages/react-reconciler/src/ReactFiberReconciler.js`中：

```js
export function updateContainer(
  element: ReactNodeList,
  container: OpaqueRoot,
  parentComponent: ?React$Component<any, any>,
  callback: ?Function,
): ExpirationTime {
  
  const current = container.current;
  const currentTime = requestCurrentTimeForUpdate();

  const suspenseConfig = requestCurrentSuspenseConfig();
  const expirationTime = computeExpirationForFiber(
    currentTime,
    current,
    suspenseConfig,
  );

  const context = getContextForSubtree(parentComponent);
  if (container.context === null) {
    container.context = context;
  } else {
    container.pendingContext = context;
  }

  const update = createUpdate(expirationTime, suspenseConfig);
  // Caution: React DevTools currently depends on this property
  // being called "element".
  update.payload = {element};

  callback = callback === undefined ? null : callback;
  if (callback !== null) {
    if (__DEV__) {
      if (typeof callback !== 'function') {
        console.error(
          'render(...): Expected the last optional `callback` argument to be a ' +
            'function. Instead received: %s.',
          callback,
        );
      }
    }
    update.callback = callback;
  }

  enqueueUpdate(current, update);
  scheduleWork(current, expirationTime);

  return expirationTime;
}
```

这段代码里面有几个非常关键的信息：

* 通过`computeExpirationForFiber`计算出的`expirationTime`，这个涉及到调度优先级，我们暂时可以认为它就是Sync的常量值。
* 调用`createUpdate`创建了一个`update`，并调用`enqueueUpdate`将`update`入队。
* 调用`scheduleWork`启动整个更新流程。

接下来先看一下`createUpdate`的逻辑，源码位于`packages/react-reconciler/src/ReactUpdateQueue.js`：

```js
export function createUpdate(
  expirationTime: ExpirationTime,
  suspenseConfig: null | SuspenseConfig,
): Update<*> {
  let update: Update<*> = {
    expirationTime,
    suspenseConfig,

    tag: UpdateState,
    payload: null,
    callback: null,

    next: (null: any),
  };
  update.next = update;
  return update;
}
```

可以看到，`update`其实就是一个环状链表的结构，自己指向自己：

![image-20220915125958637](https://tva1.sinaimg.cn/large/e6c9d24egy1h677cp304qj20dy09o3yr.jpg)

然后通过`enqueueUpdate`方法将`update`放入之前`fiber`的`updateQueue`里：

```js
export function enqueueUpdate<State>(fiber: Fiber, update: Update<State>) {
  const updateQueue = fiber.updateQueue;
  if (updateQueue === null) {
    // Only occurs if the fiber has been unmounted.
    return;
  }

  const sharedQueue = updateQueue.shared;
  const pending = sharedQueue.pending;
  if (pending === null) {
    // This is the first update. Create a circular list.
    update.next = update;
  } else {
    update.next = pending.next;
    pending.next = update;
  }
  sharedQueue.pending = update;
}
```

实际上此时的`update`被存储在`updateQueue`的`shared.pending`字段中。

#### scheduleWork启动

`scheduleWork`的启动代码位于`packages/react-reconciler/src/ReactFiberWorkLoop.js`中：

```js
export function scheduleUpdateOnFiber(
  fiber: Fiber,
  expirationTime: ExpirationTime,
) {
  checkForNestedUpdates();
  warnAboutRenderPhaseUpdatesInDEV(fiber);

  const root = markUpdateTimeFromFiberToRoot(fiber, expirationTime);
  if (root === null) {
    warnAboutUpdateOnUnmountedFiberInDEV(fiber);
    return;
  }

  checkForInterruption(fiber, expirationTime);
  recordScheduleUpdate();

  // TODO: computeExpirationForFiber also reads the priority. Pass the
  // priority as an argument to that function and this one.
  const priorityLevel = getCurrentPriorityLevel();

  if (expirationTime === Sync) {
    if (
      // Check if we're inside unbatchedUpdates
      (executionContext & LegacyUnbatchedContext) !== NoContext &&
      // Check if we're not already rendering
      (executionContext & (RenderContext | CommitContext)) === NoContext
    ) {
      // Register pending interactions on the root to avoid losing traced interaction data.
      schedulePendingInteractions(root, expirationTime);

      // This is a legacy edge case. The initial mount of a ReactDOM.render-ed
      // root inside of batchedUpdates should be synchronous, but layout updates
      // should be deferred until the end of the batch.
      performSyncWorkOnRoot(root);
    } else {
      ensureRootIsScheduled(root);
      schedulePendingInteractions(root, expirationTime);
      if (executionContext === NoContext) {
        // Flush the synchronous work now, unless we're already working or inside
        // a batch. This is intentionally inside scheduleUpdateOnFiber instead of
        // scheduleCallbackForFiber to preserve the ability to schedule a callback
        // without immediately flushing it. We only do this for user-initiated
        // updates, to preserve historical behavior of legacy mode.
        flushSyncCallbackQueue();
      }
    }
  } else {
    ensureRootIsScheduled(root);
    schedulePendingInteractions(root, expirationTime);
  }

  if (
    (executionContext & DiscreteEventContext) !== NoContext &&
    // Only updates at user-blocking priority or greater are considered
    // discrete, even inside a discrete event.
    (priorityLevel === UserBlockingPriority ||
      priorityLevel === ImmediatePriority)
  ) {
    // This is the result of a discrete event. Track the lowest priority
    // discrete update per root so we can flush them early, if needed.
    if (rootsWithPendingDiscreteUpdates === null) {
      rootsWithPendingDiscreteUpdates = new Map([[root, expirationTime]]);
    } else {
      const lastDiscreteTime = rootsWithPendingDiscreteUpdates.get(root);
      if (lastDiscreteTime === undefined || lastDiscreteTime > expirationTime) {
        rootsWithPendingDiscreteUpdates.set(root, expirationTime);
      }
    }
  }
}
```

这段代码实际上核心关注两个点：

* 调用`markUpdateTimeFromFiberToRoot`获取`root`，在这个过程中会更新`Fiber`的`expirationTime`。
* 调用`performSyncWorkOnRoot`，开始执行work的逻辑。

首先看一下`markUpdateTimeFromFiberToRoot`的逻辑：

```js
function markUpdateTimeFromFiberToRoot(fiber, expirationTime) {
  // Update the source fiber's expiration time
  if (fiber.expirationTime < expirationTime) {
    fiber.expirationTime = expirationTime;
  }
  let alternate = fiber.alternate;
  if (alternate !== null && alternate.expirationTime < expirationTime) {
    alternate.expirationTime = expirationTime;
  }
  // Walk the parent path to the root and update the child expiration time.
  let node = fiber.return;
  let root = null;
  if (node === null && fiber.tag === HostRoot) {
    root = fiber.stateNode;
  } else {
    while (node !== null) {
      alternate = node.alternate;
      if (node.childExpirationTime < expirationTime) {
        node.childExpirationTime = expirationTime;
        if (
          alternate !== null &&
          alternate.childExpirationTime < expirationTime
        ) {
          alternate.childExpirationTime = expirationTime;
        }
      } else if (
        alternate !== null &&
        alternate.childExpirationTime < expirationTime
      ) {
        alternate.childExpirationTime = expirationTime;
      }
      if (node.return === null && node.tag === HostRoot) {
        root = node.stateNode;
        break;
      }
      node = node.return;
    }
  }

  if (root !== null) {
    if (workInProgressRoot === root) {
      // Received an update to a tree that's in the middle of rendering. Mark
      // that's unprocessed work on this root.
      markUnprocessedUpdateTime(expirationTime);

      if (workInProgressRootExitStatus === RootSuspendedWithDelay) {
        // The root already suspended with a delay, which means this render
        // definitely won't finish. Since we have a new update, let's mark it as
        // suspended now, right before marking the incoming update. This has the
        // effect of interrupting the current render and switching to the update.
        // TODO: This happens to work when receiving an update during the render
        // phase, because of the trick inside computeExpirationForFiber to
        // subtract 1 from `renderExpirationTime` to move it into a
        // separate bucket. But we should probably model it with an exception,
        // using the same mechanism we use to force hydration of a subtree.
        // TODO: This does not account for low pri updates that were already
        // scheduled before the root started rendering. Need to track the next
        // pending expiration time (perhaps by backtracking the return path) and
        // then trigger a restart in the `renderDidSuspendDelayIfPossible` path.
        markRootSuspendedAtTime(root, renderExpirationTime);
      }
    }
    // Mark that the root has a pending update.
    markRootUpdatedAtTime(root, expirationTime);
  }

  return root;
}
```

这里其实在`root`挂载时，只更新了当前的`rootFiber`，以及调用`markRootUpdatedAtTime`更新了root的`firstPendingTime`：

```js
export function markRootUpdatedAtTime(
  root: FiberRoot,
  expirationTime: ExpirationTime,
): void {
  // Update the range of pending times
  const firstPendingTime = root.firstPendingTime;
  if (expirationTime > firstPendingTime) {
    root.firstPendingTime = expirationTime;
  }

  // Update the range of suspended times. Treat everything lower priority or
  // equal to this update as unsuspended.
  const firstSuspendedTime = root.firstSuspendedTime;
  if (firstSuspendedTime !== NoWork) {
    if (expirationTime >= firstSuspendedTime) {
      // The entire suspended range is now unsuspended.
      root.firstSuspendedTime = root.lastSuspendedTime = root.nextKnownPendingLevel = NoWork;
    } else if (expirationTime >= root.lastSuspendedTime) {
      root.lastSuspendedTime = expirationTime + 1;
    }

    // This is a pending level. Check if it's higher priority than the next
    // known pending level.
    if (expirationTime > root.nextKnownPendingLevel) {
      root.nextKnownPendingLevel = expirationTime;
    }
  }
}
```

接下来就是执行`performSyncWorkOnRoot`的逻辑了：

```js
function performSyncWorkOnRoot(root) {
  // Check if there's expired work on this root. Otherwise, render at Sync.
  const lastExpiredTime = root.lastExpiredTime;
  const expirationTime = lastExpiredTime !== NoWork ? lastExpiredTime : Sync;
  invariant(
    (executionContext & (RenderContext | CommitContext)) === NoContext,
    'Should not already be working.',
  );

  flushPassiveEffects();

  // If the root or expiration time have changed, throw out the existing stack
  // and prepare a fresh one. Otherwise we'll continue where we left off.
  if (root !== workInProgressRoot || expirationTime !== renderExpirationTime) {
    prepareFreshStack(root, expirationTime);
    startWorkOnPendingInteractions(root, expirationTime);
  }

  // If we have a work-in-progress fiber, it means there's still work to do
  // in this root.
  if (workInProgress !== null) {
    const prevExecutionContext = executionContext;
    executionContext |= RenderContext;
    const prevDispatcher = pushDispatcher(root);
    const prevInteractions = pushInteractions(root);
    startWorkLoopTimer(workInProgress);

    do {
      try {
        workLoopSync();
        break;
      } catch (thrownValue) {
        handleError(root, thrownValue);
      }
    } while (true);
    resetContextDependencies();
    executionContext = prevExecutionContext;
    popDispatcher(prevDispatcher);
    if (enableSchedulerTracing) {
      popInteractions(((prevInteractions: any): Set<Interaction>));
    }

    if (workInProgressRootExitStatus === RootFatalErrored) {
      const fatalError = workInProgressRootFatalError;
      stopInterruptedWorkLoopTimer();
      prepareFreshStack(root, expirationTime);
      markRootSuspendedAtTime(root, expirationTime);
      ensureRootIsScheduled(root);
      throw fatalError;
    }

    if (workInProgress !== null) {
      // This is a sync render, so we should have finished the whole tree.
      invariant(
        false,
        'Cannot commit an incomplete root. This error is likely caused by a ' +
          'bug in React. Please file an issue.',
      );
    } else {
      // We now have a consistent tree. Because this is a sync render, we
      // will commit it even if something suspended.
      stopFinishedWorkLoopTimer();
      root.finishedWork = (root.current.alternate: any);
      root.finishedExpirationTime = expirationTime;
      finishSyncRender(root);
    }

    // Before exiting, make sure there's a callback scheduled for the next
    // pending level.
    ensureRootIsScheduled(root);
  }

  return null;
}
```

这个函数实际上才是真正的挂载更新入口，排除掉其他干扰，核心逻辑有三点：

* 调用`prepareFreshStack`，初始化`workInProgress`的信息
* 调用`workLoopSync`，启动`render`流程
* 调用`finishSyncRender`，启动`commit`流程

首先我们看一下`prepareFreshStack`做了什么：

```js
function prepareFreshStack(root, expirationTime) {
  root.finishedWork = null;
  root.finishedExpirationTime = NoWork;

  const timeoutHandle = root.timeoutHandle;
  if (timeoutHandle !== noTimeout) {
    // The root previous suspended and scheduled a timeout to commit a fallback
    // state. Now that we have additional work, cancel the timeout.
    root.timeoutHandle = noTimeout;
    // $FlowFixMe Complains noTimeout is not a TimeoutID, despite the check above
    cancelTimeout(timeoutHandle);
  }

  if (workInProgress !== null) {
    let interruptedWork = workInProgress.return;
    while (interruptedWork !== null) {
      unwindInterruptedWork(interruptedWork);
      interruptedWork = interruptedWork.return;
    }
  }
  workInProgressRoot = root;
  workInProgress = createWorkInProgress(root.current, null);
  renderExpirationTime = expirationTime;
  workInProgressRootExitStatus = RootIncomplete;
  workInProgressRootFatalError = null;
  workInProgressRootLatestProcessedExpirationTime = Sync;
  workInProgressRootLatestSuspenseTimeout = Sync;
  workInProgressRootCanSuspendUsingConfig = null;
  workInProgressRootNextUnprocessedUpdateTime = NoWork;
  workInProgressRootHasPendingPing = false;

  if (enableSchedulerTracing) {
    spawnedWorkDuringRender = null;
  }
}
```

可以看到在这里创建了一个`workInProgress`，它的创建方法位于`packages/react-reconciler/src/ReactFiber.js`：

```js
export function createWorkInProgress(current: Fiber, pendingProps: any): Fiber {
  let workInProgress = current.alternate;
  if (workInProgress === null) {
    // We use a double buffering pooling technique because we know that we'll
    // only ever need at most two versions of a tree. We pool the "other" unused
    // node that we're free to reuse. This is lazily created to avoid allocating
    // extra objects for things that are never updated. It also allow us to
    // reclaim the extra memory if needed.
    workInProgress = createFiber(
      current.tag,
      pendingProps,
      current.key,
      current.mode,
    );
    workInProgress.elementType = current.elementType;
    workInProgress.type = current.type;
    workInProgress.stateNode = current.stateNode;

    workInProgress.alternate = current;
    current.alternate = workInProgress;
  } else {
    workInProgress.pendingProps = pendingProps;

    // We already have an alternate.
    // Reset the effect tag.
    workInProgress.effectTag = NoEffect;

    // The effect list is no longer valid.
    workInProgress.nextEffect = null;
    workInProgress.firstEffect = null;
    workInProgress.lastEffect = null;

    if (enableProfilerTimer) {
      // We intentionally reset, rather than copy, actualDuration & actualStartTime.
      // This prevents time from endlessly accumulating in new commits.
      // This has the downside of resetting values for different priority renders,
      // But works for yielding (the common case) and should support resuming.
      workInProgress.actualDuration = 0;
      workInProgress.actualStartTime = -1;
    }
  }

  workInProgress.childExpirationTime = current.childExpirationTime;
  workInProgress.expirationTime = current.expirationTime;

  workInProgress.child = current.child;
  workInProgress.memoizedProps = current.memoizedProps;
  workInProgress.memoizedState = current.memoizedState;
  workInProgress.updateQueue = current.updateQueue;

  // Clone the dependencies object. This is mutated during the render phase, so
  // it cannot be shared with the current fiber.
  const currentDependencies = current.dependencies;
  workInProgress.dependencies =
    currentDependencies === null
      ? null
      : {
          expirationTime: currentDependencies.expirationTime,
          firstContext: currentDependencies.firstContext,
          responders: currentDependencies.responders,
        };

  // These will be overridden during the parent's reconciliation
  workInProgress.sibling = current.sibling;
  workInProgress.index = current.index;
  workInProgress.ref = current.ref;

  if (enableProfilerTimer) {
    workInProgress.selfBaseDuration = current.selfBaseDuration;
    workInProgress.treeBaseDuration = current.treeBaseDuration;
  }
  
  return workInProgress;
}
```

实际上创建`workInProgress`的过程可以简单理解为将原来的`fiber`拷贝了一份。

接着我们来看看重头戏`workLoopSync`启动`render`流程：

```js
function workLoopSync() {
  // Already timed out, so perform work without checking if we need to yield.
  while (workInProgress !== null) {
    workInProgress = performUnitOfWork(workInProgress);
  }
}
```

这里是我们接触到的第一层循环，实际上这里的`workInProgress`会被不断改变，赋值为`performUnitOfWork`的返回值，而`performUnitOfWork`每次则会返回子节点或兄弟节点，从而开启整个`Fiber`树的遍历迭代流程。

最后我们再来看看`performUnitOfWork`的逻辑：

```js
function performUnitOfWork(unitOfWork: Fiber): Fiber | null {
  // The current, flushed, state of this fiber is the alternate. Ideally
  // nothing should rely on this, but relying on it here means that we don't
  // need an additional field on the work in progress.
  const current = unitOfWork.alternate;

  startWorkTimer(unitOfWork);
  setCurrentDebugFiberInDEV(unitOfWork);

  let next;
  if (enableProfilerTimer && (unitOfWork.mode & ProfileMode) !== NoMode) {
    startProfilerTimer(unitOfWork);
    next = beginWork(current, unitOfWork, renderExpirationTime);
    stopProfilerTimerIfRunningAndRecordDelta(unitOfWork, true);
  } else {
    next = beginWork(current, unitOfWork, renderExpirationTime);
  }

  resetCurrentDebugFiberInDEV();
  unitOfWork.memoizedProps = unitOfWork.pendingProps;
  if (next === null) {
    // If this doesn't spawn new work, complete the current work.
    next = completeUnitOfWork(unitOfWork);
  }

  ReactCurrentOwner.current = null;
  return next;
}
```

这段逻辑的核心在于两个点：

* 调用`beginWork`处理节点，拿到下一个节点
* 如果下一个节点不存在，则调用`completeUnitOfWork`处理节点，并拿到下一个节点

请记住这里的`beginWork`和`completeUnitOfWork`的调用逻辑，实际上它们在遍历过程中扮演着进入节点和离开节点所做的工作，后面我们将围绕这两个核心`Fiber`操作展开详细的讲解。

### 小结一下

整体在进入正式的`render`流程之前，React `Fiber`本身其实做了大量工作，从入口处更能理清整个挂载更新过程的主体脉络：

![image-20220915202823214](https://tva1.sinaimg.cn/large/e6c9d24egy1h67kb9ra99j20u00uujtu.jpg)

在有了这个脉络之后，我们就可以把重点放在三个方向上：

* `beginWork`具体做了什么事情？（实际上它很重要，整个`reconciler`都是这个阶段完成）
* `completeWork`具体做了什么事情？
* `commit`具体做了什么事情？

搞懂了这三个方面，我们就知道在`Fiber`架构下，React是如何进行渲染更新的了。
