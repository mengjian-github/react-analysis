# React源码学习入门（七）详解ReactMount入口

> 本文基于React v15.6.2版本介绍，原因请参见新手如何学习React源码

### 源码分析

`ReactMount`的源码位于`src/renderers/dom/client/ReactMount.js`：

![](https://tva1.sinaimg.cn/large/e6c9d24egy1h5r7ggmfkij20s80dcdha.jpg)

在`ReactMount`中，我们常用的API是`render`和`unmountComponentAtNode`，而`render`则是整个应用的启动入口：

```js
ReactDOM.render(
  <Counter />,
  document.getElementById('container')
);
```

`render`的首次初始化核心方法实现在`_renderNewRootComponent`中：

```js
  _renderNewRootComponent: function(
    nextElement,
    container,
    shouldReuseMarkup,
    context,
  ) {
    
    ReactBrowserEventEmitter.ensureScrollValueMonitoring();
    var componentInstance = instantiateReactComponent(nextElement, false);

    ReactUpdates.batchedUpdates(
      batchedMountComponentIntoNode,
      componentInstance,
      container,
      shouldReuseMarkup,
      context,
    );

    var wrapperID = componentInstance._instance.rootID;
    instancesByReactRootID[wrapperID] = componentInstance;

    return componentInstance;
  },
```

这个方法核心做了几件事情：

1. 实例化`nextElement`，创建对应的辅助类
2. 调用`batchedUpdates`
3. 设置\_instancesByReactRootID，这个被devtools使用，不重要

前两步其实都非常核心，我们展开讲解一下。

首先，实例化的这个`nextElement`是什么？实际上在调用链的上一层函数`_renderSubtreeIntoContainer`可以找到：

```js
var nextWrappedElement = React.createElement(TopLevelWrapper, {
  child: nextElement,
});
```

实际上是React自己调用`createElement`创建的一个`Element`，而它的props的child是`nextElement`，这个element就是我们调用`render`的时候传入的根组件了。

那么`TopLevelWrapper`是什么呢？在`ReactMount`中是这么定义的：

```js
var topLevelRootCounter = 1;
var TopLevelWrapper = function() {
  this.rootID = topLevelRootCounter++;
};
TopLevelWrapper.prototype.isReactComponent = {};
if (__DEV__) {
  TopLevelWrapper.displayName = 'TopLevelWrapper';
}
TopLevelWrapper.prototype.render = function() {
  return this.props.child;
};
TopLevelWrapper.isReactTopLevelWrapper = true;
```

实际上就是一个`ReactComponent`，而它挂了一个成员`rootID`，就是一层容器，React给我们传入的Element又包裹了一层Wrapper组件，主要目的还是为了统一创建一个`ReactCompositeComponent`的控制类，上篇文章讲了React有四种控制类，而我们自己传入的根组件并不能确认是属于哪种类型，于是React为了更好地处理多次调用`render`的更新逻辑，就统一包了一个`WrapperComponnent`。

所以这里在`_renderNewRootComponent`时，调用`instantiateReactComponent`创建出来的实际上是Wrapper的instance，我们观察一下被传入`batchedUpdate`的几个参数：

* componentInstance，是Wrapper的控制类，是一个`ReactCompositeComponent`
* container，是我们的DOM容器
* shouldReuseMarkup，与二次render有关，先忽略
* context，与parent的context有关，这里属于非根节点render的场景，先忽略

在Mount过程中调用`batchedUpdate`，其实是为了在`rending`的生命周期中，例如`componentWillMount`或者`componentDidMount`，去调用`setState`更新能够做到批量更新一次，这个细节我们在后面的文章里面再细说。

接着我们看一下`batchedMountComponentIntoNode`：

```js
function batchedMountComponentIntoNode(
  componentInstance,
  container,
  shouldReuseMarkup,
  context,
) {
  var transaction = ReactUpdates.ReactReconcileTransaction.getPooled(
    /* useCreateElement */
    !shouldReuseMarkup && ReactDOMFeatureFlags.useCreateElement,
  );
  transaction.perform(
    mountComponentIntoNode,
    null,
    componentInstance,
    container,
    transaction,
    shouldReuseMarkup,
    context,
  );
  ReactUpdates.ReactReconcileTransaction.release(transaction);
}
```

这个方法核心是开启了`ReactReconcileTransaction`，这个`transaction`将会伴随`mount`的整个周期，这里采用之前讲过的对象池来做复用，不再赘述。

接下来看一下这个`transaction`是什么，源码位于`src/renderers/dom/client/ReactReconcileTransaction.js`：

```js
function ReactReconcileTransaction(useCreateElement: boolean) {
  this.reinitializeTransaction();
  this.renderToStaticMarkup = false;
  this.reactMountReady = CallbackQueue.getPooled(null);
  this.useCreateElement = useCreateElement;
}

var Mixin = {
  getTransactionWrappers: function() {
    return TRANSACTION_WRAPPERS;
  },

  getReactMountReady: function() {
    return this.reactMountReady;
  },

  getUpdateQueue: function() {
    return ReactUpdateQueue;
  },

  checkpoint: function() {
    return this.reactMountReady.checkpoint();
  },

  rollback: function(checkpoint) {
    this.reactMountReady.rollback(checkpoint);
  },

  destructor: function() {
    CallbackQueue.release(this.reactMountReady);
    this.reactMountReady = null;
  },
};
```

可以看到这个`transaction`上挂载了几个属性和方法，需要关注的是`reactMountReady`是一个队列，这个在挂载过程中会存放`componentDidMount`的回调，挂载完成后依次触发，所以说`componentDidMount`整体其实是异步的。

接下来再看一下wrapper的部分：

```js
var SELECTION_RESTORATION = {
  initialize: ReactInputSelection.getSelectionInformation,
  close: ReactInputSelection.restoreSelection,
};

var EVENT_SUPPRESSION = {
  initialize: function() {
    var currentlyEnabled = ReactBrowserEventEmitter.isEnabled();
    ReactBrowserEventEmitter.setEnabled(false);
    return currentlyEnabled;
  },
  close: function(previouslyEnabled) {
    ReactBrowserEventEmitter.setEnabled(previouslyEnabled);
  },
};

var ON_DOM_READY_QUEUEING = {
  initialize: function() {
    this.reactMountReady.reset();
  },

  close: function() {
    this.reactMountReady.notifyAll();
  },
};

var TRANSACTION_WRAPPERS = [
  SELECTION_RESTORATION,
  EVENT_SUPPRESSION,
  ON_DOM_READY_QUEUEING,
];
```

前两个和保存事件状态有关，而第三个`Wrapper`则是处理队列的通知，保证执行完`Mount`之后回调能够正常触发。

最后我们看一下`mountComponentIntoNode`的实现，这个方法也是整个`Mount`流程的最后一步：

```js
function mountComponentIntoNode(
  wrapperInstance,
  container,
  transaction,
  shouldReuseMarkup,
  context,
) {
  var markup = ReactReconciler.mountComponent(
    wrapperInstance,
    transaction,
    null,
    ReactDOMContainerInfo(wrapperInstance, container),
    context,
    0 /* parentDebugID */,
  );

  wrapperInstance._renderedComponent._topLevelWrapper = wrapperInstance;
  ReactMount._mountImageIntoNode(
    markup,
    container,
    wrapperInstance,
    shouldReuseMarkup,
    transaction,
  );
}
```

这里核心做了两件事：

1. 调用`ReactReconciler.mountComponent`拿到`markup`，这个是个递归的过程，也是`stack`核心，限于篇幅我们在后续文章中详解，可以认为最终拿到的`markup`是一个已经处理好的DOM节点（开启createElement的新版本），或是要插入的HTML片段（老版本）。
2. 调用`ReactMount._mountImageIntoNode`去挂载到真实的DOM容器下。

这里额外注意的一点是新增加了一个参数`containerInfo`，我们看一下`ReactDOMContainerInfo`，源码位于`src/renderers/dom/shared/ReactDOMContainerInfo.js`：

```js
function ReactDOMContainerInfo(topLevelWrapper, node) {
  var info = {
    _topLevelWrapper: topLevelWrapper,
    _idCounter: 1,
    _ownerDocument: node
      ? node.nodeType === DOC_NODE_TYPE ? node : node.ownerDocument
      : null,
    _node: node,
    _tag: node ? node.nodeName.toLowerCase() : null,
    _namespaceURI: node ? node.namespaceURI : null,
  };
  if (__DEV__) {
    info._ancestorInfo = node
      ? validateDOMNesting.updatedAncestorInfo(null, info._tag, null)
      : null;
  }
  return info;
}
```

生成的`containerInfo`主要挂载了`topLevelWrapper`的实例，container本身的节点信息，和一个`idCounter`，这些信息在后续更新过程中十分有用，可以稍微记住这里，后续更新再回过头来看。

最后再稍微看下`_mountImageIntoNode`，实际上在首次挂载时它的执行逻辑非常简单：

```js
  _mountImageIntoNode: function(
    markup,
    container,
    instance,
    shouldReuseMarkup,
    transaction,
  ) {
    if (transaction.useCreateElement) {
      while (container.lastChild) {
        container.removeChild(container.lastChild);
      }
      DOMLazyTree.insertTreeBefore(container, markup, null);
    } else {
      setInnerHTML(container, markup);
      ReactDOMComponentTree.precacheNode(instance, container.firstChild);
    }
  },
};
```

可以看到主要是调用`DOMLazyTree.insertTreeBefore`去插入节点，`DOMLazyTree`存在的意义是为了让IE系列有更好的性能，React团队调研发现直接插入大量节点在IE/Edge下不如小批量插入节点来得快（相差10倍以上的性能），因此这个文件专门用来磨平差异，我们这里可以简单理解为它就是原生的`insertBefore`方法。

### 小结一下

上述过程中讲了非常多ReactMount的细节，实际上很多都是为了`Update`去做准备，我们抛开`Update`不谈，只看Mount，实际上非常简单，整个Mount它就做了三件事情：

1. 创建了一个根节点（在内部是统一包裹了Wrapper组件）实例。
2. 调用`ReactReconciler.mountComponent`获取Markup，这个是个递归的过程。
3. 调用`mountImageIntoNode`将Markup挂载到容器的DOM节点上。
