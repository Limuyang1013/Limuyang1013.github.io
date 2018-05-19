---
title: webpack从0到1使用指南
date: 2018-05-19 16:02:23
categories: 前端开发
toc: true
tags: 
	- webpack
---
#### 为什么要用webpack
关于为什么要使用webpack，我比较认同的一种说法是：
> webpack可以很好地管理你开发中遇到的各种HTML、JS、CSS以及各种图片资源文件，同时对应不同的资源，webpack还提供了对应的Loaders将其进行转化为适用于浏览器使用的格式

#### 如何从0开始上手webpack

后会无期里面，阿吕说过这么一句话："你连世界都没有观过，哪里来的世界观"，在我们实际的项目开发中，比如你用的是vue，那么vue已经有很好的脚手架工具(vue-cli)供你使用了，或者有的开发团队，会有技术Leader专门预先做好相关的模板，方便后来新加入的成员尽快上手项目，但是随着你开发的项目越多，你可能会越不注意到那些最基本的东西，比如说这个模板到底是如何搭建的，或者说让你自己来搭建一个模板给其他人用，是否也能做到如此的简单易上手，我想这是每一个想要在前端路上进阶的人需要考虑的问题。

最好的方式就是自己试着去搭一个最简单的模板，从0到1的过程是最痛苦的，但是1到2或者说2到3都是水到渠成的事，下面就看看怎么开始从0搭建一个基于webpack的vue的开发环境

#### 创建项目

**注：使用的webpack版本为^3.10.0**
首先通过webstorm或者你手上的其他IDE创建一个空的项目，我这里暂且叫proWebpack，创建好后是这个样子的：

![文件目录](https://user-images.githubusercontent.com/11991572/40214592-3caf2c6a-5a8e-11e8-8b62-fceca0fe7f72.png)


因为我们通过npm来管理包，然后每个项目的根目录下都会有一个`package.json`文件来管理项目的配置信息，包括名称、版本、许可证等元数据以及记录所需的各种模块，包括 执行依赖，和开发依赖，我们可以通过在命令控制台用`npm init`命令创建一个`package.json`文件，但是这样很麻烦，因为这样`npm`会通过命令行问答的方式来初始化并创建`package.json`文件，为了方便起见我们使用`npm init -y`，这样`npm`就会使用一些默认值进行初始化：

![运行结果](https://user-images.githubusercontent.com/11991572/40214608-4f2b84e2-5a8e-11e8-8475-79d9d265800e.png)


ok现在我们得到一个package.json文件，包含了一些简单的信息比如name、version等字段，接下来，因为我们是想搭建一个用于vue开发环境的webpack目录结构，到目前为止还没看到vue的影子，依然是在命令行使用`npm install vue --save`，这里注意`package.json`中的依赖包有`dependencies`和`devDependencies`两种，这两种的区别：`dependencies` 表示我们要在生产环境下使用该依赖，`devDependencies` 则表示我们仅在开发环境使用该依赖，我们install时候使用--save会把包安装在dependencies 下面，所以执行完上面的命令你可以看到这样的目录结构：
```json
{
  "name": "proWebpack",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "dependencies": {
    "vue": "^2.5.16"
  }
}
```
然后在`devDependencies `下面安装webpack，再次提醒，由于webpack4变化较大，这里使用^3.10.0版本的webpack： `npm install -D webpack@3.10.0`，此时目录结构如下：
```json
{
  "name": "proWebpack",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "dependencies": {
    "vue": "^2.5.16"
  },
  "devDependencies": {
    "webpack": "^3.10.0"
  }
}

```
#### 创建入口文件
接下来创建入口文件，在根目录下创建一个index.html作为启动页面，一个webpack.config.js作为webpack配置文件(实际项目中这里会有webpack配置文件，分别用于开发环境和生产环境，这里简便起见就用一个配置文件)，新建一个src目录，在该目录下新建一个index.js作为打包入口文件：
```
proWebpack
├─ index.html 启动页面
├─ package-lock.json
├─ package.json 包管理
├─ src
│    └─ index.js 入口文件
└─ webpack.config.js  webpack配置文件
```
为了更接近vue-cli创建出来的模板，我们还需要创建一个Vue实例提供给入口文件的el挂载，这个Vue文件很简单长这样：
```vue
<template>
  <div>proWebpack</div>
</template>

```
然后写好入口文件：
```javascript
import Vue from 'vue'
import App from '../App.vue'

new Vue({
  el: '#app',
  render: h => h(App)
})
```
同时别忘记在Index.html里面要提供一个供vue实例挂载的HTMLElement实例：
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>proWebpack</title>
</head>
<body>
<div class="app">proWebpack</div>
</body>
</html>

```
这样准备工作做得差不多了，下面做最核心的部分

#### webpack.config.js的编写

webpack有这么几个核心的概念：
-  entry 入口起点，webpack会找出入口起点的直接或间接依赖项，将他们进行处理输出为我们称之为bundle的文件中
- output 输出路径，告诉 webpack 在哪里输出它所创建的 bundles以及如何给这些buldles命名
- loader 因为webpack本身只能理解javascript文件，所以当我们要用vue或者React的时候可以使用相关的loader(比如我们熟知的vue-loader)来将其转换成webpack 能够处理的有效模块
- plugins 扩展插件，可以用于打包的优化和压缩，这个可能要结合实际使用更好理解
- module 模块 webpack中的一切你都可以理解为模块
- chunk 代码块，一个 chunk 由多个模块组合而成，用于代码合并与分割。

下面开始配置，因为所有输出文件的目标路径必须是绝对路径，所以这里要用到使Node.js 的 path 模块
```json
// 引入webpack和path模块
const path = require('path')
const webpck = require('webpack')
```
简单配置文件的entry和output
```javascript
const path = require('path')
const webpck = require('webpack')
const productionPath = require('./package.json').name

module.exports = {
  entry: {
    index: './src/index.js'
  },
  output: {
    path: path.join(__dirname, 'dist', productionPath),
    filename: 'bundle.js'
  }
}

```
这里代码应该不难理解，就是从`index.js`这个入口进去，把直接或间接依赖项打包输出dist文件夹目录下,
怎么运行呢，我们需要在`package.json`的scripts对象里面添加我们自己的build命令：
```
//  这里用最简单的命令
"build": "webpack -w",
```
> 因为我们的webpack配置文件就叫webpack.config.js，默认情况下，webpack在执行的时候会搜索当前目录webpack.config.js 文件执行，实际开发中我们会使用 --config 选项来指定配置文件来对开发环境和生产环境的配置做出区分

直接在控制台`npm run build`，果不其然报错了：
```javascript
ERROR in ./App.vue
Module parse failed: Unexpected token (1:0)
You may need an appropriate loader to handle this file type.
| <template>
|   <div>proWebpack</div>
| </template>
 @ ./src/index.js 2:0-28
```
很明显，webpack告诉我们他没法识别vue文件，需要我们提供一个相关的loader来处理成webpack可以识别的文件，这里先`npm install vue-loader -D`，然后回到`webpack.config.json`配置loader：
```
const path = require('path')
const webpck = require('webpack')

module.exports = {
  entry: {
    index: './src/index.js'
  },
  output: {
    path: path.join(__dirname, 'dist'),
    filename: 'bundle.js'
  },
  module: {
    // 模块配置
    rules: [
      //模块规则(配置 loader、解析器等选项)
      {
        // 这里是匹配条件，每个选项都接收一个正则表达式或字符串
        // test是必须匹配选项
        // exclude 是必不匹配选项(优先于 test 和 include)
        // 对选中后的文件通过 use 配置项来应用 Loader，可以只应用一个 Loader 或者按照从后往前的顺序应用一组 Loader，同时还可以分别给 Loader 传入参数。
        test: /\.vue$/,
        exclude: /node_modules/,
        use: {loader: "vue-loader"}
      }
    ]
  }
}

```
配置好之后再run一遍，咦，又报错，我们看下主要的报错信息：
```
ERROR in ./App.vue
Module build failed: Error: Cannot find module 'vue-template-compiler'
    at Function.Module._resolveFilename (module.js:527:15)
    at Function.Module._load (module.js:476:23)

```
cannot fount那就是需要我们去install呗，但是为什么要下这个模块，我找到一个比较靠谱的说法：
> 其中 vue-template-compiler 是 vue-loader 的 peerDependencies，npm3 不会自动安装 peerDependencies，然而 vue-template-compiler 又是必备的，那为什么作者不将其放到 dependencies 中呢？有人在 github 上提过这个问题，我大致翻译一下作者的回答（仅供参考）：这样做的原因是因为没有可靠的方式来固定嵌套依赖的关系，怎么理解这句话？首先 vue-template-compiler 和 vue 的版本号是一致的（目前是同步更新），将 vue-template-compiler 指定为 vue-loader 的 dependencies 并不能保证 vue-template-compiler 和 vue 的版本号是相同的，让用户自己指定版本才能保证这一点。[查看作者的回答（英文）](https://github.com/vuejs/vue-loader/issues/560) 。如果两者版本不一致，运行时就不会错误提示。

那只需要我们`npm i vue-template-compiler -D`一下，然后在跑一遍build脚本，不出意外你应该能在项目根目录看到一个dist文件夹，把这个文件夹里面的js文件引入启动页面：
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>proWebpack</title>
</head>
<body>
<div class="app">proWebpack</div>
<script src="/dist/bundle.js"></script>
</body>
</html>
```
接下来我们要访问这个页面，这里用到webpack提供的devsever来调试我们的页面， DevServer 会启动一个 HTTP 服务器用于服务网页请求，同时会帮助启动 Webpack ，并接收 Webpack 发出的文件更变信号，通过 WebSocket 协议自动刷新网页做到实时预览。这就是为什么你用vue-cli搭建出来的脚手架可以做到页面实时渲染，配置这个也很简单，在`webpack.config.js`里面添加如下配置：
```
devServer: {
    port: 3000, // 端口号
  }
```
其实webpack-dev-server可以配置很多参数，这里不过多展开，接下来需要修改下我们的build脚本：
```json
"build": "webpack-dev-server"
```
别忘了`npm install`一下`webpack-dev-server`，最后执行npm run build看到控制台报出成功信息并告诉你你的项目正运行在localhost:3000：
```
$ npm run build

> proWebpack@1.0.0 build D:\proWebpack
> webpack-dev-server --open

Project is running at http://localhost:3000/
webpack output is served from /

```
访问localhost:3000看到如下显示的话，恭喜你，webpack配置如何从0到1你应该已经清楚了：

![渲染结果](https://user-images.githubusercontent.com/11991572/40214626-60d44eb8-5a8e-11e8-92a4-a91cb08c0b5e.png)

#### 更进一步

我们知道了怎么初步配置webpack还远远不够，实际开发中我们会遇到更多样的情况，比如说
当我们只有一个输出文件的时候我们可以在output写死，就像我们上面写的：
```
output: {
    path: path.join(__dirname, 'dist'),
    filename: 'bundle.js'
  }
```
但是当我们有多个文件输出的时候，一个个去写是一个费力不讨好的做法，这个时候就要用到webpack内置的变量了，我们可以这么写：
```
output: {
    path: path.join(__dirname, 'dist'),
    filename: '[name].[hash].js'
  }
```
这样会赋予每一个bundle唯一的名称，并且在每次构建过程中，生成唯一的 hash ，这时候稍微改动一下脚本文件：
```json
"scripts": {
    "start": "webpack-dev-server",
    "build": "webpack -w",
    "test": "echo \"Error: no test specified\" && exit 1"
  },
```
之前写一起是为了方便理解， 现在我把跑本地服务器和打包的脚本拆开写，更语义化，一般正常开发也是这么做的，先跑一下build，你会发现此时dist文件夹下面生成了新的打包文件`index.8ad46f4fb5c8385db614.js`，然后把这个文件同样引入`index.html`，跑start脚本发现结果也是照常输出，但是又有了新的问题，如果我们每次build之后都要手动引入带着一长串hash的js文件也是很蠢的事情，所以这里我们可以用上一个plugin叫做`html-webpack-plugin`，这个插件主要有两个作用：
1、在每次编译完成之后动态为html文件引入外部资源(script、link)，对于我me你这种带hash的文件名来说无疑是极为方便的
2、可以制定一个模板的html文件，html-webpack-plugin可以根据这个模板来生成html入口文件

下面看下如何使用：
```javascript
// 引入
const HtmlWebpackPlugin = require('html-webpack-plugin')
    //省略若干代码
 plugins: [
    new HtmlWebpackPlugin({
      filename: './index.html', // 生成的入口文件的名字，默认就是index.html
      template: './index.tpl.html',// 有时候，插件自动生成的html文件，并不是我们需要结构，我们需要给它指定一个模板，让插件根据我们给的模板生成html
      inject: 'body',// 有四个选项值 true, body, head, false----true:默认值，script标签位于html文件的 body 底部 body:同true head:插入的js文件位于head标签内 false:不插入生成的js文件，只生成一个纯html
    })
  ],
```
用到的`index.tpl.html`只是和之前的`index.html`作了一点小小的区分，让人知道这是模板生成的：
```html
// index.tpl.html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>这是模板生成的</title>
</head>
<body>
<div class="app">proWebpack</div>
</body>
</html>
```
方便起见我们在`package.json`在添加一个clean的脚本，每次build之前先删除之前生成的dist文件：
```
"scripts": {
    "start": "webpack-dev-server",
    "build": "npm run clean && webpack -w",
    "clean": "rimraf dist", // 添加clean脚本
    "test": "echo \"Error: no test specified\" && exit 1"
  }
```
然后run build一下：
![模板生成的结果](https://user-images.githubusercontent.com/11991572/40266366-85f26740-5b7c-11e8-9476-adb1539ca6a0.png)
我们发现这个时候的入口文件已经是根据模板自动生成的了，而且自动把生成的js文件添加到了入口文件里面，同时npm run start命令看到的界面也是使用模板生成的页面，这就大功告成了。

#### 最后
其实这里还有一点没有提到的就是css的处理，除了css-loader我们平常开发还会用到一些css预处理器如scss，还有postcss等样式处理工具，这里的处理其实也不难，感兴趣的朋友可以自己尝试一下，这里由于篇幅原因(我想偷懒)就不过多展开了。
