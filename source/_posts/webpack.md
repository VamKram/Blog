title: Webpack KING！！ (一)
date: 2019/08/15
cover: https://raw.githubusercontent.com/webpack-contrib/awesome-webpack/master/media/awesome_webpack_branding.png
categories:
- tool
tags:
- webpack

---
# Webpack KING！！ (一)

## source map

webpack 中的几种不同的模式

| devtool                   | performance                              | production | quality        | comment                                                      |
| :------------------------ | :--------------------------------------- | :--------- | :------------- | :----------------------------------------------------------- |
| (none)                    | **build**: fastest  **rebuild**: fastest | yes        | bundle         | Recommended choice for production builds with maximum performance. |
| **`eval`**                | **build**: fast  **rebuild**: fastest    | no         | generated      | Recommended choice for development builds with maximum performance. |
|                           |                                          |            |                |                                                              |
|                           |                                          |            |                |                                                              |
| **`eval-source-map`**     | **build**: slowest  **rebuild**: ok      | no         | original       | Recommended choice for development builds with high quality SourceMaps. |
| `cheap-source-map`        | **build**: ok  **rebuild**: slow         | no         | transformed    |                                                              |
| `cheap-module-source-map` | **build**: slow  **rebuild**: slow       | no         | original lines |                                                              |
| **`source-map`**          | **build**: slowest  **rebuild**: slowest | yes        | original       | Recommended choice for production builds with high quality SourceMaps. |

resolve

```json
resolve: {
modules: [path.resolve('node_modules')],
alias: {
"@": "./"
},
extensions: [".js", ".css"]
}
```

### 优化点

module.noParse

```
RegExp` `[RegExp]` `function(resource)` `string` `[string]
```

防止 webpack 解析那些任何与给定正则表达式相匹配的文件。忽略的文件中 **不应该含有** `import`, `require`, `define` 的调用，或任何其他导入机制。忽略大型的 library 可以提高构建性能。

**include/exclude**

引入符合以下任何条件的模块。如果你提供了 `Rule.include` 选项，就不能再提供 `Rule.resource`。

排除所有符合条件的模块。如果你提供了 `Rule.exclude` 选项，就不能再提供 `Rule.resource`

**webpack.IgnorePlugin()**

- `resourceRegExp`: A RegExp to test the resource against.
- `contextRegExp`: (optional) A RegExp to test the context (directory) against.

```javascript
new webpack.IgnorePlugin({ resourceRegExp, contextRegExp });
// Supported in webpack 4 and earlier, unsupported in webpack 5:
new webpack.IgnorePlugin(resourceRegExp, [contextRegExp]);
```

**DLLPlugin**

```javascript

{
  entry: {
    react: ['react', 'react-dom']
  },
    output: {
      filename: '[name].js',
      path: path.resolve(__dirname, 'build'),
      library: '[name]'
    },
      plugins: [
        new webpack.DllPlugin({
          name: '[name]',
          path: path.resolve(__dirname, 'build', 'manifest.json')
        })
      ]
}

----

new DLLReferencePlugins({
  manifest: path.resolve(__dirname, 'build', 'manifest.json')
})
```

**happyPack**

> 多线程打包

```javascript
use: 'Happypack/loader?id=optmize'
new HappyPack({
{id: 'optmize'},
use: [
{
loader: 'babel-loader',
options: {
presets: [
'@babel/preset-env'
]
}
}
]
})
```

### tree shaking

> require模块会讲export的内容放在Module.default上，故不支持tree shaking。

### scope hosting

>  自动简化代码

### Optimization

## `optimization.minimize` 

```
boolean = true
```

告知 webpack 使用 [TerserPlugin](https://webpack.docschina.org/plugins/terser-webpack-plugin/) 或其它在 [`optimization.minimizer`](https://webpack.docschina.org/configuration/optimization/#optimizationminimizer) 定义的插件压缩 bundle。

## `optimization.removeEmptyChunks` 

```
boolean = true
```

如果 chunk 为空，告知 webpack 检测或移除这些 chunk。将 `optimization.removeEmptyChunks` 设置为 `false` 以禁用这项优化。

## `optimization.mergeDuplicateChunks` 

```
boolean = true
```

告知 webpack 合并含有相同模块的 chunk。将 `optimization.mergeDuplicateChunks` 设置为 `false` 以禁用这项优化。

## `optimization.providedExports` 

```
boolean
```

告知 webpack 去确定那些由模块提供的导出内容，为 `export * from ...` 生成更多高效的代码。默认 `optimization.providedExports` 会被启用。

```javascript
const TerserPlugin = require('terser-webpack-plugin');

module.exports = {
  optimization: {
    minimizer: [
      new TerserPlugin({
        parallel: true,
        sourceMap: true, // 如果在生产环境中使用 source-maps，必须设置为 true
        terserOptions: {
          // https://github.com/webpack-contrib/terser-webpack-plugin#terseroptions
        },
      }),
    ],
  },
};

splitChunks: {
	cacheGroup: { 
		common: {
      chunk: 'initial'
    },
    thirdPart:{
      test: /node_modules/,
      chunks: 'initial',
      priority: 2
    }
	}
}

```



### 懒加载

> Babel-sytax-dynamic-import

### dynamic import 原理

> 基于JSONP实现



1. 把获得promise push实例并挂在installedChunks 的List。
2. document.creatementElement（'script'）; script.src = publicpath + func(chunckID)
3. scriptComplete

4. 重写push执行webpackJsonpCallback

5. 将模块标记为已加载

6. 将模块代码挂载到modules exports上用promise包裹的。

7. 通过thenable获取到code

### 热更新原理

> 通过websocket建立建立双向通信，监听文件变化（时间变化而非hash变化），将文件内容放入内存（memory-fs），发送实时通知。



当监听文件变化，就会调用`_sendStats`方法通过`websocket`给浏览器发送文件变动数据，以让浏览器拿到最新的hash。

热更新过程

1. 通过 hotUpdate 查找旧模块
2. 将新的 加入 hotUpdate
3. 然后webpack——require执行

### 完整流程

1. 启动项目时会启动一个websocket连接
2. 启动编译
3. 调用setupHooks监听编译状态
4. 编译完成调用_sendStats发送事件
5. 客户端监听两个事件（ha sh/ok）
6. 触发 OK 事件调用reloadApp（）
7. reloadApp 发送webpackHotUpdate事件和当前hash给wbpack
8. 通过module.hot.check方法检查更新
9. 调用`hotDownloadUpdateChunk`发送`xxx/hash.hot-update.js` 请求。
10. 在webpackHotUpdate调用
11. 通过 hotUpdate 查找旧模块并将新的 加入 hotUpdate
12. 通过webpack_require加载执行