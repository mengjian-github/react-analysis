# DOM组件更新流程与Diff算法

> 本文基于React v15.6.2版本介绍，原因请参见新手如何学习React源码

### 源码分析

前面提到过最终的更新还是要在DOMComponent完成，而`setState`后，触发到DOM的更新入口是`receiveComponent`，源码在`src/renderers/dom/shared/ReactDOMComponent.js`：

```js
  receiveComponent: function(nextElement, transaction, context) {
    var prevElement = this._currentElement;
    this._currentElement = nextElement;
    this.updateComponent(transaction, prevElement, nextElement, context);
  },
```

实际上`updateComponent`我们之前在挂载过程中提到过，现在我们来关注里面的更新逻辑：

```js
  updateComponent: function(transaction, prevElement, nextElement, context) {
    var lastProps = prevElement.props;
    var nextProps = this._currentElement.props;
    this._updateDOMProperties(lastProps, nextProps, transaction);
    this._updateDOMChildren(lastProps, nextProps, transaction, context);
  },
```

省略其他分支，更新逻辑是由`properties`更新和`children`更新完成的，先看看`properties`：

```js
  _updateDOMProperties: function(lastProps, nextProps, transaction) {
    var propKey;
    var styleName;
    var styleUpdates;
    
    // 1. 处理旧的Props
    for (propKey in lastProps) {
      if (
        nextProps.hasOwnProperty(propKey) ||
        !lastProps.hasOwnProperty(propKey) ||
        lastProps[propKey] == null
      ) {
        continue;
      }
      
      // 注意判断条件，只有nextProps没有的才会走到这里，意味着这里需要做删除操作
      
      // 1.1 处理旧的style属性，清空styleUpdates对应的属性
      if (propKey === STYLE) {
        var lastStyle = this._previousStyleCopy;
        for (styleName in lastStyle) {
          if (lastStyle.hasOwnProperty(styleName)) {
            styleUpdates = styleUpdates || {};
            styleUpdates[styleName] = '';
          }
        }
        this._previousStyleCopy = null;
      } 
      
      // 1.2 处理旧的事件属性，先删除旧的监听器
      else if (registrationNameModules.hasOwnProperty(propKey)) {
        if (lastProps[propKey]) {
          deleteListener(this, propKey);
        }
      } 
      // 1.3 处理旧的属性，删除旧的属性
      else if (
        DOMProperty.properties[propKey] ||
        DOMProperty.isCustomAttribute(propKey)
      ) {
        DOMPropertyOperations.deleteValueForProperty(getNode(this), propKey);
      }
    }
    
    // 2. 处理新的属性
    for (propKey in nextProps) {
      var nextProp = nextProps[propKey];
      var lastProp = propKey === STYLE
        ? this._previousStyleCopy
        : lastProps != null ? lastProps[propKey] : undefined;
      if (
        !nextProps.hasOwnProperty(propKey) ||
        nextProp === lastProp ||
        (nextProp == null && lastProp == null)
      ) {
        continue;
      }
      
      // 2.1 处理新的样式属性
      if (propKey === STYLE) {
        if (nextProp) {
          nextProp = this._previousStyleCopy = Object.assign({}, nextProp);
        } else {
          this._previousStyleCopy = null;
        }
        if (lastProp) {
          // 和前面的处理一样，对于之前的style标签，如果next里面没有的，需要清空
          for (styleName in lastProp) {
            if (
              lastProp.hasOwnProperty(styleName) &&
              (!nextProp || !nextProp.hasOwnProperty(styleName))
            ) {
              styleUpdates = styleUpdates || {};
              styleUpdates[styleName] = '';
            }
          }
          // 更新新的样式属性
          for (styleName in nextProp) {
            if (
              nextProp.hasOwnProperty(styleName) &&
              lastProp[styleName] !== nextProp[styleName]
            ) {
              styleUpdates = styleUpdates || {};
              styleUpdates[styleName] = nextProp[styleName];
            }
          }
        } else {
          // 全新的样式属性
          styleUpdates = nextProp;
        }
      } 
      // 2.2 处理新的事件
      else if (registrationNameModules.hasOwnProperty(propKey)) {
        if (nextProp) {
          enqueuePutListener(this, propKey, nextProp, transaction);
        } else if (lastProp) {
          deleteListener(this, propKey);
        }
      }
      // 2.3 处理新的属性
      else if (
        DOMProperty.properties[propKey] ||
        DOMProperty.isCustomAttribute(propKey)
      ) {
        var node = getNode(this);
        if (nextProp != null) {
          DOMPropertyOperations.setValueForProperty(node, propKey, nextProp);
        } else {
          DOMPropertyOperations.deleteValueForProperty(node, propKey);
        }
      }
    }
    if (styleUpdates) {
      CSSPropertyOperations.setValueForStyles(
        getNode(this),
        styleUpdates,
        this,
      );
    }
  },
```

对于更新属性的逻辑，和前面提到的非常类似，只是增加了删除的处理：

1. **更新Style属性**，因为style本身是一个对象，因此要遍历style中的每个key，对比新老对象来处理是删除还是更新。
2. **更新事件属性**，这里只需要根据新属性的有无来判断是新增还是删除
3. **更新其他DOM属性**，我们只需要根据新旧属性来处理新增和删除

接下来我们重点来看一下`children`的更新：

```js
  _updateDOMChildren: function(lastProps, nextProps, transaction, context) {
    var lastContent = CONTENT_TYPES[typeof lastProps.children]
      ? lastProps.children
      : null;
    var nextContent = CONTENT_TYPES[typeof nextProps.children]
      ? nextProps.children
      : null;

    var lastHtml =
      lastProps.dangerouslySetInnerHTML &&
      lastProps.dangerouslySetInnerHTML.__html;
    var nextHtml =
      nextProps.dangerouslySetInnerHTML &&
      nextProps.dangerouslySetInnerHTML.__html;

    var lastChildren = lastContent != null ? null : lastProps.children;
    var nextChildren = nextContent != null ? null : nextProps.children;

    var lastHasContentOrHtml = lastContent != null || lastHtml != null;
    var nextHasContentOrHtml = nextContent != null || nextHtml != null;
    if (lastChildren != null && nextChildren == null) {
      this.updateChildren(null, transaction, context);
    } else if (lastHasContentOrHtml && !nextHasContentOrHtml) {
      this.updateTextContent('');
    }

    if (nextContent != null) {
      if (lastContent !== nextContent) {
        this.updateTextContent('' + nextContent);
      }
    } else if (nextHtml != null) {
      if (lastHtml !== nextHtml) {
        this.updateMarkup('' + nextHtml);
      }
    } else if (nextChildren != null) {
      this.updateChildren(nextChildren, transaction, context);
    }
  },
```

这里我们先忽略前面两个特殊的判断条件，其实逻辑非常简单：

1. 如果更新后的是content（文本节点），则直接调用`updateTextContent`
2. 如果更新后的是HTML（dangerouslySetInnerHTML），则直接调用`updateMarkup`
3. 其他情况，调用`updateChildren`

于是实现的重头戏应该是`updateChildren`：

```js
_updateChildren: function(
  nextNestedChildrenElements,
  transaction,
  context,
) {
  var prevChildren = this._renderedChildren;
  var removedNodes = {};
  var mountImages = [];
  
  // 1. 更新children
  var nextChildren = this._reconcilerUpdateChildren(
    prevChildren,
    nextNestedChildrenElements,
    mountImages,
    removedNodes,
    transaction,
    context,
  );
  if (!nextChildren && !prevChildren) {
    return;
  }
  var updates = null;
  var name;
  // `nextIndex` will increment for each child in `nextChildren`, but
  // `lastIndex` will be the last index visited in `prevChildren`.
  var nextIndex = 0;
  var lastIndex = 0;
  // `nextMountIndex` will increment for each newly mounted child.
  var nextMountIndex = 0;
  var lastPlacedNode = null;
  for (name in nextChildren) {
    if (!nextChildren.hasOwnProperty(name)) {
      continue;
    }
    var prevChild = prevChildren && prevChildren[name];
    var nextChild = nextChildren[name];
    
    // 2. 在设置了key的情况下，只移动个别节点，做到算法最优
    if (prevChild === nextChild) {
      updates = enqueue(
        updates,
        this.moveChild(prevChild, lastPlacedNode, nextIndex, lastIndex),
      );
      lastIndex = Math.max(prevChild._mountIndex, lastIndex);
      prevChild._mountIndex = nextIndex;
    } else {
      if (prevChild) {
        // Update `lastIndex` before `_mountIndex` gets unset by unmounting.
        lastIndex = Math.max(prevChild._mountIndex, lastIndex);
        // The `removedNodes` loop below will actually remove the child.
      }
      
      // 3. 对于全新节点，直接插入
      updates = enqueue(
        updates,
        this._mountChildAtIndex(
          nextChild,
          mountImages[nextMountIndex],
          lastPlacedNode,
          nextIndex,
          transaction,
          context,
        ),
      );
      nextMountIndex++;
    }
    nextIndex++;
    lastPlacedNode = ReactReconciler.getHostNode(nextChild);
  }
  // 4. 删除旧的节点
  for (name in removedNodes) {
    if (removedNodes.hasOwnProperty(name)) {
      updates = enqueue(
        updates,
        this._unmountChild(prevChildren[name], removedNodes[name]),
      );
    }
  }
  // 统一执行更新（可能包括移动、新增、删除等）
  if (updates) {
    processQueue(this, updates);
  }
  this._renderedChildren = nextChildren;
},
```

这段代码就是React Diff算法的核心了，它主要做了以下几个事情：

1. 先执行`_reconcilerUpdateChildren`，拿到处理后的children节点
2. 经过上一步的处理，也许一些DOM节点已经更新（会调用之前说到的updateComponent），但是还可能发生移动、删除、新增的操作，这个第一步并没有处理
3. 处理节点移动、删除、新增的情况，这也是Diff算法的核心。

接下来详细分析下这几个步骤：

首先来看看`_reconcilerUpdateChildren`的实现：

```js
 _reconcilerUpdateChildren: function(
    prevChildren,
    nextNestedChildrenElements,
    mountImages,
    removedNodes,
    transaction,
    context,
  ) {
    var nextChildren;
    var selfDebugID = 0;
    nextChildren = flattenChildren(nextNestedChildrenElements, selfDebugID);
    ReactChildReconciler.updateChildren(
      prevChildren,
      nextChildren,
      mountImages,
      removedNodes,
      transaction,
      this,
      this._hostContainerInfo,
      context,
      selfDebugID,
    );
    return nextChildren;
  },
```

这里其实是调用`updateChildren`方法，最终返回`nextChildren`，值得关注的是这里有一个`flattenChildren`的操作，它是实现React的`key`机制的核心：

```js
function flattenSingleChildIntoContext(
  traverseContext: mixed,
  child: ReactElement<any>,
  name: string,
  selfDebugID?: number,
): void {
  if (traverseContext && typeof traverseContext === 'object') {
    const result = traverseContext;
    const keyUnique = result[name] === undefined;
    if (keyUnique && child != null) {
      result[name] = child;
    }
  }
}

function flattenChildren(
  children: ReactElement<any>,
  selfDebugID?: number,
): ?{[name: string]: ReactElement<any>} {
  if (children == null) {
    return children;
  }
  var result = {};
    traverseAllChildren(children, flattenSingleChildIntoContext, result);
  return result;
}
```

可以看到其实`flattenChildren`就是把children的嵌套结构转化为对象，而它的key是由组件的key生成的，当我们没有指定key的时候，它是这样：

![](https://files.mdnice.com/user/13429/35aa446e-6e6e-40fb-886e-fd0c3010dfbb.png)

当我们设置`key`的时候，它是这样：

![](https://files.mdnice.com/user/13429/4f180238-708d-4726-8a23-f030637d2d49.png)

它的生成函数位于`src/shared/utils/traverseAllChildren.js`：

```js
function getComponentKey(component, index) {
  // Do some typechecking here since we call this blindly. We want to ensure
  // that we don't block potential future ES APIs.
  if (component && typeof component === 'object' && component.key != null) {
    // Explicit key
    return KeyEscapeUtils.escape(component.key);
  }
  // Implicit key determined by the index in the set
  return index.toString(36);
}
```

所以接下来的Diff过程，我们明白了，它完全是根据相同的`key`来diff，没设置`key`的话，其实是通过数组下标来进行（源码位于`src/renderers/shared/stack/reconciler/ReactChildReconciler.js`）：

```js
  updateChildren: function(
    prevChildren,
    nextChildren,
    mountImages,
    removedNodes,
    transaction,
    hostParent,
    hostContainerInfo,
    context,
    selfDebugID, // 0 in production and for roots
  ) {
    if (!nextChildren && !prevChildren) {
      return;
    }
    var name;
    var prevChild;
    
    // 这里的name实际上就是生成的key
    for (name in nextChildren) {
      if (!nextChildren.hasOwnProperty(name)) {
        continue;
      }
      prevChild = prevChildren && prevChildren[name];
      var prevElement = prevChild && prevChild._currentElement;
      var nextElement = nextChildren[name];
      if (
        prevChild != null &&
        shouldUpdateReactComponent(prevElement, nextElement)
      ) {
        ReactReconciler.receiveComponent(
          prevChild,
          nextElement,
          transaction,
          context,
        );
        nextChildren[name] = prevChild;
      } else {
        if (prevChild) {
          removedNodes[name] = ReactReconciler.getHostNode(prevChild);
          ReactReconciler.unmountComponent(prevChild, false);
        }
        var nextChildInstance = instantiateReactComponent(nextElement, true);
        nextChildren[name] = nextChildInstance;
        var nextChildMountImage = ReactReconciler.mountComponent(
          nextChildInstance,
          transaction,
          hostParent,
          hostContainerInfo,
          context,
          selfDebugID,
        );
        mountImages.push(nextChildMountImage);
      }
    }
    for (name in prevChildren) {
      if (
        prevChildren.hasOwnProperty(name) &&
        !(nextChildren && nextChildren.hasOwnProperty(name))
      ) {
        prevChild = prevChildren[name];
        removedNodes[name] = ReactReconciler.getHostNode(prevChild);
        ReactReconciler.unmountComponent(prevChild, false);
      }
    }
  },
```

这段代码我们之前分析过，要注意这里的循环是`for in`，代表着是从新的children，通过key来索引旧的child进行diff。

在这个函数中，它会执行`receiveComponent`的逻辑，这个我们之前讲过，就是用来更新组件的，要注意的是同样根据`shouldUpdateReactComponent`原则，来进行更新或销毁重新挂载，值得注意的是这里的挂载并不会真正执行DOM操作，而是生成DOM节点存放在`mountImages`中，或是删除节点存放在`removedNodes`中，真正的DOM操作其实是在外面。

最后返回来看一下外面这段代码：

```js
var nextIndex = 0;
var lastIndex = 0;
// `nextMountIndex` will increment for each newly mounted child.
var nextMountIndex = 0;
var lastPlacedNode = null;
for (name in nextChildren) {
  if (!nextChildren.hasOwnProperty(name)) {
    continue;
  }
  var prevChild = prevChildren && prevChildren[name];
  var nextChild = nextChildren[name];

  // 1. 移动节点
  if (prevChild === nextChild) {
    updates = enqueue(
      updates,
      this.moveChild(prevChild, lastPlacedNode, nextIndex, lastIndex),
    );
    lastIndex = Math.max(prevChild._mountIndex, lastIndex);
    prevChild._mountIndex = nextIndex;
  } else {
    if (prevChild) {
      // Update `lastIndex` before `_mountIndex` gets unset by unmounting.
      lastIndex = Math.max(prevChild._mountIndex, lastIndex);
      // The `removedNodes` loop below will actually remove the child.
    }

    // 2. 新增节点
    updates = enqueue(
      updates,
      this._mountChildAtIndex(
        nextChild,
        mountImages[nextMountIndex],
        lastPlacedNode,
        nextIndex,
        transaction,
        context,
      ),
    );
    nextMountIndex++;
  }
  nextIndex++;
  lastPlacedNode = ReactReconciler.getHostNode(nextChild);
}
for (name in removedNodes) {
  if (removedNodes.hasOwnProperty(name)) {
    // 3. 删除节点
    updates = enqueue(
      updates,
      this._unmountChild(prevChildren[name], removedNodes[name]),
    );
  }
}
if (updates) {
  processQueue(this, updates);
}
```

这段代码主要用来识别children最终diff出来要产生的三种行为：

1. 移动（`moveChild`）
2. 新增（`_mountChildAtIndex`）
3. 删除（`_unmountChild`）

先说一下最复杂的移动操作，来看一看`moveChild`的实现：

```js
    moveChild: function(child, afterNode, toIndex, lastIndex) {
      // If the index of `child` is less than `lastIndex`, then it needs to
      // be moved. Otherwise, we do not need to move it because a child will be
      // inserted or moved before `child`.
      if (child._mountIndex < lastIndex) {
        return makeMove(child, afterNode, toIndex);
      }
    },
```

可以看到，`moveChild`的前提条件有两个：

1. `prevChild === nextChild`，这个条件是必然的，因为在之前reconcile的时候我们就已经把`prevChild`更新为`nextChild`了，除非`nextChild`是全新节点或者删除节点的情况。
2. `child._mountIndex < lastIndex`，这个条件就值得思考一下了。

首先，这里的`child._mountIndex`实际上是`prevChild`的，而`lastIndex`则标记新节点上次最大的index，举个例子：

* 旧的children：A-B-C-D
* 新的children：B-C-D-A

这里对于第一个节点B来说，它的`_mountIndex`是1，而`lastIndex`是0，不满足条件，所以B不动，`lastIndex`更新为1。

第二个节点C，`_mountIndex`是2，`lastIndex`是1，不满足条件，所以C不动，`lastIndex`更新为2

第三个节点同理，D也不动。

直到第四个节点A，它的`_mountIndex`是0，而此时`lastIndex`已经是3了，这个时候满足`child._mountIndex < lastIndex`，所以A会移动，移动的目标位置就是`lastIndex`。

这种顺序Diff移动的算法就是React通过`Key`优化Diff的精髓所在了。

新增节点和删除的条件就比较简单了，不再赘述。

这里所有的操作均会被推到队列里面去，最终通过`processQueue`来执行，它的源码位于`src/renderers/dom/client/utils/DOMChildrenOperations.js`：

```js
  processUpdates: function(parentNode, updates) {
    for (var k = 0; k < updates.length; k++) {
      var update = updates[k];
      switch (update.type) {
        case 'INSERT_MARKUP':
          insertLazyTreeChildAt(
            parentNode,
            update.content,
            getNodeAfter(parentNode, update.afterNode),
          );
          break;
        case 'MOVE_EXISTING':
          moveChild(
            parentNode,
            update.fromNode,
            getNodeAfter(parentNode, update.afterNode),
          );
          break;
        case 'SET_MARKUP':
          setInnerHTML(parentNode, update.content);
          break;
        case 'TEXT_CONTENT':
          setTextContent(parentNode, update.content);
          break;
        case 'REMOVE_NODE':
          removeChild(parentNode, update.fromNode);
          break;
      }
    }
  }
```

这里才是真正执行DOM操作的地方，涵盖了插入、移动、删除、变更等多种DOM操作行为。

### 小结一下

React整体的DOM更新与Diff的源码还是十分艰涩与复杂的，总结一下上述的分析，我们举例来说明整个Diff的过程可能更加清晰一些：

#### 第一种情况，DOM元素不同

![](https://files.mdnice.com/user/13429/a2030a61-11f4-4cf5-a78b-eb855d48d828.png)

这种情况肯定是销毁重建，因为按顺序Diff，DOM元素发生了变化：p-span，span-p

#### 第二种情况，DOM元素不同，但相同元素设置了Key

![](https://files.mdnice.com/user/13429/73eb50ad-4ca4-41f0-8370-0af1d1b49c22.png)

当我们设置了`key`属性的时候，情况就发生了变化，Diff算法会依赖于key查找对比，发现虽然它们的位置发生了变化，但内容没有发生变化，**后续只需要移动DOM节点即可，不需要销毁重建！**

#### 同key的移动、删除、新增算法

![](https://files.mdnice.com/user/13429/27a05cb8-2528-4927-81d5-d20ee469bec9.png)

对于同一层级同一类型元素，标注了相同`key`的Diff，就是React的Diff算法最精华聪明之处，可以识别出来组件本身是移动、新增、删除，而不需要按顺序对比导致大量的销毁与DOM重建的工作。（需仔细体会`mountIndex`和`lastIndex`的对比）。

写到这里其实对`React`实现还保留一个疑问，目前React的算法强依赖于`for in`的顺序，虽然在现代浏览器引擎中基本是可以保障的，但理论上可以采取更好的策略，而非依赖于本身无序的Object，ES6的Map其实是一个很好的替代方案。
