# React源码学习入门（四）深入探究React中的对象池

>
>
> > 本文基于React v15.6.2版本介绍，原因请参见新手如何学习React源码
>
> ### 源码分析
>
> React对象池的实现在源码的`src/shared/utils/PooledClass.js`，整体实现还是比较简单的，总共就暴露了一个API，和一些针对不同个数参数的处理函数：
>
> ```js
> // 将一个类池化
> var addPoolingTo = function<T>(
>   CopyConstructor: Class<T>,
>   pooler: Pooler,
> ): Class<T> & {
>   getPooled(): T,
>   release(): void,
> } {
>   // 这里拷贝了一份class
>   var NewKlass = (CopyConstructor: any); 
>   // 为传入的class挂了一个pool池
>   NewKlass.instancePool = [];  
>   // 挂了一个获取池子中对象的方法
>   NewKlass.getPooled = pooler || DEFAULT_POOLER;  
>   if (!NewKlass.poolSize) {
>     // 限制池子中最大缓存的对象数量
>     NewKlass.poolSize = DEFAULT_POOL_SIZE;  
>   } 
>   // 挂一个释放对象的方法
>   NewKlass.release = standardReleaser; 
>   return NewKlass;
> };
> ```
>
> `addPoolingTo`这个方法，传入一个`class`，和一个`pooler`，所谓的`pooler`，就是`getPooled`调用的方法，用来获取对象池中缓存的对象，在这里React实现了1-4个参数的`Pooler`：
>
> ```js
> var oneArgumentPooler = function(copyFieldsFrom) {
>   var Klass = this;
>   if (Klass.instancePool.length) {
>     // 如果缓存池中有对象，就直接拿出来用
>     var instance = Klass.instancePool.pop();
>     // 注意这里要重新执行一下构造函数
>     Klass.call(instance, copyFieldsFrom);
>     return instance;
>   } else {
>     // 没有的话，就new一个新的
>     return new Klass(copyFieldsFrom);
>   }
> };
>
> var twoArgumentPooler = function(a1, a2) {
>   var Klass = this;
>   if (Klass.instancePool.length) {
>     var instance = Klass.instancePool.pop();
>     Klass.call(instance, a1, a2);
>     return instance;
>   } else {
>     return new Klass(a1, a2);
>   }
> };
>
> // 省略其他参数的pooler实现，原理都是一样的
> ```
>
> 最后看一下`release`的实现：
>
> ```js
> var standardReleaser = function(instance) {
>   var Klass = this;
>   invariant(
>     instance instanceof Klass,
>     'Trying to release an instance into a pool of a different type.',
>   );
>   // 释放前执行destructor方法
>   instance.destructor();  
>   if (Klass.instancePool.length < Klass.poolSize) {
>     // 返回对象池复用
>     Klass.instancePool.push(instance);  
>   }
> };
> ```
>
> `release`方法就是将对象返回到对象池，以便下一次的复用，这里注意React实现时的几点小细节：
>
> 1. 校验了释放的对象是否是属于这个类的，避免释放错对象。
> 2. 在释放前，调用了`destructor`方法，这个也强制要求了被池化的类需要实现这个方法，一般在方法内会清除一些变量的引用和回收工作。
> 3. 根据对象池的最大限制添加，若当前对象池已满，就不再回到池子里了。
>
> ### 使用方式
>
> 了解了React对象池的实现原理，那使用方式就显而易见了：
>
> ```js
> // 要被池化的class
> class Poolable {
>   constructor(xxx) {
>     this.xxx = xxx;
>   }
>   
>   destructor() {
>     this.xxx = null;
>   }
> }
> // 池化
> PoolClass.addPoolingTo(Poolable)
> // 实例化对象，这里不用自己new，而是向对象池申请：
> const instance = Poolable.getPooled();
> // 使用instance
> // 省略
> // 使用完成，释放归还实例给对象池
> Poolable.release(instance);
> ```
>
> 对象池在这一版的代码里面使用场景还是挺广泛的，主要的使用场景总结如下：
>
> * `ReactChildren`
> * `ReactEventListener`
> * `ReactReconcileTransaction`
> * `FallbackCompositionState`
> * `ReactServerRenderingTransaction`
> * `ReactNativeReconcileTransaction`
> * `SyntheticEvent`
> * `ReactUpdatesFlushTransaction`
> * `CallbackQueue`
>
> ### 思考：为什么React要重复实现不同参数的pooler？
>
> 读这一版源码的时候，一直有个疑问，针对不同参数，就算之前没有Rest特性的支持，在很早期的ES规范中就支持使用`arguments`，为啥还要大费周章地枚举出这么多参数个数产生这么多重复的函数呢？
>
> 在这个[issue](https://github.com/facebook/react/issues/9325)有人问了同样的问题。
>
> 目前的维护者回答是早期的实现者可能认为`arguments`的实现会在低版本浏览器存在性能问题，因此很多时候支持多参数都是通过枚举来做的。
>
> ### 思考：对象池的意义是什么？
>
> 一般来说，对象池在游戏场景用到的比较多，通过查阅大量资料，我总结它的作用主要是以下两点：
>
> 1. 对象的创建和销毁比较消耗性能，使用对象池可以尽可能降低性能损耗。
> 2. 对于大量频繁的创建对象操作，使用对象池可以有效减少GC的压力，避免每次GC耗时加剧影响到应用的性能。
>
> 很显然，在游戏场景下，是第一类场景，往往创建一个新的`Sprite`是十分消耗性能的；而在React中，考虑的则是第二类场景，可以看到在React的事件机制、渲染、更新机制，都加入了对象池，在此类场景下，有可能对象会在短时间内频繁地触发。
>
> ### 思考：现代JS中真的需要对象池吗？
>
> 这个主要针对上述的第二点，也就是高频快速地进行对象的创建行为来讨论。
>
> 实际上，在React 17版本中是去除了`PooledClass`的实现的，具体信息可以参考[这里](https://reactjs.org/blog/2020/08/10/react-v17-rc.html#no-event-pooling)。
>
> 因为对象池的机制，经常导致React中的`event`在下个事件循环中被释放的情况，不得不使用`persist`方法去阻止对象的释放回收，对象池给React用户带来了一些负担。
>
> 另外，React团队认为在现代浏览器中，对象池的实现机制并不能带来性能提升，收益非常小，因此最终在17版本移除。
>
> 为什么说在现代的浏览器中可以不使用对象池技术呢？React官方没有给出明确的回答，不过我们可以从GC的机制来分析下：
>
> 我们知道，现代浏览器中，实现垃圾回收基本上都采用的是标记清除的机制，从Root节点往下寻找，清除掉没有被标记，也就是不存在引用的变量。
>
> 而V8针对GC做了大量优化，其中一个很重要的优化是`分代式垃圾回收`：
>
> <img src="https://tva1.sinaimg.cn/large/e6c9d24egy1h5r7h0oqz7j213y0dwgo3.jpg" alt="" data-size="original">
>
> V8在堆内存中开辟出新生代和老生代的划分区，分代式机制把一些新、小、存活时间短的对象作为新生代，采用一小块内存频率较高的快速清理，而一些大、老、存活时间长的对象作为老生代，使其很少接受检查，这样来提高整个GC的效率。
>
> 之所以JS的GC会影响到渲染性能，本质原因还是单线程引起的，所以V8针对新生代开启了多线程机制来辅助执行GC，这样就大大减少了对主线程的阻塞时间：
>
> <img src="https://tva1.sinaimg.cn/large/e6c9d24egy1h5r7h1oefej214s0a474s.jpg" alt="" data-size="original">
>
> 基于V8的上述两点主要优化可以看到，对于小的对象创建，实际上GC的压力已经不再是瓶颈了，将老生代剥离出去和多线程的机制，已经让GC是一个非常轻量的过程，而JS创建对象的数量始终是有限的，另外V8启用lazy sweeping机制，可以很好地应对绝大多数情况，所以在目前看来，在大多数应用中，使用JS的对象池技术是没有太大必要的。
>
> ### 小结一下
>
> React内部的对象池，在早期的源码中得到了广泛的应用，虽然JS作为高级语言是自动进行垃圾回收的，但并不代表我们可以不关注内存，作为一个成千上万人使用的基础库来说，性能是十分重要的，这也是为什么各大UI框架为什么那么注重benchmark。
>
> 虽然在现代浏览器中对象池也许没有那么重要了，但这个思想却是非常值得学习的通用思想，很多诸如连接池复用的思想都是类似的，在其他场景下还是有很多用武之地。
