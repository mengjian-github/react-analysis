# 详解React中的Transaction事务机制

### 什么是React中的事务

其实`Transaction`这个词对我们开发并不陌生，在数据库中，事务表示的是一个原子化的操作序列，要么全部执行，要么全部不执行，是一个不可分割的工作单位。

我们可以思考一下事务的实现原理，要将多个串行的操作原子化，必然需要在出错的时候，撤销之前操作的能力，也就是需要一个现场保护和还原的机制。

而React之所以取名为`Transaction`，大概也就是因为在它的`initialize`和`close`API中，做到了`close`可以拿到`initialize`的状态的能力，并且对抛出的异常进行比较到位的处理，它的原理如下：

```
 *                       wrappers (injected at creation time)
 *                                      +        +
 *                                      |        |
 *                    +-----------------|--------|--------------+
 *                    |                 v        |              |
 *                    |      +---------------+   |              |
 *                    |   +--|    wrapper1   |---|----+         |
 *                    |   |  +---------------+   v    |         |
 *                    |   |          +-------------+  |         |
 *                    |   |     +----|   wrapper2  |--------+   |
 *                    |   |     |    +-------------+  |     |   |
 *                    |   |     |                     |     |   |
 *                    |   v     v                     v     v   | wrapper
 *                    | +---+ +---+   +---------+   +---+ +---+ | invariants
 * perform(anyMethod) | |   | |   |   |         |   |   | |   | | maintained
 * +----------------->|-|---|-|---|-->|anyMethod|---|---|-|---|-|-------->
 *                    | |   | |   |   |         |   |   | |   | |
 *                    | |   | |   |   |         |   |   | |   | |
 *                    | |   | |   |   |         |   |   | |   | |
 *                    | +---+ +---+   +---------+   +---+ +---+ |
 *                    |  initialize                    close    |
 *                    +-----------------------------------------+
```

可以看到React中实现的`Transaction`其实是`AOP`思想，对一个函数`anyMethod`进行切片包裹`wrapper`，每个`wrapper`可以实现自己的`initialize`和`close`接口，可以嵌套使用。

### 源码分析

> 本文基于React v15.6.2版本介绍，原因请参见新手如何学习React源码

`Transaction`的实现位于`src/renderers/utils/Transaction.js`

```js
perform: function<
    A,
    B,
    C,
    D,
    E,
    F,
    G,
    T: (a: A, b: B, c: C, d: D, e: E, f: F) => G,
  >(method: T, scope: any, a: A, b: B, c: C, d: D, e: E, f: F): G {
  invariant(
    !this.isInTransaction(),
    'Transaction.perform(...): Cannot initialize a transaction when there ' +
      'is already an outstanding transaction.',
  );
  var errorThrown;
  var ret;
  try {
    // 加锁，只允许同时存在一个相同类型的Transaction
    this._isInTransaction = true;
    errorThrown = true;
    // 执行initialize钩子
    this.initializeAll(0);
    // 执行主体函数
    ret = method.call(scope, a, b, c, d, e, f);
    errorThrown = false;
  } finally {
    try {
      if (errorThrown) {
        try {
          // 执行close钩子
          this.closeAll(0);
        } catch (err) {}
      } else {
        // 执行close钩子
        this.closeAll(0);
      }
    } finally {
      this._isInTransaction = false;
    }
  }
  return ret;
},
```

这段代码看起来好像占了一定篇幅，其实去掉那些边边角角的try catch，这段代码核心就变成了三句话：

```js
// 执行initialize钩子
this.initializeAll(0);
// 执行主体函数
ret = method.call(scope, a, b, c, d, e, f);
// 执行close钩子
this.closeAll(0);
```

这三行代码也是`Transaction`实现的主要能力，在主体函数运行前，先运行`initialize`钩子，运行之后，执行`close`钩子。

接下来让我们关注一下实现的细节处理：

* 多个参数的枚举，是React源码的惯用处理手段，为什么不使用`arguments`我在上篇文章中已经解释过了，不做赘述。
* 同一时间只能有一个同类的Transaction在执行，这就是`_isInTransaction`控制锁的作用，也保证了事务运行过程中不被打断。
* 在finally的代码中可以看到，无论前面的`initialize`还是主体函数遇到报错，最后的`close`一定会执行，抛出的错误则以第一个遇到的错误为准。

接下来看一下`initializeAll`的实现：

```js
  initializeAll: function(startIndex: number): void {
    var transactionWrappers = this.transactionWrappers;
    for (var i = startIndex; i < transactionWrappers.length; i++) {
      var wrapper = transactionWrappers[i];
      try {
        this.wrapperInitData[i] = OBSERVED_ERROR;
        // 执行钩子
        this.wrapperInitData[i] = wrapper.initialize
          ? wrapper.initialize.call(this)
          : null;
      } finally {
        if (this.wrapperInitData[i] === OBSERVED_ERROR) {
          try {
            // 继续执行下一个Wrapper的钩子
            this.initializeAll(i + 1);
          } catch (err) {}
        }
      }
    }
  },
```

可以看到`initializeAll`的实现，就是拿到所有的`wrapper`，执行其中的`initialize`钩子，值得注意的是，如果有钩子报错了，剩下的`wrapper`的钩子还是会被执行，结合上面的分析我们可以知道React这样做的原因——保持事务的原子性，有一个操作错误了，需要返回之前的现场，也就是完整的`initialize`和`close`钩子都要走一遍，以撤销之前可能已经做的操作。

`closeAll`的实现与`initializeAll`的实现类似：

```js
  closeAll: function(startIndex: number): void {
    invariant(
      this.isInTransaction(),
      'Transaction.closeAll(): Cannot close transaction when none are open.',
    );
    var transactionWrappers = this.transactionWrappers;
    for (var i = startIndex; i < transactionWrappers.length; i++) {
      var wrapper = transactionWrappers[i];
      var initData = this.wrapperInitData[i];
      var errorThrown;
      try {
        errorThrown = true;
        if (initData !== OBSERVED_ERROR && wrapper.close) {
          // 执行close钩子
          wrapper.close.call(this, initData);
        }
        errorThrown = false;
      } finally {
        if (errorThrown) {
          try {
            // 继续执行下一个Wrapper的close钩子
            this.closeAll(i + 1);
          } catch (e) {}
        }
      }
    }
    this.wrapperInitData.length = 0;
  },
};
```

这里需要注意的是，`close`钩子的传参来源是`this.wrapperInitData`，也就是上一步`initialize`执行的时候的返回值，这样才能够做到对现场的保护还原。

最后看一下`reinitializeTransaction`方法的实现：

```js
  reinitializeTransaction: function(): void {
    this.transactionWrappers = this.getTransactionWrappers();
    if (this.wrapperInitData) {
      this.wrapperInitData.length = 0;
    } else {
      this.wrapperInitData = [];
    }
    this._isInTransaction = false;
  },
```

这个方法比较简单，就是初始化操作，为什么需要这么一个方法呢？我们可以结合前面一篇对象池的文章来思考，`transaction`对象也是可以在对象池中复用的，那么每一次复用，都需要重置一下之前的状态，实际上在React中`transaction`大多也是结合对象池一起用。

### 如何使用

了解原理之后，使用方式就很容易理解了：

```js
const TestTransaction = function() {
  this.reinitializeTransaction();
};
Object.assign(TestTransaction.prototype, Transaction);
TestTransaction.prototype.getTransactionWrappers = function() {
  return [
    {
      initialize: function() {
        console.log('前置函数执行');
        return 'firstResult';
      },
      close: function(initResult) {
        console.log('后置函数执行');
        console.log(initResult);
      },
    }
  ];
};

const transaction = new TestTransaction();
transaction.perform(() => {
  console.log('主体函数执行')
})
```

用法上，构造函数默认调用`reinitializeTransaction`，原型继承自`Transaction`后，挂载`getTransactionWrappers`方法，然后执行`perform`包裹要执行的主体函数就可以了。这个时候主体函数相当于是处于一个事务中执行，会原子化地执行前置和后置函数。

### 在React中的应用

React中的`Transaction`不多，总共就5个，但每一个都是核心中的核心：

* `ReactReconcileTransaction`
* `ReactServerRendingTransaction`
* `ReactNativeReconcileTransaction`
* `ReactDefaultBatchingStrategyTransaction`
* `ReactUpdatesFlushTransaction`

不要小看`Transaction`在React中的地位，上面的`ReactReconcileTransaction`恰恰是`componentDidMount`的关键，而`ReactDefaultBatchingStrategyTransaction`是实现`setState`异步化的关键。限于篇幅，具体的`transaction`我们在后续的应用场景展开介绍。

### 小结一下

React事务实现可以算是React底层的基石，虽然它只是一个utils，但是React很多非常重要的特性都是依赖于事务的。

事务的实现其实不难，可以简单理解为React仅仅是为方法加了前置和后置函数的钩子，并原子化执行函数，只有理解事务机制后，你才不会在React源码中晕头转向，因为React源码的执行顺序跟事务的钩子有极大的关联。

自此开始，我们也真正迈入了React核心实现的大门！
