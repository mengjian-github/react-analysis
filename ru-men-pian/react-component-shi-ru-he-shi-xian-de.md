# React Component是如何实现的？



> 本文基于React v15.6.2版本介绍，原因请参见新手如何学习React源码

### 源码解析

`ReactComponent`的实现超出想象的简单，位于`src/isomorphic/class/ReactBaseClasses.js`：

```js
function ReactComponent(props, context, updater) {
  this.props = props;
  this.context = context;
  this.refs = emptyObject;
  // We initialize the default updater but the real one gets injected by the
  // renderer.
  this.updater = updater || ReactNoopUpdateQueue;
}

ReactComponent.prototype.isReactComponent = {};
```

是的，你没有看错，这就是我们平时继承的`ReactComponent`的全部面貌，其中的`updater`和React的更新机制有关，后续文章会详细介绍。

既然Component的实现如此简单，那React又是如何去处理背后的复杂逻辑呢？

这个需要从实例化说起。

### React组件实例化

React组件实例化代码位于`src/renderers/shared/stack/reconciler`：

```js
function instantiateReactComponent(node, shouldHaveDebugID) {
  var instance;

  if (node === null || node === false) {
    // 实例化一个空组件
    instance = ReactEmptyComponent.create(instantiateReactComponent);
  } else if (typeof node === 'object') {
    var element = node;
    var type = element.type;

    if (typeof element.type === 'string') {
       // 实例化一个Host组件
      instance = ReactHostComponent.createInternalComponent(element);
    } else if (isInternalComponentType(element.type)) {
      // 此处为native使用，省略这段代码
    } else {
      // 实例化一个Component组件
      instance = new ReactCompositeComponentWrapper(element);
    }
  } else if (typeof node === 'string' || typeof node === 'number') {
    // 实例化一个Text组件
    instance = ReactHostComponent.createInstanceForText(node);
  } else {
    invariant(false, 'Encountered invalid React node of type %s', typeof node);
  }

  instance._mountIndex = 0;
  instance._mountImage = null;

  return instance;
}
```

首先看一下`instantiateReactComponent`的入参，是`node`，这个`node`就是我们前面提到的`ReactNode`，也就是`render`的返回值。

回顾一下之前我们`render`的返回值类型：

* null、false，这个在React里面会初始化一个`ReactEmptyComponent`
* string，纯字符串，React会初始化成一个`ReactTextComponent`
* type为string，也就是表示DOM原生标签，会初始化成一个`HostComponent->ReactDOMComponent`
* 最后type是一个ReactComponent，会初始化成一个`ReactCompositeComponent`

以我们之前分析的JSX为例：

```js
render() {
  return /*#__PURE__*/React.createElement("button", {
    onClick: this.handleClick
  }, "Click me! Number of clicks: ", this.state.count);
}
```

这个结构的Element，被实例化的情况如下：

* `button`会被实例化为一个`ReactDOMComponent`
* "Click me! Number of clicks: "实例化为一个`ReactTextComponent`
* `this.state.count`也会被实例化为一个`ReactTextComponent`

### 四个控制类

#### ReactEmptyComponent

源码位于`src/renderers/dom/shared/ReactDOMEmptyComponent.js`：

![](https://files.mdnice.com/user/13429/6d3143f6-2593-4072-aa73-9fb9d7641d28.png)

`ReactEmptyComponent`是最简单的组件控制类，实际上是由空节点构成，只包含了挂载等核心接口。

#### ReactTextComponent

源码位于`src/renderers/dom/shared/ReactDOMTextComponent.js`：

![](https://files.mdnice.com/user/13429/b5ce8088-5920-488a-8303-683f34d2f4d7.png)

`ReactTextComponent`相对来说也是比较简单的组件，与`ReactDOMEmptyComponent`不同的是，文本节点是有更新逻辑的，更新逻辑为替换其中的文本内容。

#### ReactDOMComponent

源码位于`src/renderers/dom/shared/ReactDOMComponent.ts`

![](https://files.mdnice.com/user/13429/ecbcf113-d88d-46ce-8302-b60bed10225b.png)

`ReactDOMComponent`相对来说一个比较复杂且核心的文件，它实现了整个DOM的挂载、更新、卸载逻辑，整体DOM属性的挂载、更新也是在这里完成。

#### ReactCompositeComponent

源码位于`src/renderers/shared/stack/reconciler/ReactCompositeComponent.js`：

![](https://files.mdnice.com/user/13429/c9329fec-fbd8-4134-bd8b-5a628484b5c0.png)

`ReactCompositeComponent`的实现逻辑也较为复杂，但是React核心生命周期都在这里实现，我们写的React组件大多都是需要这个控制类的辅助，最终访问到`DOMComponent`和`TextComponent`，从而实现整体的挂载和更新。

### 小结一下

`ReactComponent`本身没有什么实现，只是提供了统一的方法包裹和构造函数。在React内部，是通过4个控制类来初始化组件的，这四个控制类非常重要，承载了React组件的核心逻辑实现。

本文介绍的组件实例化过程，实际上就是React内部将组件树逐步建立的过程，通过控制类-DOM/文本这样的映射机制，搭建起整体React的骨架结构。
