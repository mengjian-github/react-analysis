# React源码学习进阶（一）新版React如何调试源码？

> React 16版本之后，对源码架构进行了较大的升级调整，项目从gulp/grunt迁移到rollup，采用多包构建的方式组织代码，我们常常debug的是打包后的文件，本文可以解决我们想debug到源码的问题。

### 使用`create-react-app`创建项目

```
npx create-react-app react-debug
```

此时，我们如果打一个`debugger`，会发现调用堆栈是`bundle.js`：

![image-20220902202201589](https://tva1.sinaimg.cn/large/e6c9d24egy1h5sj2n51uzj20ls0sy423.jpg)

我们先启用VSCode的调试模式，在项目下新建一个`launch.json`（注意我这里cra启动的端口是3001）：

```js
{
  // Use IntelliSense to learn about possible attributes.
  // Hover to view descriptions of existing attributes.
  // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
  "version": "0.2.0",
  "configurations": [
    {
      "type": "pwa-chrome",
      "request": "launch",
      "name": "Launch Chrome against localhost",
      "url": "http://localhost:3001",
      "webRoot": "${workspaceFolder}"
    }
  ]
}
```

启动调试就可以看到我们已经可以通过VSCode来调试了：

![image-20220902202630782](https://tva1.sinaimg.cn/large/e6c9d24egy1h5sj7b8t8ej21360f2whm.jpg)

但是目前还只能调试打包后的代码，我们需要定位到源码。

### 下载React源码

```
git clone https://github.com/facebook/react.git
```

然后我们在React下编译一下代码：

```
yarn
yarn build
```

接着我们去外层`eject`一下，方便调整`webpack`配置：

```
npm run eject
```

然后在`config/webpack.config.js`下增加`alias`：

```js
alias: {
        'react-dom$': path.resolve(__dirname, '../react/build/node_modules/react-dom/umd/react-dom.development.js'),
        'react$': path.resolve(__dirname, '../react/build/node_modules/react/umd/react.development.js'),
}
```

这时跑起来我们发现有报错：

![image-20220902205300771](https://tva1.sinaimg.cn/large/e6c9d24egy1h5sjyxc2saj20n009sac3.jpg)

这个是因为`ModuleScopePlugin`的限制，去配置里把这个插件删掉，我们就发现堆栈走到了最新的`react`代码里：

![image-20220902205546585](https://tva1.sinaimg.cn/large/e6c9d24egy1h5sk1re8ihj212u0gq422.jpg)

但是目前我们还是在`development.js`里面，想要定位到源码，还需要生成`sourceMap`。

### 支持`sourceMap`

首先我们将`vscode`的`sourcemap`打开，在`launch.json`中加入配置：

```
"sourceMaps": true
```

然后在`react`源码编译时，加入`sourceMap`，更改`react/scripts/rollup/build.js`：

```js
function getRollupOutputOptions(
  outputPath,
  format,
  globals,
  globalName,
  bundleType
) {
  const isProduction = isProductionBundleType(bundleType);

  return {
    file: outputPath,
    format,
    globals,
    freeze: !isProduction,
    interop: false,
    name: globalName,
    sourcemap: true,
    esModule: false,
  };
}
```

然后重新运行`build`，会发现如下报错：

![image-20220902210001644](https://tva1.sinaimg.cn/large/e6c9d24egy1h5sk66ittfj20p604ugm7.jpg)

这是因为有些`rollup`插件不支持`sourcemap`，在文件里面去掉以下4个插件：

```js
    // {
    //   transform(source) {
    //     return source.replace(/['"]use strict["']/g, '');
    //   },
    // }

    // isProduction &&
    //   closure(
    //     Object.assign({}, closureOptions, {
    //       // Don't let it create global variables in the browser.
    //       // https://github.com/facebook/react/issues/10909
    //       assume_function_wrapper: !isUMDBundle,
    //       renaming: !shouldStayReadable,
    //     })
    //   )

    // shouldStayReadable &&
    //   prettier({
    //     parser: 'babel',
    //     singleQuote: false,
    //     trailingComma: 'none',
    //     bracketSpacing: true,
    //   })

    // {
    //   renderChunk(source) {
    //     return Wrappers.wrapBundle(
    //       source,
    //       bundleType,
    //       globalName,
    //       filename,
    //       moduleType
    //     );
    //   },
    // },
```

这个时候就可以build起来了。我们可以看到map文件已经生成到目录下：

![image-20220902211113714](https://tva1.sinaimg.cn/large/e6c9d24egy1h5skhu0hyoj20ec09omxv.jpg)

此时我们在VSCode中开启`debug`就完全可以看到源码堆栈了：

![image-20220902212829836](https://tva1.sinaimg.cn/large/e6c9d24egy1h5skzsq82vj20yq0iyq6j.jpg)

（如果看不到的话，确认下create-react-app中引用`react-dom`的地方，要把`/client`去掉）。
