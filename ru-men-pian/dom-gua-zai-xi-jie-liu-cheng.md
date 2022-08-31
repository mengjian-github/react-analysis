# DOM挂载细节流程



> 本文基于React v15.6.2版本介绍，原因请参见新手如何学习React源码

### 源码分析

React挂载DOM的核心流程在`src/renderers/dom/shared/ReactDOMComponents.js`下：

```js
  mountComponent: function(
    transaction,
    hostParent,
    hostContainerInfo,
    context,
  ) {
    this._rootNodeID = globalIdCounter++;
    this._domID = hostContainerInfo._idCounter++;
    this._hostParent = hostParent;
    this._hostContainerInfo = hostContainerInfo;

    var props = this._currentElement.props;

    var mountImage;
    
    if (transaction.useCreateElement) {
      var ownerDocument = hostContainerInfo._ownerDocument;
      var el;
      
      // 1. 创建一个DOM元素
      if (namespaceURI === DOMNamespaces.html) {
        if (this._tag === 'script') {
          var div = ownerDocument.createElement('div');
          var type = this._currentElement.type;
          div.innerHTML = `<${type}></${type}>`;
          el = div.removeChild(div.firstChild);
        } else if (props.is) {
          el = ownerDocument.createElement(this._currentElement.type, props.is);
        } else {
          el = ownerDocument.createElement(this._currentElement.type);
        }
      } else {
        el = ownerDocument.createElementNS(
          namespaceURI,
          this._currentElement.type,
        );
      }
      
      // 2. precache DOM元素
      ReactDOMComponentTree.precacheNode(this, el);
      this._flags |= Flags.hasCachedChildNodes;
      if (!this._hostParent) {
        DOMPropertyOperations.setAttributeForRoot(el);
      }
      
      // 3. 更新DOM元素的Properties
      this._updateDOMProperties(null, props, transaction);
      var lazyTree = DOMLazyTree(el);
      
      // 4. 创建children
      this._createInitialChildren(transaction, props, context, lazyTree);
      mountImage = lazyTree;
    } else {
      var tagOpen = this._createOpenTagMarkupAndPutListeners(
        transaction,
        props,
      );
      var tagContent = this._createContentMarkup(transaction, props, context);
      if (!tagContent && omittedCloseTags[this._tag]) {
        mountImage = tagOpen + '/>';
      } else {
        mountImage =
          tagOpen + '>' + tagContent + '</' + this._currentElement.type + '>';
      }
    }

    return mountImage;
  },
```

这里省略了一系列针对特殊DOM（如表单元素的Focus）处理，通用的DOM挂载实际上做了以下几件事：

1. 创建对应的DOM元素（在之前的版本是通过字符串的拼接的方式，后面出于性能考虑改为createElement）
2. precache这个元素，便于在更新时候能够找到
3. 更新DOM元素的Properties
4. 创建children并挂载children(这是一个递归过程)

其中3和4就是DOM元素挂载的重头戏了，下面详细阐述一下：

#### 更新DOM元素的Properties

```js
  _updateDOMProperties: function(lastProps, nextProps, transaction) {
    var propKey;
    var styleName;
    var styleUpdates;
    
    for (propKey in nextProps) {
      if (propKey === STYLE) {
        if (nextProp) {
          nextProp = this._previousStyleCopy = Object.assign({}, nextProp);
        } else {
          this._previousStyleCopy = null;
        }
        styleUpdates = nextProp;
      } else if (registrationNameModules.hasOwnProperty(propKey)) {
        if (nextProp) {
          enqueuePutListener(this, propKey, nextProp, transaction);
        }
      } else if (isCustomComponent(this._tag, nextProps)) {
        if (!RESERVED_PROPS.hasOwnProperty(propKey)) {
          DOMPropertyOperations.setValueForAttribute(
            getNode(this),
            propKey,
            nextProp,
          );
        }
      } else if (
        DOMProperty.properties[propKey] ||
        DOMProperty.isCustomAttribute(propKey)
      ) {
        var node = getNode(this);
        // If we're updating to null or undefined, we should remove the property
        // from the DOM node instead of inadvertently setting to a string. This
        // brings us in line with the same behavior we have on initial render.
        if (nextProp != null) {
          DOMPropertyOperations.setValueForProperty(node, propKey, nextProp);
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

这里更新Property的行为我们忽略了update的部分，只看初次挂载，update后续会专门分析，其实挂载的属性分为几种类型：

* style标签，会处理空字符串的情况（删除CSS属性）
* 事件，在React中已注册的事件，以onXXX开头，这个是要交给事件中心去处理的
* DOM本身的属性，通过setAttribute设置，同样处理了空属性的问题

#### 创建并挂载children

第二个重点就是创建并挂载children的过程了，我们知道在React的JSX写法中，DOM元素的Child可以是React组件、DOM元素等等多种类型，所以这里理论上也是一个递归的过程，交给`Reconciler`的mount来重新处理：

```js
  _createInitialChildren: function(transaction, props, context, lazyTree) {
    var innerHTML = props.dangerouslySetInnerHTML;
    if (innerHTML != null) {
      if (innerHTML.__html != null) {
        // 直接设置innerHTML
        DOMLazyTree.queueHTML(lazyTree, innerHTML.__html);
      }
    } else {
      var contentToUse = CONTENT_TYPES[typeof props.children]
        ? props.children
        : null;
      var childrenToUse = contentToUse != null ? null : props.children;
      if (contentToUse != null) {
        if (contentToUse !== '') {
          // 直接设置textContent
          DOMLazyTree.queueText(lazyTree, contentToUse);
        }
      } else if (childrenToUse != null) {
        // 挂载children
        var mountImages = this.mountChildren(
          childrenToUse,
          transaction,
          context,
        );
        // 挂载DOM
        for (var i = 0; i < mountImages.length; i++) {
          DOMLazyTree.queueChild(lazyTree, mountImages[i]);
        }
      }
    }
  },
```

这里的`DOMLazyTree`在前面介绍过，是为了更好地性能专门处理IE的问题，我们可以简单理解为`queueText`实际上就是设置`textContent`，`queueChild`实际上就是执行`appendChild`，`queueHTML`对应`innerHTML`。

这里的逻辑比较简单，大概归纳如下：

* 设置了`dangerouslySetInnerHTML.__html`的，不管子元素，直接使用innerHTML覆盖子元素内容。
* 如果子元素是一个字符串或者数字，那直接设置textContent
* 否则，遍历整个children，执行`mountChildren`，并最终append到DOM上。

最后我们来看一看`mountChildren`，它的实现位于`src/renderers/shared/reconciler/ReactMultiChild.js`：

```js
mountChildren: function(nestedChildren, transaction, context) {
  var children = this._reconcilerInstantiateChildren(
    nestedChildren,
    transaction,
    context,
  );
  this._renderedChildren = children;

  var mountImages = [];
  var index = 0;
  for (var name in children) {
    if (children.hasOwnProperty(name)) {
      var child = children[name];
      var selfDebugID = 0;
      if (__DEV__) {
        selfDebugID = getDebugID(this);
      }
      var mountImage = ReactReconciler.mountComponent(
        child,
        transaction,
        this,
        this._hostContainerInfo,
        context,
        selfDebugID,
      );
      child._mountIndex = index++;
      mountImages.push(mountImage);
    }
  }
    
  return mountImages;
}
```

这里面做的事情非常简单，就两个步骤：

1. 初始化每个child的实例（回忆一下我们在初始挂载的方法里也是做这个事情）
2. 对每个child执行`mountComponent`，拿到markup（images）列表返回

至此，整个DOM挂载的过程就结束了，生成的这个DOM节点最终通过之前我们提到的mountImage挂载到Container上。

### 小结一下

这里就是React实质挂载执行的叶子节点了，实际上也是DOM树真正形成的起点，当然结合之前我们提到的React组件的挂载流程，实际上就会发现最终能够挂载到DOM上的元素就是这里创建生成的DOM节点，或者是`EmptyComponent`的注释类节点和`TextComponent`生成的文本节点，整体DOM的挂载流程可以总结为下图：

![](https://files.mdnice.com/user/13429/c83ee05e-a680-48e1-b982-7befdaa4b888.png)
