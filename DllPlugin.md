# DllPlugin 使用方法
DllPlugin：https://webpack.docschina.org/plugins/dll-plugin/

### 使用场景
- 多项目依赖，有公用代码
- 本地编译打包速度比较慢
- 一些模块长期不需要更新

### 带来的好处
- 减小代码体积
- 部分模块不需要构建时重复编译
- 不需要更新的脚本，可以放在云端缓存

### 和externals外部依赖的对比
相同点：都是避免重复打包一些模块

不同点：externals不打包这些模块，而是从当前环境中获取，如果当前环境中没有就会报错。而dllPlugin会打包这些模块，但打包一次，其他仓库引入后不会重复打包这些模块。

### 使用方法
DllPlugin 要提取的模块需要单独一个 webpack 配置，如下：
```javascript
// webpack.dll.js
const path = require('path');
const webpack = require('webpack');
module.exports = {
    entry: {
        // 将共用的模块放到entry里
        'mdll': ['react', 'react-dom', 'lodash', ...]
    }
    output: {
        path: path.resolve(__dirname, 'dist', 'js'),
        filename: '[name].js',
        library: '[name]',
        libraryTarget: 'umd'
   },
    // webpack其他配置...
    plugins: [
        new webpack.DllPlugin({
           context: __dirname,            // 必填项，用来标志manifest中的路径
           path: path.join(__dirname, 'dist/manifest.json'),    // 必填项，存放manifest的路径
           name: '[name]'                 // 必填项，manifest中的name
       }),
    ]
}
```

项目中再依赖 DllPlugin 打包的模块时，则需要使用 DllReferencePlugin，如下：
```javascript
const webpack = require('webpack');
 
module.exports = {
    // ...
    plugins: [
        new webpack.DllReferencePlugin({
            context: __dirname,
            manifest: require('./dist/manifest.json')
        })
    ]
    // ...
}
```
DllReferencePlugin 通过引用 dll 的 manifest 文件，找到依赖模块名称对应 DllPlugin 打包好的 js 里的 id，之后再在需要的时候通过内置的 __webpack_require__ 函数来加载他们。

使用 DllPlugin 生成的js因为不在项目 webpack.config.js 的 entry 里，所以不会被 HtmlWebpackPlugin 自动添加到 html 中，需要做如下处理：
```javascript
const path = require('path');
const webpack = require('webpack');
const HtmlWebpackPlugin = require('html-webpack-plugin');
const AddAssetHtmlPlugin = require('add-asset-html-webpack-plugin');
 
module.exports = {
    ...
    plugins: [
        new webpack.DllReferencePlugin({
            context: __dirname,
            manifest: require('./dist/manifest.json')
        }),
        new HtmlWebpackPlugin({
            title: 'myApp',
            template: 'demo/index.html',
            filename: 'index.html'
        }),
        new AddAssetHtmlPlugin({
            filepath: path.join(__dirname, 'dist', 'js', 'mdll.js'),
            outputPath: 'js', // 根据output设置的脚本存放目录，不然会默认放在index.html所在目录
            publicPath: 'js', // 在index.html中script标签src的目录
            includeSourcemap: true // 是否同时拷贝sourcemap
        })
    ]
    ...
}
```