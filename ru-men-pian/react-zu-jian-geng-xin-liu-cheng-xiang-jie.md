# React组件更新流程详解

> 本文基于React v15.6.2版本介绍，原因请参见新手如何学习React源码

### 源码分析

上一篇文章提到最后更新组件是走到了`performUpdateIfNecessary`方法，让我们来看一看它的实现：

```js
  performUpdateIfNecessary: function(transaction) {
    if (this._pendingElement != null) {
      ReactReconciler.receiveComponent(
        this,
        this._pendingElement,
        transaction,
        this._context,
      );
    } else if (this._pendingStateQueue !== null || this._pendingForceUpdate) {
      this.updateComponent(
        transaction,
        this._currentElement,
        this._currentElement,
        this._context,
        this._context,
      );
    } else {
      this._updateBatchNumber = null;
    }
  },
```

这个方法其实最终走到的是`updateComponent`方法，并且注意的是，在我们更新state的当前这个组件，它传入的prev和next都是相同的，这个后面会决定`willReceiveProps`会不会触发。

接下来就是React组件核心更新方法`updateComponent`，源码位于`src/renderers/shared/stack/reconciler/ReactCompositeComponent.js`：

```js
  updateComponent: function(
    transaction,
    prevParentElement,
    nextParentElement,
    prevUnmaskedContext,
    nextUnmaskedContext,
  ) {
    var inst = this._instance;

    var willReceive = false;
    var nextContext;

    if (this._context === nextUnmaskedContext) {
      nextContext = inst.context;
    } else {
      nextContext = this._processContext(nextUnmaskedContext);
      willReceive = true;
    }

    var prevProps = prevParentElement.props;
    var nextProps = nextParentElement.props;

    if (prevParentElement !== nextParentElement) {
      willReceive = true;
    }

    // 1. 如果是receive的情况，触发componentWillReceiveProps
    if (willReceive && inst.componentWillReceiveProps) {
      inst.componentWillReceiveProps(nextProps, nextContext);
    }

    // 2. 合并当前的未处理的state
    var nextState = this._processPendingState(nextProps, nextContext);
    var shouldUpdate = true;

    // 3. 计算shouldUpdate
    if (!this._pendingForceUpdate) {
      if (inst.shouldComponentUpdate) {
          shouldUpdate = inst.shouldComponentUpdate(
            nextProps,
            nextState,
            nextContext,
          );
      } else {
        if (this._compositeType === CompositeTypes.PureClass) {
          shouldUpdate =
            !shallowEqual(prevProps, nextProps) ||
            !shallowEqual(inst.state, nextState);
        }
      }
    }

    this._updateBatchNumber = null;
    if (shouldUpdate) {
      this._pendingForceUpdate = false;
      this._performComponentUpdate(
        nextParentElement,
        nextProps,
        nextState,
        nextContext,
        transaction,
        nextUnmaskedContext,
      );
    } else {
      this._currentElement = nextParentElement;
      this._context = nextUnmaskedContext;
      inst.props = nextProps;
      inst.state = nextState;
      inst.context = nextContext;
    }
  },
```

这个函数核心做了3件事情：

1. 触发`componentWillReceiveProps`钩子，这个钩子的触发条件是当context或element发生变化时，显然，刚刚我们进来时发现这里的prev和next都是一样的，也就是触发`setState`的那个组件是不会调用`componentWillReceiveProps`的。
2. 合并当前的未处理的state，这个就是将之前setState插入队列里的state一次性合并到当前的state上，这里的合并用的是`Object.assign`。
3. 计算shouldUpdate，shouldUpdate默认为true，这也是React最大程度保证了组件都能被更新到，我们可以在组件里面实现自己的`shouldComponentUpdate`方法来决定是否重新render，另外对于PureComponent来说，这里通过`shallowEqual`来判断state和props是否发生了变化，主要利用的是`Object.is`判断是否相等。

当组件被判定为`shouldUpdate`的时候，就会走到`_performComponentUpdate`来执行更新：

```js
  _performComponentUpdate: function(
    nextElement,
    nextProps,
    nextState,
    nextContext,
    transaction,
    unmaskedContext,
  ) {
    var inst = this._instance;

    var hasComponentDidUpdate = Boolean(inst.componentDidUpdate);
    var prevProps;
    var prevState;
    var prevContext;
    if (hasComponentDidUpdate) {
      prevProps = inst.props;
      prevState = inst.state;
      prevContext = inst.context;
    }

    // 1. 触发willUpdate钩子
    if (inst.componentWillUpdate) {
        inst.componentWillUpdate(nextProps, nextState, nextContext);
    }

    // 2. 更新当前的props和state
    this._currentElement = nextElement;
    this._context = unmaskedContext;
    inst.props = nextProps;
    inst.state = nextState;
    inst.context = nextContext;

    // 3. 更新子组件
    this._updateRenderedComponent(transaction, unmaskedContext);

    // 4. componentDidUpdate入队
    if (hasComponentDidUpdate) {
        transaction
          .getReactMountReady()
          .enqueue(
            inst.componentDidUpdate.bind(
              inst,
              prevProps,
              prevState,
              prevContext,
            ),
            inst,
          );
      }
  },
```

这个函数主要做了以下几件事：

1. 触发`componentWillUpdate`钩子
2. 更新当前组件实例的props和state
3. 更新子组件
4. componentDidUpdate入队，这个和componentDidMount是一样的，都是通过Reconciler的transaction在close阶段按照队列触发。

接下来着重看一下更新子组件的流程：

```js
  _updateRenderedComponent: function(transaction, context) {
    var prevComponentInstance = this._renderedComponent;
    var prevRenderedElement = prevComponentInstance._currentElement;
    var nextRenderedElement = this._renderValidatedComponent();

    // shouldUpdateReactComponent，则调用receiveComponent更新子组件
    if (shouldUpdateReactComponent(prevRenderedElement, nextRenderedElement)) {
      ReactReconciler.receiveComponent(
        prevComponentInstance,
        nextRenderedElement,
        transaction,
        this._processChildContext(context),
      );
    } else {
      // 否则，卸载当前的组件重新执行mount流程
      var oldHostNode = ReactReconciler.getHostNode(prevComponentInstance);
      ReactReconciler.unmountComponent(prevComponentInstance, false);

      var nodeType = ReactNodeTypes.getType(nextRenderedElement);
      this._renderedNodeType = nodeType;
      var child = this._instantiateReactComponent(
        nextRenderedElement,
        nodeType !== ReactNodeTypes.EMPTY /* shouldHaveDebugID */,
      );
      this._renderedComponent = child;

      var nextMarkup = ReactReconciler.mountComponent(
        child,
        transaction,
        this._hostParent,
        this._hostContainerInfo,
        this._processChildContext(context),
      );

      this._replaceNodeWithMarkup(
        oldHostNode,
        nextMarkup,
        prevComponentInstance,
      );
    }
  },
```

这个函数核心是判断`shouldUpdateReactComponent`，如果是的话，那就走子组件的更新流程，否则，就销毁子组件，重新挂载。

一般来说，针对子组件的销毁和重建是比较消耗性能的，而且会使得生命周期函数被重复触发，所以React采用一个简单的原则来判断是否需要重新挂载，这也是Diff算法的起点：

```js
function shouldUpdateReactComponent(prevElement, nextElement) {
  var prevEmpty = prevElement === null || prevElement === false;
  var nextEmpty = nextElement === null || nextElement === false;
  if (prevEmpty || nextEmpty) {
    return prevEmpty === nextEmpty;
  }

  var prevType = typeof prevElement;
  var nextType = typeof nextElement;
  if (prevType === 'string' || prevType === 'number') {
    return nextType === 'string' || nextType === 'number';
  } else {
    return (
      nextType === 'object' &&
      prevElement.type === nextElement.type &&
      prevElement.key === nextElement.key
    );
  }
}
```

解读一下这个关键函数，分几类情况：

1. 是`emptyComponent`的场景，如果同为false或者同为null，则不需要重新挂载，否则重新挂载。
2. 是`string`或`number`的场景，也就是一个文本节点，前后都是文本节点的话，是不需要重新挂载的。
3. 其他的情况，得看两个组件是否是同一个类型，以及`key`是否相同，若两个条件同时满足，则不需要重新挂载。

所有触发的子组件，默认按照`receiveComponent`的模式往下递归，如果遇到React组件，又会重复之前的步骤，它的入口是：

```js
  receiveComponent: function(nextElement, transaction, nextContext) {
    var prevElement = this._currentElement;
    var prevContext = this._context;

    this._pendingElement = null;

    this.updateComponent(
      transaction,
      prevElement,
      nextElement,
      prevContext,
      nextContext,
    );
  },
```

`updateComponent`流程上面已经分析过了，不再赘述。

### 小结一下

本文主要分析了React组件的更新过程，重在几个生命周期函数的触发，以及更新策略，具体真正的更新是在DOMComponent中。我们可以简单总结一下React组件更新的流程图：

![](https://files.mdnice.com/user/13429/15087513-121d-42d4-ab25-ab69a70872cd.png)
