title: 傻瓜式解读koa中间件处理模块koa-compose
author: 若邪
tags:
  - koa
  - compose
  - 中间件
categories:
  - 前端
date: 2018-10-29 00:00:00
---
最近需要单独使用到koa-compose这个模块，虽然使用koa的时候大致知道中间件的执行流程，但是没仔细研究过源码用起来还是不放心(主要是这个模块代码少，多的话也没兴趣去研究了)。

koa-compose看起来代码少，但是确实绕。闭包，递归，Promise。。。看了一遍脑子里绕不清楚。看了网上几篇解读文章，都是针对单行代码做解释，还是绕不清楚。最后只好采取一种傻瓜的方式：

koa-compose去掉一些注释，类型校验后，源码如下：

```js
function compose (middleware) {
  return function (context, next) {
    // last called middleware #
    let index = -1
    return dispatch(0)
    function dispatch (i) {
      if (i <= index) return Promise.reject(new Error('next() called multiple times'))
      index = i
      let fn = middleware[i]
      if (i === middleware.length) fn = next
      if (!fn) return Promise.resolve()
      try {
        return Promise.resolve(fn(context, dispatch.bind(null, i + 1)));
      } catch (err) {
        return Promise.reject(err)
      }
    }
  }
}
```

写出如下代码：
```js
var index = -1;
function compose() {
    return dispatch(0)
}
function dispatch (i) {
      if (i <= index) return Promise.reject(new Error('next() called multiple times'))
      index = i
      var fn = middleware[i]
      if (i === middleware.length) fn = next
      if (!fn) return Promise.resolve('fn is undefined')
      try {
        return Promise.resolve(fn(context, dispatch.bind(null, i + 1)));
      } catch (err) {
        return Promise.reject(err)
      }
 }
 
 function f1(context,next){
    console.log('middleware 1');
    next().then(data=>console.log(data));
    console.log('middleware 1');
    return 'middleware 1 return';
  }
  function f2(context,next){
    console.log('middleware 2');
    next().then(data=>console.log(data));
    console.log('middleware 2');
    return 'middleware 2 return';
  }
  function f3(context,next){
    console.log('middleware 3');
    next().then(data=>console.log(data));
    console.log('middleware 3');
    return 'middleware 3 return';
  }
var middleware=[
  f1,f2,f3
]

var context={};
var next=function(context,next){
    console.log('middleware 4');
    next().then(data=>console.log(data));
    console.log('middleware 4');
    return 'middleware 4 return';
};
compose().then(data=>console.log(data));
```
直接运行结果如下：

"middleware 1"

"middleware 2"

"middleware 3"

"middleware 4"

"middleware 4"

"middleware 3"

"middleware 2"

"middleware 1"

"fn is undefined"

"middleware 4 return"

"middleware 3 return"

"middleware 2 return"

"middleware 1 return"


按着代码运行流程一步步分析：

> `dispatch(0)`

> i==0,index==-1 i>index 往下

> `index=0`

> `fn=f1`

> `Promise.resolve(f1(context, dispatch.bind(null, 0 + 1)))` 

> 这就会执行

> `f1(context, dispatch.bind(null, 0 + 1))`

> 进入到f1执行上下文

> `console.log('middleware 1');`

输出middleware 1

> `next()` 

> 其实就是调用`dispatch(1)` bind的功劳

递归开始

> `dispatch(1)`

> i==1,index==0 i>index 往下

> `index=1`

> `fn=f2`

> `Promise.resolve(f2(context, dispatch.bind(null, 1 + 1)))` 

> 这就会执行

> `f2(context, dispatch.bind(null, 1 + 1))`

> 进入到f2执行上下文

> `console.log('middleware 2');`

输出middleware 2

> `next()` 

> 其实就是调用`dispatch(2)`

接着递归

> `dispatch(2)`

> i==2,index==1 i>index 往下

> `index=2`

> `fn=f3`

> `Promise.resolve(f3(context, dispatch.bind(null, 2 + 1)))` 

> 这就会执行

> `f3(context, dispatch.bind(null, 2 + 1))`

> 进入到f3执行上下文

> `console.log('middleware 3');`

输出middleware 3

> `next()` 

> 其实就是调用`dispatch(3)`

接着递归

> `dispatch(3)`

> i==3,index==2 i>index 往下

> `index=3`

> **`i === middleware.length`**

> **`fn=next`**

> `Promise.resolve(next(context, dispatch.bind(null, 3 + 1)))` 

> 这就会执行

> `next(context, dispatch.bind(null, 3 + 1))`

> 进入到next执行上下文

> `console.log('middleware 4');`

输出middleware 4

> `next()` 

> 其实就是调用`dispatch(4)`

接着递归

> `dispatch(4)`

> i==4,index==3 i>index 往下

> `index=4`

> **`fn=middleware[4]`**

> **`fn=undefined`**

> `reuturn Promise.resolve('fn is undefined')` 

> 回到next执行上下文

> `console.log('middleware 4');`

输出middleware 4

>`return 'middleware 4 return'`

>`Promise.resolve('middleware 4 return')`

> 回到f3执行上下文

> `console.log('middleware 3');`

输出middleware 3

>`return 'middleware 3 return'`

>`Promise.resolve('middleware 3 return')`

> 回到f2执行上下文

> `console.log('middleware 2');`

输出middleware 2

>`return 'middleware 2 return'`

>`Promise.resolve('middleware 2 return')`

> 回到f1执行上下文

> `console.log('middleware 1');`

输出middleware 1

>`return 'middleware 1 return'`

>`Promise.resolve('middleware 1 return')`

>回到全局上下文

至此已经输出

"middleware 1"

"middleware 2"

"middleware 3"

"middleware 4"

"middleware 4"

"middleware 3"

"middleware 2"

"middleware 1"

那么

"fn is undefined"

"middleware 4 return"

"middleware 3 return"

"middleware 2 return"

"middleware 1 return"

怎么来的呢

回头看一下，每个中间件里都有

`next().then(data=>console.log(data));`

按照之前的分析，then里最先拿到结果的应该是next中间件的，而且结果就是`Promise.resolve('fn is undefined')`的结果，然后分别是f4,f3,f2,f1。那么为什么都是最后才输出呢？

```js
Promise.resolve('fn is undefined').then(data=>console.log(data));
console.log('middleware 4');
```

运行一下就清楚了

或者
```js
setTimeout(()=>console.log('fn is undefined'),0);
console.log('middleware 4');
```

整个调用过程还可以看成是这样的：
```js
function composeDetail(){
  return Promise.resolve(
    f1(context,function(){
      return Promise.resolve(
        f2(context,function(){
          return Promise.resolve(
            f3(context,function(){
              return Promise.resolve(
                next(context,function(){
                  return Promise.resolve('fn is undefined')
                })
              )
            })
          )
        })
      )
    })
  )
}
composeDetail().then(data=>console.log(data));
```

方法虽蠢，但是compose的作用不言而喻了

最后，`if (i <= index) return Promise.reject(new Error('next() called multiple times'))`这句代码何时回其作用呢？

一个中间件里调用两次`next()`，按照上面的套路走，相信很快就明白了。


























