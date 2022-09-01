# React源码学习入门（一）新手如何学习React源码

众所周知，对于前端开发来说，React现在已经是非常流行的深受大众喜爱的框架，在我们日常中使用非常广泛：

![](https://tva1.sinaimg.cn/large/e6c9d24egy1h5r7aazcn1j21ff0u0ad7.jpg)

学习React源码可以帮助我们更深入地理解React背后的原理机制，让我们更高效和快速地编写代码，同时也是高级工程师深入技术研究的一个非常好的方向，有助于提升个人的技术深度。

但是，不得不说React源码读起来是十分困难和陡峭的，这源自于React团队本身不断创新和复杂，尤其是Fiber架构采用了之后，原本就难以理解的代码更是雪上加霜。

因此从本文开始，我将开启一个React源码学习的专栏。从一个完全新手的视角来分析阐述React背后的运作原理，包括但不限于：

* 由浅入深地讲解React源码运行机制
* 通过UML图辅助分析源码
* 单模块的拆解分析
* Stack和Fiber架构独立讲解
* 核心算法辅助图解
* 动手实现自己的mini React版本

对于新手来说，如何更好地学习React源码，有几点建议：

### 1. 良好的前端基础

深入到源码层次，首先需要打好必备的前端基础，对于HTML、JS的基础知识有系统的掌握，对前端构建体系、工程化、模块化、ES Next语法有一定的了解和认知，最好有1年以上的前端开发经验。否则读起源码可能会比较吃力。

另外，你需要熟练地使用过React，通读React官方文档，了解React各个特性的使用方式，不然在分析源码时可能会因为对功能的不熟悉而增大理解难度。

### 2. 建议从15.6.2版本开始

这个版本实际上也是React 15的最后一个版本，大家都知道自从React 16开始React底层架构经过了很大的变更，整体切换到Fiber上，而Fiber本身是React团队酝酿2年之久的设计，本身是较为复杂的。

新人阅读源码不建议从最新的Fiber版本入手，原因如下：

* Fiber架构较复杂，React团队酝酿2年之久才完成的代码，会让人望而却步。
* React的核心概念是渲染、更新、事件机制，虽然Fiber是一种非常前沿的实验性尝试，但暂时性不看Fiber，能让我们更专注于React其他方面的实现，再返回来看Fiber就会轻松很多。
* Stack的架构亦是经典，先有同步，而后才可能异步，同步模式也是最容易理解的方式，我们可以比较轻松地实现自己的Stack的版本，对React能有更深入的认知。

![](https://tva1.sinaimg.cn/large/e6c9d24egy1h5r7ac8350j20zu0kkgnk.jpg)

如上图所示，我们应该让自己处于舒适区的边缘，此时的成长进步才最快。一上来就看Fiber的源码，仿佛就是把自己推向了困难区，真正的从入门到放弃了。

### 3. 保持耐心

学习源码，其实本质上就是在上手一个Codebase，只是这个Codebase可能比平时做的项目要更加地复杂和抽象，所以不能急于求成，要给予自己一些时间和耐心。

我相信没有人能够在1天内就可以看懂React源码，也没有人可以通读源码不遇到障碍和问题，无论是多么有经验的专家，也需要一些时间和思考的空间。

往往分析一条链路，我们需要经过十几遍的从头调用来记录和分析作者的编码思想，这时唯一需要保持的就是耐心与坚持，最终肯定是可以达成目标的。

### 4. 动手实践

![](https://tva1.sinaimg.cn/large/e6c9d24egy1h5r7ad1xufj20fi0c03z0.jpg)

大家都知道学习金字塔中阅读是处于顶端的，如果我们只是读一读源码，它只能够成为你的短期记忆知识，并不能内化为自己的能力。

因此本专栏里会包含一个动手实践mini React环节，只有真正自己去做了，才可以内化为自己的能力，更深刻地认识其中的细节。
