# webpack 实现编译预渲染
## 常见的网页渲染方式
### 客户端渲染 CSR
HTML 只提供一个根 DOM 节点，由浏览器加载完脚本后再渲染，在此期间页面呈现空白状态，知道 js 执行完成，也不利于SEO，常见于现在使用组件开发的项目。
### 服务端渲染 SSR
由服务器提供 DOM 节点比较完善的 HTML，浏览器在接收到 HTML 文档和样式表后就可以渲染出比较完整的 UI，是前端组件式开发前常见的方式。现在 vue、react 也有提供相应的方法，可以实现服务端渲染，但会给后端带来额外的逻辑。
### 编译预渲染 Prerendering
是在 webpack 编译完成后，使用插件工具开启一个浏览器对页面进行加载，在页面加载完成后获取整个 HTML 的内容，替换掉原 HTML。但是这种渲染方式会导致 js 执行后插入一些重复的内容，需要做特殊处理。

webpack 实现编译预渲染主要使用了一个插件 [prerender-spa-plugin](https://github.com/chrisvfritz/prerender-spa-plugin)，示例如下：
```javascript
// webpack.config.js
const path = require('path');
const fs = require('fs');
// 预渲染插件
const PrerenderSPAPlugin = require('prerender-spa-plugin');
// 将组件的css抽离成单独css文件
// 如果将css打包到js中，js执行时动态插入到html中，则会导致css样式重复插入到html中，出现代码冗余，而且也会导致js文件较大
// 但是如果将css抽离成文件，则dom节点在css文件加载完成前将会没有样式，用户体验也会比较差，最好将首屏样式读取后插入到header，其他css文件使用link引入
const ExtractTextPlugin = require('extract-text-webpack-plugin');
 
// 建议只在生产环境使用预渲染
module.exports = {
    mode: 'production',
    // ...
    module: {
        rules: [
            // ...
            {
                test: /\.css$/,
                use: ExtractTextPlugin.extract({
                    fallback: 'style-loader',
                    use: [
                        { loader: 'css-loader', options: { importLoaders: 1 } },
                            { loader: 'postcss-loader' }
                    ]
                })
                },
            // ...
        ]
    },
    plugins: [
        new ExtractTextPlugin({
            filename: 'style/[name].css',
            allChunks: true
        }),
        new PrerenderSPAPlugin({
            // index.html is in the root directory.
            staticDir: path.join(__dirname, 'build'),
            routes: [ '/' ],
            postProcess (renderedRoute) {
                // 有一些由js动态插入的内容会在页面中重复添加，可以在这里将其处理
                renderedRoute.html = renderedRoute.html
                    .replace(//, '');
                // 将css插入html，移除css文件
                const styles = fs.readFileSync(path.join(__dirname, 'build', 'style/index.css'));
                renderedRoute.html = renderedRoute.html
                    .replace('', `${styles}`)
                return renderedRoute;
            },
            // Optional minification.
            minify: {
                collapseBooleanAttributes: true,
                collapseWhitespace: true,
                decodeEntities: true,
                keepClosingSlash: true,
                sortAttributes: true
            },
            renderer: new Renderer({
                // 页面可能有异步操作，可以设置在页面打开几秒后获取html的内容
                renderAfterTime: 3000,
                // 设置预渲染时的变量，可以在组件中根据这个变量做一些特殊处理，下边模拟SSR就会用到这个变量
                injectProperty: '__PRERENDER_INJECTED',
                inject: {
                    prerender: 'prerender'
                }
            })
        })
    ]
};
```
其中对 css 样式表和一些动态注入的 js 做了字符串替换的特殊处理，你可能还会遇到用了 iconfont.js 导致重复插入大量 svg 的问题，这就需要用类似的方式去移除。

如果只是在 webpack 中做处理还是不够的，js 执行后 react 会向根节点中再次注入 DOM 节点。为避免这种情况，还需要使用 react 提供的一个特殊方法，如下：
```javascript
// app.jsx
import React from 'react';
import ReactDOM from 'react-dom';
import ReactDOMServer from 'react-dom/server';
import Gui from './gui.jsx';
 
const container = document.querySelector('#root');
const reactElement =
// 在预渲染时，有该变量，调用ReactDOMServer.renderToString方法注入dom
if (window.__PRERENDER_INJECTED) {
    const appHTML = ReactDOMServer.renderToString(reactElement);
   container.innerHTML = appHTML;
} else {
   // 正式浏览器打开时，调用ReactDOM.hydrate方法将组件和已存在dom节点建立对应关系
   ReactDOM.hydrate(reactElement, container);
}
```

ps: webpack 插件 prerender-spa-plugin，会启动一个 google 提供的 [Puppeteer](https://github.com/GoogleChrome/puppeteer#readme) 无界面 Chrome 工具，这是一个 node 包，也可以用来做自动化测试、爬虫等