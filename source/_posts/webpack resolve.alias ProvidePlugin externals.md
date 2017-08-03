title: webpack resolve.alias ProvidePlugin externals
author: 若邪
tags:
 - webpack
categories:
   - 前端
date: 2016-12-27
---
>resolve.alias这个配置项相当于为文件目录配置一个别名

比如下面这样的目录结构
![](http://upload-images.jianshu.io/upload_images/2125695-3636aa829ece4cda.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

要在main.js中使用jquery,需要这样``var $=require("./lib/jquery")``。如果lib中的库很多，而且目录也很多，使用的时候就要写一长串的地址。

使用resolve.alias配置如下
```javascript

module.exports = {
  entry:
  {
    main:'./main.js',
  },
  output: {
    path:__dirname+'/dist',
    filename: '[name].js'
  },
  resolve:{
    //配置别名，在项目中可缩减引用路径
    alias: {
      jquery: "./lib/jquery"
    }
  },
  plugins: [

  ]
};
```
使用的时候，这样就可以``var $=require("jquery");``
配置项中，key值得配置方式也有很多种，更多的可以看[这里](http://webpack.github.io/docs/configuration.html#resolve)

>resolve.alias使我们不用频繁地写一长串的引用路径，但是使用的时候还是先要require，如果我们懒到require都不想写呢？ProvidePlugin这个插件就派上用场了。

webpack.config.js
```javascript
var webpack=require("webpack");
module.exports = {
  entry:
  {
    main:'./main.js',
  },
  output: {
    path:__dirname+'/dist',
    filename: '[name].js'
  },
  resolve:{
    //配置别名，在项目中可缩减引用路径
    alias: {
      jquery: "./lib/jquery"
    }
  },
  plugins: [
    //提供全局的变量，在模块中使用无需用require引入
    new webpack.ProvidePlugin({
      $: "jquery"
    }),
  ]
};
```
因为已经配置的别名，所以
```javascript
new webpack.ProvidePlugin({ $: "jquery" })
```
就可以，jquery就是我们配置的别名，如果没有配置别名，则要这样写
```javascript
new webpack.ProvidePlugin({ $: "./lib/jquery" })
```
使用的时候
```javascript
var arr=[1,2,3,4];
$.each(arr,function(){
    console.log(this);
});
```
没毛病，但是如果没有配置ProvidePlugin，也没有require,这样写webpack打包的时候是不会报错的，浏览器运行的时候才知道错误。

>不管是使用resolve.alias还是ProvidePlugin，打包的时候，webpack都会将使用到的库进行打包。打包的方式可以使用CommonsChunkPlugin这个插件再进行配置（我以前的文章中有写这个插件的详细用法）。如果我们不想webpack打包某个文件，而是直接在页面使用script标签手动引入，或者使用CDN资源的时候，externals这个配置项就起作用了。

webpack.config.js
```javascript

module.exports = {
  entry:
  {
    main:'./main.js',
  },
  output: {
    path:__dirname+'/dist',
    filename: '[name].js'
  },
  externals: { $: "window.jQuery" },
  plugins: [

  ]
};
```
使用
```javascript
var $ = require("$");
var arr=[1,2,3,4];
$.each(arr,function(){
    console.log(this);
});

```
一定要记得require,不然和不配置externals没区别，不想写可以使用ProvidePlugin
```javascript
externals: { $: "window.jQuery" },
  plugins: [
    //提供全局的变量，在模块中使用无需用require引入
    new webpack.ProvidePlugin({
      $: "$"
    })
  ]
```

打包后
![](http://upload-images.jianshu.io/upload_images/2125695-d74bd937b98805bf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
然后页面上通过script标签手动引入CDN地址或者本地文件地址就行了，需要注意的是引入的顺序和依赖关系，将webpack打包的文件放到后面引入。

其实不使用externals也是可以的，我们看一下不使用externals，直接这样写
```javascript
var arr=[1,2,3,4];
$.each(arr,function(){
    console.log(this);
});
```
打包后

![](http://upload-images.jianshu.io/upload_images/2125695-46b49a86f084eaee.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当我们手动引入JQ后，$肯定是有的，没毛病。但始终感觉这样会给自己挖坑。