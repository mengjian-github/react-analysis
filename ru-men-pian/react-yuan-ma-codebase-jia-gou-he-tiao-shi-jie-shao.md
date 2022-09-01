# React源码学习入门（三）React源码codebase架构和调试介绍

> 本文基于v15.6.2版本介绍，原因请参见新手如何学习React源码

### 代码目录结构

```
.
├── Gruntfile.js 
├── README.md
├── docs
├── examples
├── grunt
├── gulp
├── gulpfile.js
├── package.json
├── scripts
├── src # 源代码
    ├── ReactVersion.js
    ├── addons # 插件目录
    ├── isomorphic # React core导出实现
    │   ├── React.js
    │   ├── __tests__
    │   ├── children
    │   ├── classic # 经典方式使用（React.createClass）
    │   ├── getNextDebugID.js
    │   ├── hooks
    │   └── modern # 现代方式使用（React.Component）
    ├── package.json
    ├── renderers # 渲染器实现
    │   ├── art
    │   ├── dom # dom实现
    │   ├── native # native实现
    │   ├── noop
    │   ├── shared  # 公共协调代码，包含了Stack和Fiber两种模式
    │   └── testing
    ├── shared
    │   ├── types
    │   ├── utils  # 公共工具库
    │   └── vendor
    ├── test
    └── umd
└── yarn.lock
```

React整体的源码在`src`目录下，我们重点需要关注的几个目录：

* `isomorphic`，存放React本身的API，如`createElement`、`Component`等。
* `renderers`，存放React核心渲染逻辑
* `renderers/dom`，封装DOM事件、DOM更新、DOM挂载等行为，跟DOM有关的核心逻辑
* `renderers/shared`，公共目录，主要存放native和dom的公共行为，也是React核心的实现
* `renderers/stack`，与Fiber相对，React之前render的核心策略就是通过函数递归调用来实现的，因此命名为`stack`，这里面包含了事件模型、Diff算法等的核心实现
* `utils`，存放公共方法，非常重要的`Transaction`机制就放在这里。

### 宏观架构

![](https://tva1.sinaimg.cn/large/e6c9d24egy1h5r7h966efj21160u0q6x.jpg)

React核心的部分其实是由入口、核心协调器、事件中心、DOM渲染器来实现的，后续的文章我们会从渲染挂载、更新、事件触发等角度详细剖析内部的原理。

### 调试代码

15.6版本React是基于gulp构建的，因此调试源码比较简单，步骤如下：

```
# 1. 安装依赖（yarn v1版本）
yarn
# 2. 执行构建（node v7版本）
yarn build
```

构建产物生成后，我们可以直接在`examples/`目录的index.html打入断点，这样直接打开HTML文件就可以在浏览器调试了：

![](https://tva1.sinaimg.cn/large/e6c9d24egy1h5r7ha0mnbj21jk0e7q7g.jpg)

### 小结一下

在查看React源码前，第一步要做的事情是大致了解整个Codebase的情况，目录结构，调试方法，本文作为一个开始准备过程，帮助大家能够更顺利地走进React源码世界，下一步就可以尽情地分析源码了！
