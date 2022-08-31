# setState是怎么做到异步化的？



> 本文基于React v15.6.2版本介绍，原因请参见新手如何学习React源码

### 源码解析

还记得我们之前在介绍React组件的时候，ReactComponent的实现吗？

让我们来回顾一下（源码位于`src/isomorphic/modern/class/ReactBaseClasses.js`）：

```js
function ReactComponent(props, context, updater) {
  this.props = props;
  this.context = context;
  this.refs = emptyObject;
  this.updater = updater || ReactNoopUpdateQueue;
}

ReactComponent.prototype.isReactComponent = {};

ReactComponent.prototype.setState = function(partialState, callback) {
  invariant(
    typeof partialState === 'object' ||
      typeof partialState === 'function' ||
      partialState == null,
    'setState(...): takes an object of state variables to update or a ' +
      'function which returns an object of state variables.',
  );
  this.updater.enqueueSetState(this, partialState);
  if (callback) {
    this.updater.enqueueCallback(this, callback, 'setState');
  }
};
```

可以看到React在setState入口也非常简单，并没有复杂的逻辑，只是调用了`updater`的`enqueueSetState`方法而已。

这个`updater`会在前面我们提到的挂载流程中注入，实际上是位于`src/renderers/shared/stack/reconciler/ReactUpdateQueue.js`：

```js
enqueueSetState: function(publicInstance, partialState) {
  if (__DEV__) {
    ReactInstrumentation.debugTool.onSetState();
    warning(
      partialState != null,
      'setState(...): You passed an undefined or null state object; ' +
        'instead, use forceUpdate().',
    );
  }

  var internalInstance = getInternalInstanceReadyForUpdate(
    publicInstance,
    'setState',
  );

  if (!internalInstance) {
    return;
  }

  var queue =
    internalInstance._pendingStateQueue ||
    (internalInstance._pendingStateQueue = []);
  queue.push(partialState);

  enqueueUpdate(internalInstance);
},
```

这段代码其实非常简单，就是通过当前React组件找到我们之前创建好的控制类实例，也就是代码里面的`internalInstance`，在它的`_pendingStateQueue`里面把当前要更新的state给push进去，然后调用`enqueueUpdate`方法。

`enqueueUpdate`位于`src/renderers/shared/stack/reconciler/ReactUpdates.js`中，这个方法则是整个React更新机制的灵魂：

```js
function enqueueUpdate(component) {
  ensureInjected();

  if (!batchingStrategy.isBatchingUpdates) {
    batchingStrategy.batchedUpdates(enqueueUpdate, component);
    return;
  }

  dirtyComponents.push(component);
  if (component._updateBatchNumber == null) {
    component._updateBatchNumber = updateBatchNumber + 1;
  }
}
```

这短短地几行代码里面蕴藏了整个更新机制的核心，就是这个`isBatchingUpdates`的控制变量，在之前挂载流程的时候我们有提过这个变量。

这里的逻辑其实比较简单，如果`isBatchingUpdates`是falsy，那直接调用`batchingStrategy.batchedUpdates`方法，否则就往`dirtyComponents`当中把当前的控制类实例push进去。

如果你直接去看这一块代码，可能很难理解这里面真正的含义是什么。

让我们回想一下，我们一般会把`setState`写在哪里。最常见的场景下，我们是在React生命周期的钩子函数中去调用setState，或者是在事件的回调函数里面。

而生命周期函数则是在React挂载和更新流程中触发，而在React挂载、事件触发前，我们的`isBatchingUpdates`已经开启了，回顾一下我们之前提到的挂载流程：

源码位于`src/renderers/shared/stack/reconciler/ReactDefaultBatchingStategy.js`:

```js
batchedUpdates: function(callback, a, b, c, d, e) {
  var alreadyBatchingUpdates = ReactDefaultBatchingStrategy.isBatchingUpdates;

  ReactDefaultBatchingStrategy.isBatchingUpdates = true;

  // The code is written this way to avoid extra allocations
  if (alreadyBatchingUpdates) {
    return callback(a, b, c, d, e);
  } else {
    return transaction.perform(callback, null, a, b, c, d, e);
  }
},
```

这段代码我们之前分析过，对于首次进来的情况，会开启一个`transaction`：

```js
var RESET_BATCHED_UPDATES = {
  initialize: emptyFunction,
  close: function() {
    ReactDefaultBatchingStrategy.isBatchingUpdates = false;
  },
};

var FLUSH_BATCHED_UPDATES = {
  initialize: emptyFunction,
  close: ReactUpdates.flushBatchedUpdates.bind(ReactUpdates),
};

var TRANSACTION_WRAPPERS = [FLUSH_BATCHED_UPDATES, RESET_BATCHED_UPDATES];
```

关注这两个Wrapper。它的执行顺序是先执行`FLUSH_BATCHED_UPDATES`，再执行`RESET_BATCHED_UPDATES`，而`FLUSH_BATCHED_UPDATES`这个Wrapper非常关键，它会在close回调的时候统一调用`ReactUpdates.flushBatchedUpdates`方法，然后再将`isBatchingUpdates`设为FALSE。

> 请注意，这个是在React挂载或是事件触发的时候启动的，它们是首次调用`batchedUpdates`的场景。

接着我们来看一下`flushBatchedUpdates`的实现：

```js
var flushBatchedUpdates = function() {
  while (dirtyComponents.length || asapEnqueued) {
    if (dirtyComponents.length) {
      var transaction = ReactUpdatesFlushTransaction.getPooled();
      transaction.perform(runBatchedUpdates, null, transaction);
      ReactUpdatesFlushTransaction.release(transaction);
    }

    if (asapEnqueued) {
      asapEnqueued = false;
      var queue = asapCallbackQueue;
      asapCallbackQueue = CallbackQueue.getPooled();
      queue.notifyAll();
      CallbackQueue.release(queue);
    }
  }
};
```

这里面`asap`的逻辑我们可以不用管它，主要用于一些表单组件的事件触发当中。

这里的核心逻辑就是把我们之前的`dirtyComponents`中所有被标记的组件都取出来，依次执行`runBatchedUpdates`，当然这里开启了一个`transaction`，让我们来看看它的Wrapper：

```js
var NESTED_UPDATES = {
  initialize: function() {
    this.dirtyComponentsLength = dirtyComponents.length;
  },
  close: function() {
    if (this.dirtyComponentsLength !== dirtyComponents.length) {
      dirtyComponents.splice(0, this.dirtyComponentsLength);
      flushBatchedUpdates();
    } else {
      dirtyComponents.length = 0;
    }
  },
};

var UPDATE_QUEUEING = {
  initialize: function() {
    this.callbackQueue.reset();
  },
  close: function() {
    this.callbackQueue.notifyAll();
  },
};

var TRANSACTION_WRAPPERS = [NESTED_UPDATES, UPDATE_QUEUEING];
```

这里面`UPDATE_QUEUEING`很好理解，是用来处理`setState`的回调函数的。而`NESTED_UPDATES`，则是在一个`setState`更新的周期内，又遇到了嵌套的`setState`调用，这个比较常见在`componentDidUpdate`钩子当中，很可能update之后又触发了`setState`。这个Wrapper的核心作用是确保，当前`setState`更新结束之后，能够让嵌套的`setState`流程继续触发。

接着我们看看`runBatchedUpdates`的核心实现：

```js
function runBatchedUpdates(transaction) {
  dirtyComponents.sort(mountOrderComparator);
  updateBatchNumber++;

  for (var i = 0; i < len; i++) {
    var component = dirtyComponents[i];

    var callbacks = component._pendingCallbacks;
    component._pendingCallbacks = null;

    ReactReconciler.performUpdateIfNecessary(
      component,
      transaction.reconcileTransaction,
      updateBatchNumber,
    );

    if (callbacks) {
      for (var j = 0; j < callbacks.length; j++) {
        transaction.callbackQueue.enqueue(
          callbacks[j],
          component.getPublicInstance(),
        );
      }
    }
  }
}
```

`runBatchedUpdates`的逻辑，我们去除了一些干扰的分支逻辑，它的核心逻辑是非常清晰的，那就是依次将所有标记的`dirtyComponent`取出，分别执行`performUpdateIfNecessary`方法，这个也是React用来更新组件的核心方法。如果包含回调，则会在执行完成更新后，依次触发回调。

至此，其实`setState`整体的流程已经分析完了，可以看到这里利用了多个`transaction`和队列去做异步化，最后再通过`performUpdateIfNecessary`来真正做到更新，这就是`batchedUpdate`的核心原理。

### 小结一下

整个React的`setState`异步化，或者说是`update`流程的异步化，其实还是比较难以理解的，要结合我们之前讲解的`transaction`核心原理、`React Mount`挂载过程才可以比较好地理解到，整体异步化的原理我们用一幅图来总结一下：

![](https://files.mdnice.com/user/13429/e2ede7f6-8165-4ad8-b5e3-f76791ac2dcf.png)

最后我们思考一下React对更新做异步化的原因：

* 出于性能考虑，update相对来说是一个比较重的操作，如果同步执行很多update，可能会导致浏览器出现卡顿，其实很多重复的setState操作都是可以合并成一次完成的。
* 不打断当前的执行流程，比如我们本身是在做挂载的流程，正常来说挂载后面还有一些收尾工作要处理，如果这时候遇到了setState操作，这个流程就会被打断，从而直接进入了另一个更新流程，整个生命周期就会变得非常复杂，有些必要的回收和通知操作也无法执行了。

关于`setState`异步化的考虑gaearon已经在issue里回复的非常深刻了，具体可以参见[这里](https://github.com/facebook/react/issues/11527#issuecomment-360199710)
