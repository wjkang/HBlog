title: 简要分析webpack打包后代码
author: 若邪
tags:
 - webpack
categories:
 - 前端
date: 2016-09-04
---
开门见山
### 1.打包单一模块
webpack.config.js
```javascript
module.exports = {
    entry:"./chunk1.js",
    output: {
        path: __dirname + '/dist',
        filename: '[name].js'
    },
};
```
chunk1.js
```javascript
var chunk1=1;
exports.chunk1=chunk1;
```
打包后，main.js(webpack生成的一些注释已经去掉)
```javascript
 (function(modules) { // webpackBootstrap
 	// The module cache
 	var installedModules = {};
 	// The require function
 	function __webpack_require__(moduleId) {
 		// Check if module is in cache
 		if(installedModules[moduleId])
 			return installedModules[moduleId].exports;
 		// Create a new module (and put it into the cache)
 		var module = installedModules[moduleId] = {
 			exports: {},
 			id: moduleId,
 			loaded: false
 		};
 		// Execute the module function
 		modules[moduleId].call(module.exports, module, module.exports, __webpack_require__);
 		// Flag the module as loaded
 		module.loaded = true;
 		// Return the exports of the module
 		return module.exports;
 	}


 	// expose the modules object (__webpack_modules__)
 	__webpack_require__.m = modules;
 	// expose the module cache
 	__webpack_require__.c = installedModules;
 	// __webpack_public_path__
 	__webpack_require__.p = "";
 	// Load entry module and return exports
 	return __webpack_require__(0);
 })([function(module, exports) {
	var chunk1=1;
	exports.chunk1=chunk1;
}]);
```
这其实就是一个立即执行函数，简化一下就是：
```javascript
(function(module){})([function(){},function(){}]);
```
OK,看一下自运行的匿名函数里面干了什么：
```javascript
function(modules) { // webpackBootstrap
 	// modules就是一个数组，元素就是一个个函数体，就是我们声明的模块
 	var installedModules = {};
 	// The require function
 	function __webpack_require__(moduleId) {
 		...
 	}
 	// expose the modules object (__webpack_modules__)
 	__webpack_require__.m = modules;
 	// expose the module cache
 	__webpack_require__.c = installedModules;
 	// __webpack_public_path__
 	__webpack_require__.p = "";
 	// Load entry module and return exports
 	return __webpack_require__(0);
 }
```
整个函数里就声明了一个变量installedModules 和函数__webpack_require__，并在函数上添加了一个m,c,p属性，m属性保存的是传入的模块数组，c属性保存的是installedModules变量，P是一个空字符串。最后执行__webpack_require__函数，参数为零，并将其执行结果返回。下面看一下__webpack_require__干了什么：
```javascript
function __webpack_require__(moduleId) {
		//moduleId就是调用是传入的0
 		// installedModules[0]是undefined,继续往下
 		if(installedModules[moduleId])
 			return installedModules[moduleId].exports;
 		// module就是{exports: {},id: 0,loaded: false}
 		var module = installedModules[moduleId] = {
 			exports: {},
 			id: moduleId,
 			loaded: false
 		};
 		// 下面接着分析这个
 		modules[moduleId].call(module.exports, module, module.exports, __webpack_require__);
 		// 表明模块已经载入
 		module.loaded = true;
 		// 返回module.exports(注意modules[moduleId].call的时候module.exports会被修改)
 		return module.exports;
 	}
```
接着看一下modules[moduleId].call(module.exports, module, module.exports, __webpack_require__)，其实就是
```javascript
modules[moduleId].call({}, module, module.exports, __webpack_require__)
```
对call不了解当然也可以认为是这样(但是并不是等价，call能确保当模块中使用this的时候，this是指向module.exports的)：
```javascript
function  a(module, exports) {
	var chunk1=1;
	exports.chunk1=chunk1;
}
a(module, exports,__webpack_require__);
```
传入的module就是{exports: {},id: 0,loaded: false}，exports就是{}，__webpack_require__就是声明的__webpack_require__函数(传入这个函数有什么用呢，第二节将会介绍)；
运行后module.exports就是{chunk1:1}。所以当我们使用chunk1这个模块的时候（比如var chunk1=require("chunk1"),得到的就是一个对象{chunk1:1}）。如果模块里没有exports.chunk1=chunk1或者module.exports=chunk1得到的就是一个空对象{}
### 2.使用模块
上面我们已经分析了webpack是怎么打包一个模块的（入口文件就是一个模块），现在我们来看一下使用一个模块，然后使用模块的文件作为入口文件
webpack.config.js
```javascript
module.exports = {
    entry:"./main.js",
    output: {
        path: __dirname + '/dist',
        filename: '[name].js'
    }
};
```

main.js
```javascript
var chunk1=require("./chunk1");
console.log(chunk1);
```
打包后
```javascript
(function (modules) { // webpackBootstrap
	// The module cache
	var installedModules = {};
	// The require function
	function __webpack_require__(moduleId) {
		// Check if module is in cache
		if (installedModules[moduleId])
			return installedModules[moduleId].exports;
		// Create a new module (and put it into the cache)
		var module = installedModules[moduleId] = {
			exports: {},
			id: moduleId,
			loaded: false
		};
		// Execute the module function
		modules[moduleId].call(module.exports, module, module.exports, __webpack_require__);
		// Flag the module as loaded
		module.loaded = true;
		// Return the exports of the module
		return module.exports;
	}
	// expose the modules object (__webpack_modules__)
	__webpack_require__.m = modules;
	// expose the module cache
	__webpack_require__.c = installedModules;
	// __webpack_public_path__
	__webpack_require__.p = "";
	// Load entry module and return exports
	return __webpack_require__(0);
})([function (module, exports, __webpack_require__) {
	var chunk1=__webpack_require__(1);
	console.log(chunk1);
}, function (module, exports) {
    var chunk1 = 1;
	exports.chunk1 = chunk1;
}]);
```
不一样的地方就是自执行函数的参数由
```javascript
[function(module, exports) { var chunk1=1; exports.chunk1=chunk1;}]
```
变为
```javascript
[function (module, exports, __webpack_require__) {
	var chunk1=__webpack_require__(1);
	console.log(chunk1);
}, function (module, exports) {
    var chunk1 = 1;
	exports.chunk1 = chunk1;
}]
```
其实就是多了一个main模块，不过这个模块没有导出项，而且这个模块依赖于chunk1模块。所以当运行__webpack_require__(0)的时候，main模块缓存到installedModules[0]上，modules[0].call(也就是调用main模块)的时候，chunk1被缓存到installedModules[1]上，并且导出对象{chunk1：1}给模块main使用
### 3.重复使用模块
webpack.config.js
```javascript
module.exports = {
    entry:"./main.js",
    output: {
        path: __dirname + '/dist',
        filename: '[name].js'
    }
};
```
main.js
```javascript
var chunk1=require("./chunk1");
var chunk2=require(".chunlk2");
console.log(chunk1);
console.log(chunk2);
```
chunk1.js
```javascript
var chunk2=require("./chunk2");
var chunk1=1;
exports.chunk1=chunk1;
```
chunk2.js
```javascript
var chunk2=1;
exports.chunk2=chunk2;
```
打包后
```javascript
(function (modules) { // webpackBootstrap
	// The module cache
	var installedModules = {};
	// The require function
	function __webpack_require__(moduleId) {
		// Check if module is in cache
		if (installedModules[moduleId])
			return installedModules[moduleId].exports;
		// Create a new module (and put it into the cache)
		var module = installedModules[moduleId] = {
			exports: {},
			id: moduleId,
			loaded: false
		};
		// Execute the module function
		modules[moduleId].call(module.exports, module, module.exports, __webpack_require__);
		// Flag the module as loaded
		module.loaded = true;
		// Return the exports of the module
		return module.exports;
	}
	// expose the modules object (__webpack_modules__)
	__webpack_require__.m = modules;
	// expose the module cache
	__webpack_require__.c = installedModules;
	// __webpack_public_path__
	__webpack_require__.p = "";
	// Load entry module and return exports
	return __webpack_require__(0);
})([function (module, exports, __webpack_require__) {

	var chunk1 = __webpack_require__(1);
	var chunk2 = __webpack_require__(2);
	console.log(chunk1);
	console.log(chunk2);
}, function (module, exports, __webpack_require__) {

	__webpack_require__(2);
	var chunk1 = 1;
	exports.chunk1 = chunk1;
}, function (module, exports) {

	var chunk2 = 1;
	exports.chunk2 = chunk2;
}]);
```
不难发现，当需要重复使用模块的时候，缓存变量installedModules 就起作用了
### 4.多个打包入口
不管是单一模块还是重复模块，和以上两种一样
### 5.入口参数为数组
webpack.config.js
```javascript
module.exports = {
    entry:['./main.js','./main1.js'],
    output: {
        path: __dirname + '/dist',
        filename: '[name].js'
    }
};
```
打包后
```javascript
[
/* 0 */
/***/ function(module, exports, __webpack_require__) {
	__webpack_require__(1);
	module.exports = __webpack_require__(3);
/***/ },
/* 1 */
/***/ function(module, exports, __webpack_require__) {
	var chunk1=__webpack_require__(2);
	console.log(chunk1);
/***/ },
/* 2 */
/***/ function(module, exports) {
	var chunk1=1;
	exports.chunk1=chunk1;
/***/ },
/* 3 */
/***/ function(module, exports, __webpack_require__) {
	var chunk1=__webpack_require__(2);
/***/ }
/******/ ]
```
这里只截取自执行匿名函数的参数，因为其他代码与之前一样。可以看到1就是main默模块，2就是chunk1模块，3就是mian1模块，0的作用就是运行模块mian,mian1,然后将main1模块导出（main1中没有导出项，所以到导出{}），总结一下：入口参数是字符串不管是多入口还是单入口，最后都会将入口模块的导出项导出,没有导出项就导出{}，而入口参数是数组，就会将最后一个模块导出（webpackg官网有说明）
### 6.使用CommonsChunkPlugin插件
webpack.config.js
```javascript
var CommonsChunkPlugin = require("webpack/lib/optimize/CommonsChunkPlugin");
module.exports = {
    entry: {
        main: './main.js',
        main1: './main1.js',
    },
    output: {
        path: __dirname + '/dist',
        filename: '[name].js'
    },
    plugins: [
        new CommonsChunkPlugin({
        name: "common"
        })
    ]
};
```
main mian1中都require了chunk1,所以chunk1会被打包到common。
打包后，common.js
```javascript
(function (modules) { // webpackBootstrap
	// install a JSONP callback for chunk loading
	var parentJsonpFunction = window["webpackJsonp"];
	window["webpackJsonp"] = function webpackJsonpCallback(chunkIds, moreModules) {
		// add "moreModules" to the modules object,
		// then flag all "chunkIds" as loaded and fire callback
		var moduleId, chunkId, i = 0, callbacks = [];
		for (; i < chunkIds.length; i++) {
			chunkId = chunkIds[i];
			if (installedChunks[chunkId])
				callbacks.push.apply(callbacks, installedChunks[chunkId]);
			installedChunks[chunkId] = 0;
		}
		for (moduleId in moreModules) {
			modules[moduleId] = moreModules[moduleId];
		}
		if (parentJsonpFunction) parentJsonpFunction(chunkIds, moreModules);
		while (callbacks.length)
			callbacks.shift().call(null, __webpack_require__);
		if (moreModules[0]) {
			installedModules[0] = 0;
			return __webpack_require__(0);
		}
	};
	// The module cache
	var installedModules = {};
	// object to store loaded and loading chunks
	// "0" means "already loaded"
	// Array means "loading", array contains callbacks
	var installedChunks = {
		2: 0
	};
	// The require function
	function __webpack_require__(moduleId) {
		// Check if module is in cache
		if (installedModules[moduleId])
			return installedModules[moduleId].exports;
		// Create a new module (and put it into the cache)
		var module = installedModules[moduleId] = {
			exports: {},
			id: moduleId,
			loaded: false
		};
		// Execute the module function
		modules[moduleId].call(module.exports, module, module.exports, __webpack_require__);
		// Flag the module as loaded
		module.loaded = true;
		// Return the exports of the module
		return module.exports;
	}
	// This file contains only the entry chunk.
	// The chunk loading function for additional chunks
	__webpack_require__.e = function requireEnsure(chunkId, callback) {
		// "0" is the signal for "already loaded"
		if (installedChunks[chunkId] === 0)
			return callback.call(null, __webpack_require__);
		// an array means "currently loading".
		if (installedChunks[chunkId] !== undefined) {
			installedChunks[chunkId].push(callback);
		} else {
			// start chunk loading
			installedChunks[chunkId] = [callback];
			var head = document.getElementsByTagName('head')[0];
			var script = document.createElement('script');
			script.type = 'text/javascript';
			script.charset = 'utf-8';
			script.async = true;
			script.src = __webpack_require__.p + "" + chunkId + "." + ({ "0": "main", "1": "main1" }[chunkId] || chunkId) + ".js";
			head.appendChild(script);
		}
	};
	// expose the modules object (__webpack_modules__)
	__webpack_require__.m = modules;
	// expose the module cache
	__webpack_require__.c = installedModules;
	// __webpack_public_path__
	__webpack_require__.p = "";
})([, function (module, exports) {

	var chunk1 = 1;
	exports.chunk1 = chunk1;

}]);
```
main.js
```javascript
webpackJsonp([0],[function(module, exports, __webpack_require__) {

	var chunk1=__webpack_require__(1);
	console.log(chunk1);
 }]);
```
main1.js
```javascript
webpackJsonp([1],[function(module, exports, __webpack_require__) {
	var chunk1=__webpack_require__(1);
	console.log(chunk1);
}]);
```
与之前相比，多了webpackJsonp函数，立即执行的匿名函数没有立即调用__webpack_require__(0)。看一下webpackJsonp：
```javascript
var parentJsonpFunction = window["webpackJsonp"];
	window["webpackJsonp"] = function webpackJsonpCallback(chunkIds, moreModules) {
		//moreModules为独立chunk代码，chunkIds标记独立chunk唯一性避免按需加载时重复加载
		//以main.js中代码为例，chunkIds为[0],moreModules为
		//[function(module, exports, __webpack_require__) {
		//	var chunk1=__webpack_require__(1);
		//	console.log(chunk1);
		//}]
		var moduleId, chunkId, i = 0, callbacks = [];
		for (; i < chunkIds.length; i++) {
			chunkId = chunkIds[i];//chunkId=0
			if (installedChunks[chunkId])
				callbacks.push.apply(callbacks,installedChunks[chunkId]);//0 push入callbacks(使用requireEnsure不再是0)
			//赋值为0表明chunk已经loaded
			installedChunks[chunkId] = 0;
		}
		for (moduleId in moreModules) {
			//modules[0]会被覆盖
			modules[moduleId] = moreModules[moduleId];
		}
		//按当前情况parentJsonpFunction一直未undefined
		if (parentJsonpFunction) parentJsonpFunction(chunkIds, moreModules);
		//按当前情况callbacks=[]
		while (callbacks.length)
			callbacks.shift().call(null, __webpack_require__);
		if (moreModules[0]) {
			installedModules[0] = 0;
			return __webpack_require__(0);
		}
	};
	// 缓存模块，通过闭包引用(window["webpackJsonp"]可以访问到)
	var installedModules = {};
	//2为公共chunck唯一ID，0表示已经loaded
	var installedChunks = {
		2: 0
	};
	// The require function
	function __webpack_require__(moduleId) {
		// Check if module is in cache
		if (installedModules[moduleId])
			return installedModules[moduleId].exports;
		// Create a new module (and put it into the cache)
		var module = installedModules[moduleId] = {
			exports: {},
			id: moduleId,
			loaded: false
		};
		// Execute the module function
		modules[moduleId].call(module.exports, module, module.exports, __webpack_require__);
		// Flag the module as loaded
		module.loaded = true;
		// Return the exports of the module
		return module.exports;
	}
	//按需加载
	__webpack_require__.e = function requireEnsure(chunkId, callback) {
		// "0" is the signal for "already loaded"
		if (installedChunks[chunkId] === 0)
			return callback.call(null, __webpack_require__);
		// an array means "currently loading".
		if (installedChunks[chunkId] !== undefined) {
			installedChunks[chunkId].push(callback);
		} else {
			// start chunk loading
			installedChunks[chunkId] = [callback];
			var head = document.getElementsByTagName('head')[0];
			var script = document.createElement('script');
			script.type = 'text/javascript';
			script.charset = 'utf-8';
			script.async = true;
			script.src = __webpack_require__.p + "" + chunkId + "." + ({ "0": "main", "1": "main1" }[chunkId] || chunkId) + ".js";
			head.appendChild(script);
		}
	};
```
好像看不出什么。。。，修改一下
webpack.config.js
```javascript
var CommonsChunkPlugin = require("webpack/lib/optimize/CommonsChunkPlugin");
module.exports = {
    entry: {
        main: './main.js',
        main1: './main1.js',
        chunk1:["./chunk1"]
    },
    output: {
        path: __dirname + '/dist2',
        filename: '[name].js'
    },
    plugins: [
        new CommonsChunkPlugin({
        name: ["chunk1"],
        filename:"common.js",
        minChunks:3,
        })
    ]
};
```
main,main1都分别require chunk1,chunk2,然后将chunk1打包到公共模块（minChunks:3，chunk2不会被打包到公共模块）。自运行匿名函数最后多了
```javascript
   return __webpack_require__(0);
```
则installedModules[0]为已经loaded,看common.js，installedModules[1]也会loaded。
main.js
```javascript
webpackJsonp([1], [function (module, exports, __webpack_require__) {

	var chunk1 = __webpack_require__(1);
	var chunk2 = __webpack_require__(2);
	exports.a = 1;
	console.log(chunk1);
}, , function (module, exports) {
	var chunk2 = 1;
	exports.chunk2 = chunk2;

}
]);
```
main1.js
```javascript
webpackJsonp([2], [function (module, exports, __webpack_require__) {

	var chunk1 = __webpack_require__(1);
	var chunk2 = __webpack_require__(2);
	exports.a = 1;
	console.log(chunk1);
}, , function (module, exports) {
	var chunk2 = 1;
	exports.chunk2 = chunk2;
}
]);
```
common.js  modules：
```javascript
[function (module, exports, __webpack_require__) {

	module.exports = __webpack_require__(1);
}, function (module, exports) {

	var chunk1 = 1;
	exports.chunk1 = chunk1;
}]

```
以main.js的代码为例，调用webpackJsonp，传入的参数chunkIds为[1],moreModules为
```javascript
[function (module, exports, __webpack_require__) {

	var chunk1 = __webpack_require__(1);
	var chunk2 = __webpack_require__(2);
	exports.a = 1;
	console.log(chunk1);
}, , function (module, exports) {
	var chunk2 = 1;
	exports.chunk2 = chunk2;

}]
```
```javascript
var moduleId, chunkId, i = 0, callbacks = [];
		for (; i < chunkIds.length; i++) {
			chunkId = chunkIds[i];//1
			//false,赋值为0后还是false
			if (installedChunks[chunkId])
				callbacks.push.apply(callbacks, installedChunks[chunkId]);
			installedChunks[chunkId] = 0;
		}
		//三个模块
		for (moduleId in moreModules) {
			//moduleId:0,1,2  moreModules[1]为空模块，自执行函数的参数(公共模块)会被覆盖，但是参数中的相应模块已经loaded并且缓存
			modules[moduleId] = moreModules[moduleId];
		}
		if (parentJsonpFunction) parentJsonpFunction(chunkIds, moreModules);
		while (callbacks.length)
			callbacks.shift().call(null, __webpack_require__);
		if (moreModules[0]) {
			//installedModules[0]会重新load,但是load的是moreModules[0]，因为modules[0]已经被覆盖，moreModules[0]依赖于
			//modules[1]、modules[2],modules[1]已经loaded
			installedModules[0] = 0;
			return __webpack_require__(0);
		}
```
再看下面的情况：
common.js 自执行函数参数（公共模块）（没有return __webpack_require__(0)）
```javascript
[,function(module, exports, __webpack_require__) {

	var chunk1=1;
	var chunk2=__webpack_require__(2);
	exports.chunk1=chunk1;
},function(module, exports) {

	var chunk2=1;
	exports.chunk2=chunk2;
}]
```
main.js
```javascript
webpackJsonp([0],[
/* 0 */
/***/ function(module, exports, __webpack_require__) {

	var chunk1=__webpack_require__(1);
	var chunk2=__webpack_require__(2);
	exports.a=1;
	console.log(chunk1);
	//main
/***/ }
]);
```
以main调用分析
```javascript
         var moduleId, chunkId, i = 0, callbacks = [];
 		for(;i < chunkIds.length; i++) {
 			chunkId = chunkIds[i];//0
 			if(installedChunks[chunkId])
 				callbacks.push.apply(callbacks, installedChunks[chunkId]);
 			installedChunks[chunkId] = 0;//表明唯一索引为0的chunk已经loaded
 		}
 		for(moduleId in moreModules) {
			//moreModules只有一个元素，所以modules[1]、modules[2]不会被覆盖
 			modules[moduleId] = moreModules[moduleId];
 		}
 		if(parentJsonpFunction) parentJsonpFunction(chunkIds, moreModules);
 		while(callbacks.length)
 			callbacks.shift().call(null, __webpack_require__);
 		if(moreModules[0]) {
 			installedModules[0] = 0;
			//moreModules[0]即modules[0]依赖modules[1]、即modules[2](没有被覆盖很关键)
 			return __webpack_require__(0);
 		}
```
还有这种打包情况：
common.js不包含公共模块，即自执行函数参数为[]。
main.js
```javascript
webpackJsonp([0,1],[
function(module, exports, __webpack_require__) {

	var chunk1=__webpack_require__(1);
	var chunk2=__webpack_require__(2);
	exports.a=1;
	console.log(chunk1);
},function(module, exports) {
    var chunk1=1;
	exports.chunk1=chunk1;
},function(module, exports) {
    var chunk2=1;
	exports.chunk2=chunk2;
}]);
```
以main调用分析
```javascript
     var moduleId, chunkId, i = 0, callbacks = [];
 		for(;i < chunkIds.length; i++) {
 			chunkId = chunkIds[i];//0,1
 			if(installedChunks[chunkId])
 				callbacks.push.apply(callbacks, installedChunks[chunkId]);
 			installedChunks[chunkId] = 0;
 		}
 		for(moduleId in moreModules) {
            //moreModules全部转移到modules
 			modules[moduleId] = moreModules[moduleId];
 		}
 		if(parentJsonpFunction) parentJsonpFunction(chunkIds, moreModules);
 		while(callbacks.length)
 			callbacks.shift().call(null, __webpack_require__);
 		if(moreModules[0]) {
            //modules[0]是chunk文件运行代码
 			installedModules[0] = 0;
 			return __webpack_require__(0);
 		}
```
### 7.按需加载
webpack.config.json
```javascript
module.exports = {
  entry: './main.js',
  output: {
    filename: 'bundle.js'
  }
};
```
main.js
```javascript
require.ensure(['./a'], function(require) {
  var content = require('./a');
  document.open();
  document.write('<h1>' + content + '</h1>');
  document.close();
});
```
a.js
```javascript
module.exports = 'Hello World';
```
打包后

![](https://segmentfault.com/img/bVEHy5?w=193&h=136)

bundle.js
```javascript
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

/******/ 	};

/******/ 	// The module cache
/******/ 	var installedModules = {};

/******/ 	// object to store loaded and loading chunks
/******/ 	// "0" means "already loaded"
/******/ 	// Array means "loading", array contains callbacks
/******/ 	var installedChunks = {
/******/ 		0:0
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

/******/ 			script.src = __webpack_require__.p + "" + chunkId + ".bundle.js";
/******/ 			head.appendChild(script);
/******/ 		}
/******/ 	};

/******/ 	// expose the modules object (__webpack_modules__)
/******/ 	__webpack_require__.m = modules;

/******/ 	// expose the module cache
/******/ 	__webpack_require__.c = installedModules;

/******/ 	// __webpack_public_path__
/******/ 	__webpack_require__.p = "";

/******/ 	// Load entry module and return exports
/******/ 	return __webpack_require__(0);
/******/ })
/************************************************************************/
/******/ ([
/* 0 */
/***/ function(module, exports, __webpack_require__) {

	__webpack_require__.e/* nsure */(1, function(require) {
	  var content = __webpack_require__(1);
	  document.open();
	  document.write('<h1>' + content + '</h1>');
	  document.close();
	});


/***/ }
/******/ ]);
```
1.bundle.js
```javascript
webpackJsonp([1],[
/* 0 */,
/* 1 */
/***/ function(module, exports) {

	module.exports = 'Hello World';


/***/ }
]);
```
和使用CommonsChunkPlugin打包的差异在于
```javascript
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

/******/ 			script.src = __webpack_require__.p + "" + chunkId + ".bundle.js";
/******/ 			head.appendChild(script);
/******/ 		}
/******/ 	};
```
模块main的id为0，模块a的id为1。return __webpack_require__(0)，则加载main模块，
modules[0].call(module.exports, module, module.exports, __webpack_require__)则调用函数
```javascript
function(module, exports, __webpack_require__) {

	__webpack_require__.e/* nsure */(1, function(require) {
	  var content = __webpack_require__(1);
	  document.open();
	  document.write('<h1>' + content + '</h1>');
	  document.close();
	}
```
```javascript
/******/ 	// This file contains only the entry chunk.
/******/ 	// The chunk loading function for additional chunks
/******/ 	__webpack_require__.e = function requireEnsure(chunkId, callback) {
	            //installedChunks[1]为undefined
/******/ 		// "0" is the signal for "already loaded"
/******/ 		if(installedChunks[chunkId] === 0)
/******/ 			return callback.call(null, __webpack_require__);

/******/ 		// an array means "currently loading".
/******/ 		if(installedChunks[chunkId] !== undefined) {
/******/ 			installedChunks[chunkId].push(callback);
/******/ 		} else {
/******/ 			// start chunk loading
/******/ 			installedChunks[chunkId] = [callback];//installedChunks[1]为数组，表明currently loading
/******/ 			var head = document.getElementsByTagName('head')[0];
/******/ 			var script = document.createElement('script');
/******/ 			script.type = 'text/javascript';
/******/ 			script.charset = 'utf-8';
/******/ 			script.async = true;

/******/ 			script.src = __webpack_require__.p + "" + chunkId + ".bundle.js";
/******/ 			head.appendChild(script);
					//加载完后直接调用
					/******/webpackJsonp([1],[
					/******//* 0 */,
					/******//* 1 */
					/******//***/ function(module, exports) {
					/******/
					/******/	module.exports = 'Hello World';
					/******/
					/******/
					/******//***/ }
					/******/]);
					/******/ 		}
					/******/ 	};
					//installedChunks[1]在webpackJsonp得到调用
```
installedChunks[1]为数组，元素为main模块的执行代码
```javascript
/******/ 	window["webpackJsonp"] = function webpackJsonpCallback(chunkIds, moreModules) {
	            //moreModules为模块a的代码
/******/ 		// add "moreModules" to the modules object,
/******/ 		// then flag all "chunkIds" as loaded and fire callback
/******/ 		var moduleId, chunkId, i = 0, callbacks = [];
/******/ 		for(;i < chunkIds.length; i++) {
/******/ 			chunkId = chunkIds[i];
/******/ 			if(installedChunks[chunkId])//installedChunks[0]==0,installedChunks[1]为数组
/******/ 				callbacks.push.apply(callbacks, installedChunks[chunkId]);//callbacks为模块main执行代码，不为数组
/******/ 			installedChunks[chunkId] = 0;//installedChunks[1]不为数组，表明已经加载
/******/ 		}
/******/ 		for(moduleId in moreModules) {
/******/ 			modules[moduleId] = moreModules[moduleId];
/******/ 		}
/******/ 		if(parentJsonpFunction) parentJsonpFunction(chunkIds, moreModules);
/******/ 		while(callbacks.length)
/******/ 			callbacks.shift().call(null, __webpack_require__);

/******/ 	};
```