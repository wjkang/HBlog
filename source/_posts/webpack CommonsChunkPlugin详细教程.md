title: webpack CommonsChunkPlugin详细教程
author: 若邪
tags:
 - CommonsChunkPlugin
categories:
 - 前端
data: 2016-09-03
---
###1.demo结构：
![](http://upload-images.jianshu.io/upload_images/2125695-12d6fb8aec5c1dfd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
###2.package.json配置：
```
{
  "name": "webpack-simple-demo",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "webpack": "webpack"
  },
  "author": "",
  "license": "ISC",
  "dependencies": {
    "jquery": "^3.1.0",
    "vue": "^1.0.26"
  },
  "devDependencies": {
    "css-loader": "^0.24.0",
    "style-loader": "^0.13.1",
    "webpack": "^1.13.2",
    "webpack-dev-server": "^1.15.1"
  }
}
```
###3.多种打包情况(未使用CommonsChunkPlugin)
####(1)单一入口，模块单一引用
webpack.config.js
```
var CommonsChunkPlugin = require("webpack/lib/optimize/CommonsChunkPlugin");
module.exports = {
  entry:
  {
    main:'./main.js',
  },
  output: {
    path:__dirname+'/dist',
    filename: 'build.js'
  },
  plugins: [

  ]
};
```
main.js
```
require("jquery");
```
demo目录下运行命令行  webpack或npm run webpack
![](http://upload-images.jianshu.io/upload_images/2125695-8aa6da95a1f4191a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
jquery模块被一起打包到build.js
####(2)单一入口，模块重复引用
webpack.config.js不变，main.js：
```
require("./chunk1");
require("./chunk2");
```
chunk1.js:
```
require("./chunk2");
var chunk1=1;
exports.chunk1=chunk1;
```
chunk2.js:
```
var chunk2=1;
exports.chunk2=chunk2;
```
main.js引用了chunk1、chunk2,chunk1又引用了chunk2，打包后：
build.js:
```
 ...省略webpack生成代码
/************************************************************************/
/******/ ([
/* 0 */
/***/ function(module, exports, __webpack_require__) {

	__webpack_require__(1);
	__webpack_require__(2);


/***/ },
/* 1 */
/***/ function(module, exports, __webpack_require__) {

	__webpack_require__(2);
	var chunk1=1;
	exports.chunk1=chunk1;

/***/ },
/* 2 */
/***/ function(module, exports) {

	var chunk2=1;
	exports.chunk2=chunk2;

/***/ }
/******/ ]);
```
####(3)多入口，模块单一引用，分文件输出
webpack.config.js
```
var CommonsChunkPlugin = require("webpack/lib/optimize/CommonsChunkPlugin");
module.exports = {
  entry:
  {
    main:'./main.js',
    main1:'./main1.js'
  },
  output: {
    path:__dirname+'/dist',
    filename: '[name].js'
  },
  plugins: [

  ]
};
```
打包后文件main.js,main1.js  内容与（2）build.js一致
####（4）多入口，模块单一引用，单一文件输出
```
var CommonsChunkPlugin = require("webpack/lib/optimize/CommonsChunkPlugin");
module.exports = {
  entry:
  {
    main:'./main.js',
    main1:'./main1.js'
  },
  output: {
    path:__dirname+'/dist',
    filename: 'buid.js'
  },
  plugins: [

  ]
};
```
build.js与（2）一致
####（5）多入口，模块重复引用，单文件输出
webpack.config.js与（4）一致
main.js
```
require("./chunk1");
require("./chunk2");
exports.main=1;
```
main1.js
```
require("./chunk1");
require("./chunk2");
require("./main");
```
报错：ERROR in ./main1.js
Module not found: Error: a dependency to an entry point is not allowed
 @ ./main1.js 3:0-17
###4.使用CommonsChunkPlugin
(1)单一入口，模块单一引用，分文件输出:
webpack.config.js
```
var CommonsChunkPlugin = require("webpack/lib/optimize/CommonsChunkPlugin");
module.exports = {
  entry:
  {
    main:'./main.js',
  },
  output: {
    path:__dirname+'/dist',
    filename: '[name].js'//不使用[name]，并且插件中没有filename，这输出文件中只用chunk.js的内容，
    main.js的内容不知跑哪里去了
  },
  plugins: [
   new CommonsChunkPlugin({
       name:"chunk",
       filename:"chunk.js"//忽略则以name为输出文件的名字，否则以此为输出文件名字
   })
  ]
};
```
main.js
```
require("./chunk1");
require("./chunk2");
require("jquery");
```
![](http://upload-images.jianshu.io/upload_images/2125695-80c6450f282c71ec.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
输出文件main.js chunk.js,chunk1.js,chunck2.js,jquery都被打包到main.js里，好像并没有什么卵用，但是页面上使用的时候chunk.js必须在mian.js前引用

将chunk1.js,chunck2.js打包到chunk.js:
webpack.config.js
```
var CommonsChunkPlugin = require("webpack/lib/optimize/CommonsChunkPlugin");
module.exports = {
  entry:
  {
    main:'./main.js',
    chunk: ["./chunk1", "./chunk2"],//插件中name,filename必须以这个key为值
  },
  output: {
    path:__dirname+'/dist',
    filename: '[name].js'//不使用[name]，并且插件中没有filename，
    这输出文件中只用chunk.js的内容，main.js的内容不知跑哪里去了
  },
  plugins: [
   new CommonsChunkPlugin({
       name:"chunk",
      // filename:"chunk.js"//忽略则以name为输出文件的名字，否则以此为输出文件名字
   })
  ]
};
```
![](http://upload-images.jianshu.io/upload_images/2125695-ee4302704803a8a6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
####(1)单一入口，模块重复引用，分文件输出(单一入口CommonsChunkPlugin能否将多次引用的模块打包到公共模块呢？)：
webpack.config.js
```
var CommonsChunkPlugin = require("webpack/lib/optimize/CommonsChunkPlugin");
module.exports = {
  entry:
  {
    main:'./main.js',
    //main1:'./main1.js',
  },
  output: {
    path:__dirname+'/dist',
    filename: '[name].js'//不使用[name]，并且插件中没有filename，
这输出文件中只用chunk.js的内容，main.js的内容不知跑哪里去了
  },
  plugins: [
   new CommonsChunkPlugin({
       name:"chunk",
      // filename:"chunk.js"//忽略则以name为输出文件的名字，否则以此为输出文件名字
      minChunks:2
   })
  ]
};
```
main.js
```
require("./chunk1");
require("./chunk2");
```
chunk1.js
```
require("./chunk2");
var chunk1=1;
exports.chunk1=chunk1;
```
chunk2模块被引用了两次
打包后，所有模块还是被打包到main.js中
####(3)多入口，模块重复引用，分文件输出（将多次引用的模块打包到公共模块）
webpack.config.js
```
var CommonsChunkPlugin = require("webpack/lib/optimize/CommonsChunkPlugin");
module.exports = {
  entry:
  {
    main:'./main.js',
    main1:'./main1.js',
  },
  output: {
    path:__dirname+'/dist',
    filename: '[name].js'//不使用[name]，并且插件中没有filename，
    这输出文件中只用chunk.js的内容，main.js的内容不知跑哪里去了
  },
  plugins: [
   new CommonsChunkPlugin({
       name:"chunk",
      // filename:"chunk.js"//忽略则以name为输出文件的名字，否则以此为输出文件名字
      minChunks:2
   })
  ]
};
```
main.js,main1.js里都引用chunk1,chunk2.
打包后：
chunk1,chunk2被打包到chunk.js,不再像3(3)chunk1,chunk2分别被打包到mian,mian1中。
###5.将公共业务模块与类库或框架分开打包
webpack.config.js
```
var CommonsChunkPlugin = require("webpack/lib/optimize/CommonsChunkPlugin");
module.exports = {
    entry: {
        main: './main.js',
        main1: './main1.js',
        common1: ['jquery'],
        common2: ['vue']
    },
    output: {
        path: __dirname + '/dist',
        filename: '[name].js'//不使用[name]，并且插件中没有filename，
        //这输出文件中只用chunk.js的内容，main.js的内容不知跑哪里去了
    },
    plugins: [
        new CommonsChunkPlugin({
            name: ["chunk","common1","common2"],//页面上使用的时候common2
            //必须最先加载
            // filename:"chunk.js"//忽略则以name为输出文件的名字，
                //否则以此为输出文件名字
            minChunks: 2
        })
    ]
};
```
![](http://upload-images.jianshu.io/upload_images/2125695-c393f6cb60aed289.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
jquery被打包到common1.js,vue被打包到common2.js,chunk.js打包的是公共的业务模块(webpack用插件CommonsChunkPlugin进行打包的时候，将符合引用次数(minChunks)的模块打包到name参数的数组的第一个块里（chunk）,然后数组后面的块依次打包(查找entry里的key,没有找到相关的key就生成一个空的块)，最后一个块包含webpack生成的在浏览器上使用各个块的加载代码，所以页面上使用的时候最后一个块必须最先加载)
将webpack.config.js改为
```
var CommonsChunkPlugin = require("webpack/lib/optimize/CommonsChunkPlugin");
module.exports = {
    entry: {

        main: './main.js',
        main1: './main1.js',
        jquery:["jquery"],
        vue:["vue"]
    },
    output: {
        path: __dirname + '/dist',
        filename: '[name].js'
    },
    plugins: [
        new CommonsChunkPlugin({
            name: ["common","jquery","vue","load"],

            minChunks:2

        })
    ]
};
```
main.js
```
require("./chunk1");
require("./chunk2");
var jq=require("jquery");
console.log(jq);
```
main1.js
```
require("./chunk1");
require("./chunk2");
var vue=require("vue");
console.log(vue);
exports.vue=vue;
```
打包后
![](http://upload-images.jianshu.io/upload_images/2125695-d053503532733508.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
common.js
```
webpackJsonp([4,5],[
/* 0 */,
/* 1 */,
/* 2 */
/***/ function(module, exports, __webpack_require__) {

	__webpack_require__(3);
	var chunk1=1;
	exports.chunk1=chunk1;

/***/ },
/* 3 */
/***/ function(module, exports) {

	var chunk2=1;
	exports.chunk2=chunk2;

/***/ }
]);
```
相当于公共的业务代码都打包到了common.js里
load.js
```
/******/ (function(modules) { // webpackBootstrap
/******/ 	// install a JSONP callback for chunk loading
/******/ 	var parentJsonpFunction = window["webpackJsonp"];
/******/ 	window["webpackJsonp"] = function webpackJsonpCallback(chunkIds, moreModules) {
/******/ 		// add "moreModules" to the modules object,
/******/ 		// then flag all "chunkIds" as loaded and fire callback
/******/ 		var moduleId, chunkId, i = 0, callbacks = [];
/******/ 		for(;i < chunkIds.length; i++) {
/******/ 			chunkId = chunkIds[i];
/******/ 			if(installedChunks[chunkId])
/******/ 				callbacks.push.apply(callbacks, installedChunks[chunkId]);
/******/ 			installedChunks[chunkId] = 0;
/******/ 		}
/******/ 		for(moduleId in moreModules) {
/******/ 			modules[moduleId] = moreModules[moduleId];
/******/ 		}
/******/ 		if(parentJsonpFunction) parentJsonpFunction(chunkIds, moreModules);
/******/ 		while(callbacks.length)
/******/ 			callbacks.shift().call(null, __webpack_require__);
/******/ 		if(moreModules[0]) {
/******/ 			installedModules[0] = 0;
/******/ 			return __webpack_require__(0);
/******/ 		}
/******/ 	};

/******/ 	// The module cache
/******/ 	var installedModules = {};

/******/ 	// object to store loaded and loading chunks
/******/ 	// "0" means "already loaded"
/******/ 	// Array means "loading", array contains callbacks
/******/ 	var installedChunks = {
/******/ 		5:0
/******/ 	};

/******/ 	// The require function
/******/ 	function __webpack_require__(moduleId) {

/******/ 		// Check if module is in cache
/******/ 		if(installedModules[moduleId])
/******/ 			return installedModules[moduleId].exports;

/******/ 		// Create a new module (and put it into the cache)
/******/ 		var module = installedModules[moduleId] = {
/******/ 			exports: {},
/******/ 			id: moduleId,
/******/ 			loaded: false
/******/ 		};

/******/ 		// Execute the module function
/******/ 		modules[moduleId].call(module.exports, module, module.exports, __webpack_require__);

/******/ 		// Flag the module as loaded
/******/ 		module.loaded = true;

/******/ 		// Return the exports of the module
/******/ 		return module.exports;
/******/ 	}

/******/ 	// This file contains only the entry chunk.
/******/ 	// The chunk loading function for additional chunks
/******/ 	__webpack_require__.e = function requireEnsure(chunkId, callback) {
/******/ 		// "0" is the signal for "already loaded"
/******/ 		if(installedChunks[chunkId] === 0)
/******/ 			return callback.call(null, __webpack_require__);

/******/ 		// an array means "currently loading".
/******/ 		if(installedChunks[chunkId] !== undefined) {
/******/ 			installedChunks[chunkId].push(callback);
/******/ 		} else {
/******/ 			// start chunk loading
/******/ 			installedChunks[chunkId] = [callback];
/******/ 			var head = document.getElementsByTagName('head')[0];
/******/ 			var script = document.createElement('script');
/******/ 			script.type = 'text/javascript';
/******/ 			script.charset = 'utf-8';
/******/ 			script.async = true;

/******/ 			script.src = __webpack_require__.p + "" + chunkId + "." + ({"0":"jquery","1":"main","2":"main1","3":"vue","4":"common"}[chunkId]||chunkId) + ".js";
/******/ 			head.appendChild(script);
/******/ 		}
/******/ 	};

/******/ 	// expose the modules object (__webpack_modules__)
/******/ 	__webpack_require__.m = modules;

/******/ 	// expose the module cache
/******/ 	__webpack_require__.c = installedModules;

/******/ 	// __webpack_public_path__
/******/ 	__webpack_require__.p = "";
/******/ })
/************************************************************************/
/******/ ([]);
```
使用的时候必须最先加载load.js
###6.参数minChunks: Infinity
看一下下面的配置会是什么结果
```
var CommonsChunkPlugin = require("webpack/lib/optimize/CommonsChunkPlugin");
module.exports = {
    entry: {

        main: './main.js',
        main1: './main1.js',
        jquery:["jquery"]
    },
    output: {
        path: __dirname + '/dist',
        filename: '[name].js'
    },
    plugins: [
        new CommonsChunkPlugin({
            name: "jquery",

            minChunks:2

        })
    ]
};
```
![](http://upload-images.jianshu.io/upload_images/2125695-ab6cac77d9eaa278.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
main.js,main1.js共同引用的chunk1和chunk2会被打包到jquery.js里

minChunks:2修改为minChunks:Infinity后，chunk1和chunk2都打包到main.js,main1.js里
###7.参数chunks
webpack.config.js
```
var CommonsChunkPlugin = require("webpack/lib/optimize/CommonsChunkPlugin");
module.exports = {
    entry: {

        main: './main.js',
        main1: './main1.js',
        jquery:["jquery"]
    },
    output: {
        path: __dirname + '/dist',
        filename: '[name].js'
    },
    plugins: [
        new CommonsChunkPlugin({
            name: "jquery",

            minChunks:2,

            chunks:["main","main1"]

        })
    ]
};
```
只有在main.js和main1.js中都引用的模块才会被打包的到公共模块（这里即jquery.js）