# React源码学习入门（八）React组件挂载Component细节流程

> 本文基于React v15.6.2版本介绍，原因请参见新手如何学习React源码

### 源码分析

在上一篇文章的最后，我们走到了`mountComponentIntoNode`，它通过调用`ReactReconciler.mountComponent`来获取Markup，这个也是React执行挂载的核心入口，源码位于`src/renderers/shared/stack/reconciler/ReactReconciler.js`：

```js
  mountComponent: function(
    internalInstance,
    transaction,
    hostParent,
    hostContainerInfo,
    context,
    parentDebugID, // 0 in production and for roots
  ) {
    var markup = internalInstance.mountComponent(
      transaction,
      hostParent,
      hostContainerInfo,
      context,
      parentDebugID,
    );
    if (
      internalInstance._currentElement &&
      internalInstance._currentElement.ref != null
    ) {
      transaction.getReactMountReady().enqueue(attachRefs, internalInstance);
    }
    return markup;
  },
```

可以看到这个方法的核心只是调用`instance`的`mountComponent`方法，这个`instance`是我们一开始进来的`wrapperInstance`，也就是React自己包裹的那一层组件，它是一个`CompositeComponent`，注意这里的参数顺序换了，实际上其他参数没有变，`hostParent`在我们初次挂载的时候为null。

一般来说，真正去产生`markup`的是真实的DOM节点，而不是抽象的React组件，顺序一般是`ReactCompositeComponent` -> `ReactDOMComponent` -> `ReactDOMTextComponent`/`ReactEmptyComponent`，所以一开始去看一下`ReactCompositeComponent`是明智的选择。

源码位于`src/renderers/shared/stack/reconciler/ReactCompositeComponent.js`：

```js
  mountComponent: function(
    transaction,
    hostParent,
    hostContainerInfo,
    context,
  ) {
    this._context = context;
    this._mountOrder = nextMountID++;
    this._hostParent = hostParent;
    this._hostContainerInfo = hostContainerInfo;

    var publicProps = this._currentElement.props;
    var publicContext = this._processContext(context);

    var Component = this._currentElement.type;

    var updateQueue = transaction.getUpdateQueue();

    // 1. 实例化组件
    var doConstruct = shouldConstruct(Component);
    var inst = this._constructComponent(
      doConstruct,
      publicProps,
      publicContext,
      updateQueue,
    );
    var renderedElement;

    // Support functional components
    if (!doConstruct && (inst == null || inst.render == null)) {
      renderedElement = inst;
      warnIfInvalidElement(Component, renderedElement);
      inst = new StatelessComponent(Component);
      this._compositeType = CompositeTypes.StatelessFunctional;
    } else {
      if (isPureComponent(Component)) {
        this._compositeType = CompositeTypes.PureClass;
      } else {
        this._compositeType = CompositeTypes.ImpureClass;
      }
    }

    inst.props = publicProps;
    inst.context = publicContext;
    inst.refs = emptyObject;
    inst.updater = updateQueue;

    this._instance = inst;

    ReactInstanceMap.set(inst, this);

    var initialState = inst.state;
    if (initialState === undefined) {
      inst.state = initialState = null;
    }

    this._pendingStateQueue = null;
    this._pendingReplaceState = false;
    this._pendingForceUpdate = false;

    var markup;
    if (inst.unstable_handleError) {
      markup = this.performInitialMountWithErrorHandling(
        renderedElement,
        hostParent,
        hostContainerInfo,
        transaction,
        context,
      );
    } else {
      // 2. 执行挂载
      markup = this.performInitialMount(
        renderedElement,
        hostParent,
        hostContainerInfo,
        transaction,
        context,
      );
    }

    // 3. 处理`componentDidMount`回调
    if (inst.componentDidMount) {
        transaction.getReactMountReady().enqueue(inst.componentDidMount, inst);
    }

    return markup;
  },
```

对于React组件的`mountComponent`过程，主要做了3件事：

1. 实例化React组件本身
2. 执行performInitialMount，获得Markup
3. 处理`componentDidMount`回调

其中第二步处理是最复杂的，我们先来看看1和3：

#### 实例化React组件

首先我们先拿到React组件本身：

```js
var Component = this._currentElement.type;
```

这里的`_currentElement`，其实就是我们JSX所调用的`createElement`的返回值，这一点之前已经详细介绍过，而对于React组件来说，它的整个class都是挂载`type`这个字段下的，所以这里通过`type`就可以拿到React组件的类。

接着有一个`shouldConstruct`的判断：

```js
var doConstruct = shouldConstruct(Component);

function shouldConstruct(Component) {
  return !!(Component.prototype && Component.prototype.isReactComponent);
}
```

我们知道，React组件有两种模式：类组件和函数组件。

对于类组件的写法之前已经分析过，它会在`prototype`上挂一个`isReactComponent`属性，所以这里的判断是用来区分类组件和函数组件的。

紧接着开始实例化，实例化过程最终会走到这个函数：

```js
_constructComponentWithoutOwner: function(
  doConstruct,
  publicProps,
  publicContext,
  updateQueue,
) {
  var Component = this._currentElement.type;

  if (doConstruct) {
      return new Component(publicProps, publicContext, updateQueue);
  }

  return Component(publicProps, publicContext, updateQueue);
},
```

可以看到这里函数组件就直接执行函数了，所以这里拿到的实例有两种情况：

* 类组件，这里拿到的是instance
* 函数组件，这里拿到的其实是element

所以有下面这段代码来修正一下函数组件的instance：

```js
if (!doConstruct && (inst == null || inst.render == null)) {
  renderedElement = inst;
  warnIfInvalidElement(Component, renderedElement);
  inst = new StatelessComponent(Component);
  this._compositeType = CompositeTypes.StatelessFunctional;
} else {
  if (isPureComponent(Component)) {
    this._compositeType = CompositeTypes.PureClass;
  } else {
    this._compositeType = CompositeTypes.ImpureClass;
  }
}
```

这里对于函数组件，会实例化一个`StatelessComponent`的实例出来：

```js
function StatelessComponent(Component) {}
StatelessComponent.prototype.render = function() {
  var Component = ReactInstanceMap.get(this)._currentElement.type;
  var element = Component(this.props, this.context, this.updater);
  warnIfInvalidElement(Component, element);
  return element;
};
```

这个实例只挂载了一个`render`方法，用来执行函数组件的渲染。这个`render`方法会在后续更新流程中被统一调用。

#### 处理`componentDidMount`回调

```js
transaction.getReactMountReady().enqueue(inst.componentDidMount, inst);
```

这里其实之前看过了，队列会在`transaction`周期的末尾被触发，而上个函数本身是个递归的过程，根据队列的特性，这里叶子节点的`componentDidMount`就会被先触发。

#### 执行`performInitialMount`拿到Markup

```js
performInitialMount: function(
  renderedElement,
  hostParent,
  hostContainerInfo,
  transaction,
  context,
) {
  var inst = this._instance;

  // 1. 执行componentWillMount钩子
  if (inst.componentWillMount) {
      inst.componentWillMount();
      inst.state = this._processPendingState(inst.props, inst.context);
  }

  // If not a stateless component, we now render
  if (renderedElement === undefined) {
    renderedElement = this._renderValidatedComponent();
  }

  // 2. 初始化child实例
  var nodeType = ReactNodeTypes.getType(renderedElement);
  this._renderedNodeType = nodeType;
  var child = this._instantiateReactComponent(
    renderedElement,
    nodeType !== ReactNodeTypes.EMPTY /* shouldHaveDebugID */,
  );
  this._renderedComponent = child;
  
  // 3. 挂载child
  var markup = ReactReconciler.mountComponent(
    child,
    transaction,
    hostParent,
    hostContainerInfo,
    this._processChildContext(context),
    debugID,
  );

  return markup;
},
```

在这个`performInitialMount`核心实现中，主要做了3件事：

1. 执行`componentWillMount`钩子
2. 初始化child实例
3. 挂载Child

实际上，如果说在上一步处理的是当前组件的信息，那么这一步主要就处理的是它的渲染组件的信息，也就是它的子组件，首先执行钩子，并初始化子组件的控制类，接着执行子组件的`mountComponent`方法。

注意一下这里的细节，如果是函数组件，实际上之前相当于已经执行过render了，所以这里看触发render只针对类组件。

而挂载子组件则可以触发一个递归的过程，最终的markup还是通过挂载子组件取到。

我们知道一个React组件并不实际会被浏览器加载，只有到达DOM节点时才会被渲染到浏览器中，所以markup只有在这里递归子组件走到了叶子节点，才会有真正的实质内容出现。

### 小结一下

上面主要分析了React组件内部是如何实现挂载的，实际上对于一个`ReactCompositeComponent`来说，最终是不会被挂载到浏览器上的，它主要在`reconciler`目录下实现，表示这里的逻辑实际上和平台无关，而是React自身的底层逻辑，我们把重要的步骤画一个图：

![](https://tva1.sinaimg.cn/large/e6c9d24egy1h5r7gycbroj20ma0usgna.jpg)

实际上，通过实例化、执行render、执行生命周期、递归子组件挂载的过程，就是整个React组件挂载的全貌了，而真正处理挂载的细节逻辑是在叶子节点（DOM处理）上，下篇文章我们将仔细讨论下叶子节点的挂载过程。
