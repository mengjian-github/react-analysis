# React的render究竟返回的是什么？

在进行React源码的深度讲解之前，我们先来看看一个最基础的核心问题：

> ❝
>
> React render的返回值到底是什么？
>
> ❞

理解这个问题，才能顺利完成对React源码的进一步分析。

### React render的返回值类型

其实要回答这个问题很简单，我们只需要看一下React官方TS声明的类型：

```
class Component<P, S> {  render(): ReactNode;}type ReactNode = ReactElement | string | number | ReactFragment | ReactPortal | boolean | null | undefined;
```

[声明源文件](https://github.com/DefinitelyTyped/DefinitelyTyped/blob/master/types/react/index.d.ts)

可以很明显的看出来，render返回值是一个`ReactNode`，而`ReactNode`可以是很多类型，其中最重要常见的类型是`ReactElement`。

### JSX的编译

我们一般在写React的时候，一般是这样写的：

```
render() {  return (    <button onClick={this.handleClick}>      Click me! Number of clicks: {this.state.count}    </button>  );}
```

在JS里面书写类似HTML标签的语法，是React团队早期定义的一个JavaScript的语法扩展，名叫[JSX](https://zh-hans.reactjs.org/docs/introducing-jsx.html)。

这个语法在经过babel编译时，会产生如下结果：

```
render() {  return /*#__PURE__*/React.createElement("button", {    onClick: this.handleClick  }, "Click me! Number of clicks: ", this.state.count);}
```

[在babel平台上直接编译](https://babeljs.io/repl)

可以看到，我们书写的HTML标签、React组件，最终都会被编译成`React.createElement`的方法调用，而render的返回值，正是这个函数的返回值。

### createElement原理解析

> ❝
>
> 以下代码分析源自于React v15.6.2版本，原因可参考新手如何学习React源码
>
> ❞

`createElement`的实现位于`src/isomorphic/classic/element/ReactElement.js`:

```
ReactElement.createElement = function(type, config, children) {  // ...}
```

我们先来看看它的入参，它支持三个参数：

* **「type」**，这个其实就是标签本身，如果是个HTML标签，那它是字符串，比如上面的`'button'`，如果是一个React组件，那它就是这个组件的`class/function`本身。
* **「config」**，这个是标签上的属性对象，对于React组件来说，其实可以简单理解为它的`props`，对于HTML元素来说，是它的`attribute`所构成的对象。
* **「children」**，顾名思义就是它的子元素了，children的类型同样也是一个`ReactNode`，由`createElement`生成。注意children参数可以再往后扩展，第三个参数之后的传参全部都会被视为children。

这个时候再看下面的代码就会清晰很多：

```
React.createElement("button", {  onClick: this.handleClick}, "Click me! Number of clicks: ", this.state.count);
```

现在来看看`createElement`的实现，首先，是props的生成：

```
ReactElement.createElement = function(type, config, children) {  // 以下为生成props的片段  if (config != null) {    if (hasValidRef(config)) {      ref = config.ref;    }    if (hasValidKey(config)) {      key = '' + config.key;    }    self = config.__self === undefined ? null : config.__self;    source = config.__source === undefined ? null : config.__source;    // Remaining properties are added to a new props object    for (propName in config) {      if (        hasOwnProperty.call(config, propName) &&        !RESERVED_PROPS.hasOwnProperty(propName)      ) {        props[propName] = config[propName];      }    }  }}
```

这段代码并不复杂，我们忽略内部变量`self`和`source`，其实这段代码就是从`config`中提取生成了以下属性：

* key，也就是React中的key属性
* ref，也就是React中的ref属性
* props，剩下的config被拷贝到props对象上

其次是children的生成：

```
ReactElement.createElement = function(type, config, children) {  // children的生成  var childrenLength = arguments.length - 2;  if (childrenLength === 1) {    props.children = children;  } else if (childrenLength > 1) {    var childArray = Array(childrenLength);    for (var i = 0; i < childrenLength; i++) {      childArray[i] = arguments[i + 2];    }    if (__DEV__) {      if (Object.freeze) {        Object.freeze(childArray);      }    }    props.children = childArray;  }}
```

这段代码同样也非常简单，就是把第三个参数和之后的参数，全部合并到`props`的`children`属性上。

### Virtual DOM的表示

最终`createElement`的返回值，其实是一个`ReactElement`对象：

```
ReactElement.createElement = function(type, config, children) {  // 返回值  return ReactElement(    type,    key,    ref,    self,    source,    ReactCurrentOwner.current,    props,  );}var ReactElement = function(type, key, ref, self, source, owner, props) {  var element = {    // This tag allow us to uniquely identify this as a React Element    $$typeof: REACT_ELEMENT_TYPE,    // Built-in properties that belong on the element    type: type,    key: key,    ref: ref,    props: props,    // Record the component responsible for creating this element.    _owner: owner,  };  return element;};
```

可以看到，`ReactElement`就是一个非常普通的对象，包含了`type`、`props`等这些关键属性，而这个对象就是React内部render返回的实际底层表示。

这样一个对象的形式，完全包含了ReactNode想要表达的所有信息，我们也可以称它为一个标记（token），更进一步，它可以理解为一种`DSL`。

而React通过这层`DSL`的抽象表示，构建了整个组件的嵌套树，我们称之为`Virtual DOM`，通过这样的数据结构，React屏蔽了DOM和Natvie在具体实现上的差异，做到了跨端跨平台的通用处理，也正是得益于这样的表示，让`SSR`成为了可能，同时有了高效的Diff算法提升整体更新的性能。

不得不说，在2013年React团队就能提出这样的思想和实现，十分令人敬佩，也同样开启了前端一个崭新的时代。

### 一句话总结

回到标题的问题：

* React的render究竟返回的是什么？

本质上，它返回的就是一个`ReactElement`，一个普普通通的对象，通过这些对象，React构建出了大名鼎鼎的`Virtual DOM`，从而开启了前端新纪元。
