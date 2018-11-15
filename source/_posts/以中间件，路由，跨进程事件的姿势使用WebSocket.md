title: 以中间件，路由，跨进程事件的姿势使用WebSocket
author: 若邪
tags:
  - WebSocket
categories:
  - 前端
date: 2018-11-05 00:00:00
---
通过参考koa中间件，socket.io远程事件调用，以一种新的姿势来使用WebSocket。

## 浏览器端

浏览器端使用WebSocket很简单
```js
// Create WebSocket connection.
const socket = new WebSocket('ws://localhost:8080');

// Connection opened
socket.addEventListener('open', function (event) {
    socket.send('Hello Server!');
});

// Listen for messages
socket.addEventListener('message', function (event) {
    console.log('Message from server ', event.data);
});
```
[MDN](https://developer.mozilla.org/zh-CN/docs/Web/API/WebSocket)关于WebSocket的介绍

能注册的事件有onclose，onerror，onmessage，onopen。用的比较多的是onmessage，从服务器接受到数据后，会触发message事件。通过注册相应的事件处理函数，可以根据后端推送的数据做相应的操作。

如果只是写个demo,单单输出后端推送的信息,如下使用即可：
```js
socket.addEventListener('message', function (event) {
    console.log('Message from server ', event.data);
});
```

实际使用过程中，我们需要判断后端推送的数据然后执行相应的操作。比如聊天室应用中，需要判断消息是广播的还是私聊的或者群聊的，以及是纯文字信息还是图片等多媒体信息。这时message处理函数里可能就是一堆的if else。那么有没有什么别的优雅的姿势呢？答案就是中间件与事件，跨进程的事件的发布与订阅。在说远程事件发布订阅之前，需要先从中间件开始，因为后面实现的远程事件发布订阅是基于中间件的。

## 中间件

前面说了，在WebSocket实例上可以注册事件有onclose，onerror，onmessage，onopen。每一个事件的处理函数里可能需要做各种判断，特别是message事件。参考koa，可以将事件处理函数以中间件方式来进行使用，将不同的操作逻辑分发到不同的中间件中，比如聊天室应用中，聊天信息与系统信息(比如用户登录属于系统信息)是可以放到不同的中间件中处理的。

koa提供use接口来注册中间件。我们针对不同的事件提供相应的中间件注册接口，并且对原生的WebSocket做封装。

```js
export default class EasySocket{
    constructor(config) {
       this.url = config.url;
       this.openMiddleware = [];
       this.closeMiddleware = [];
       this.messageMiddleware = [];
       this.errorMiddleware = [];
       
       this.openFn = Promise.resolve();
       this.closeFn = Promise.resolve();
       this.messageFn = Promise.resolve();
       this.errorFn = Promise.resolve();
    }
    openUse(fn) {
        this.openMiddleware.push(fn);
        return this;
    }
    closeUse(fn) {
        this.closeMiddleware.push(fn);
        return this;
    }
    messageUse(fn) {
        this.messageMiddleware.push(fn);
        return this;
    }
    errorUse(fn) {
        this.errorMiddleware.push(fn);
        return this;
    }
}
```

通过`xxxUse`注册相应的中间件。 `xxxMiddleware`中就是相应的中间件。`xxxFn` 中间件通过compose处理后的结构

再添加一个connect方法，处理相应的中间件并且实例化原生WebSocket
```js
connect(url) {
        this.url = url || this.url;
        if (!this.url) {
            throw new Error('url is required!');
        }
        try {
            this.socket = new WebSocket(this.url, 'echo-protocol');
        } catch (e) {
            throw e;
        }

        this.openFn = compose(this.openMiddleware);
        this.socket.addEventListener('open', (event) => {
            let context = { client: this, event };
            this.openFn(context).catch(error => { console.log(error) });
        });

        this.closeFn = compose(this.closeMiddleware);
        this.socket.addEventListener('close', (event) => {
            let context = { client: this, event };
            this.closeFn(context).then(() => {
            }).catch(error => {
                console.log(error)
            });
        });

        this.messageFn = compose(this.messageMiddleware);
        this.socket.addEventListener('message', (event) => {
            let res;
            try {
                res = JSON.parse(event.data);
            } catch (error) {
                res = event.data;
            }
            let context = { client: this, event, res };
            this.messageFn(context).then(() => {

            }).catch(error => {
                console.log(error)
            });
        });

        this.errorFn = compose(this.errorMiddleware);
        this.socket.addEventListener('error', (event) => {
            let context = { client: this, event };
            this.errorFn(context).then(() => {

            }).catch(error => {
                console.log(error)
            });
        });
        return this;
    }
```

使用[koa-compose](https://github.com/koajs/compose)模块处理中间件。注意context传入了哪些东西，后续定义中间件的时候都可以使用。

>compose的作用可看这篇文章  [傻瓜式解读koa中间件处理模块koa-compose](https://juejin.im/post/5bd7238d51882579201028b0)

然后就可以使用了：
```js
new EasySocket()
  .openUse((context, next) => {
    console.log("open");
    next();
  })
  .closeUse((context, next) => {
    console.log("close");
    next();
  })
  .errorUse((context, next) => {
    console.log("error", context.event);
    next();
  })
  .messageUse((context, next) => {
    //用户登录处理中间件
    if (context.res.action === 'userEnter') {
      console.log(context.res.user.name+' 进入聊天室');
    }
    next();
  })
  .messageUse((context, next) => {
    //创建房间处理中间件
    if (context.res.action === 'createRoom') {
      console.log('创建房间 '+context.res.room.anme);
    }
    next();
  })
  .connect('ws://localhost:8080')
```

可以看到，用户登录与创建房间的逻辑放到两个中间件中分开处理。不足之处就是每个中间件都要判断context.res.action，而这个context.res就是后端返回的数据。怎么消除这个频繁的if判断呢? 我们实现一个简单的消息处理路由。

## 路由

定义消息路由中间件

messageRouteMiddleware.js

```js
export default (routes) => {
    return async (context, next) => {
        if (routes[context.res.action]) {
            await routes[context.res.action](context,next);
        } else {
            console.log(context.res)
            next();
        }
    }
}

```

定义路由

router.js
```js
export default {
    userEnter:function(context,next){
        console.log(context.res.user.name+' 进入聊天室');
        next();
    },
    createRoom:function(context,next){
        console.log('创建房间 '+context.res.room.anme);
        next();
    }
}
```

使用：
```js
new EasySocket()
  .openUse((context, next) => {
    console.log("open");
    next();
  })
  .closeUse((context, next) => {
    console.log("close");
    next();
  })
  .errorUse((context, next) => {
    console.log("error", context.event);
    next();
  })
  .messageUse(messageRouteMiddleware(router))//使用消息路由中间件，并传入定义好的路由
  .connect('ws://localhost:8080')
```

一切都变得美好了，感觉就像在使用koa。想一个问题，当接收到后端推送的消息时，我们需要做相应的DOM操作。比如路由里面定义的userEnter，我们可能需要在对应的函数里操作用户列表的DOM，追加新用户。这使用原生JS或JQ都是没有问题的，但是如果使用vue,react这些，因为是组件化的，用户列表可能就是一个组件，怎么访问到这个组件实例呢？(当然也可以访问vuex,redux的store,但是并不是所有组件的数据都是用store管理的)。


我们需要一个运行时注册中间件的功能，然后在组件的相应的生命周期钩子里注册中间件并且传入组件实例

运行时注册中间件，修改如下代码：
```js
messageUse(fn, runtime) {
        this.messageMiddleware.push(fn);
        if (runtime) {
            this.messageFn = compose(this.messageMiddleware);
        }
        return this;
    }
```

修改 messageRouteMiddleware.js

```js
export default (routes,component) => {
    return async (context, next) => {
        if (routes[context.req.action]) {
            context.component=component;//将组件实例挂到context下
            await routes[context.req.action](context,next);
        } else {
            console.log(context.req)
            next();
        }
    }
}

```

类似vue mounted中使用

```js
mounted(){
  let client = this.$wsClients.get("im");//获取指定EasySocket实例
  client.messageUse(messageRouteMiddleware(router,this),true)//运行时注册中间件，并传入定义好的路由以及当前组件中的this
}
```

路由中通过 context.component 即可访问到当前组件。


完美了吗？每次组件mounted 都注册一次中间件，问题很大。所以需要一个判断中间件是否已经注册的功能。也就是一个支持具名注册中间件的功能。这里就暂时不实现了，走另外一条路，也就是之前说到的远程事件的发布与订阅，我们也可以称之为跨进程事件。

## 跨进程事件

看一段socket.io的代码：

Server (app.js)

```js
var app = require('http').createServer(handler)
var io = require('socket.io')(app);
var fs = require('fs');
app.listen(80);
function handler (req, res) {
  fs.readFile(__dirname + '/index.html',
  function (err, data) {
    if (err) {
      res.writeHead(500);
      return res.end('Error loading index.html');
    }

    res.writeHead(200);
    res.end(data);
  });
}
io.on('connection', function (socket) {
  socket.emit('news', { hello: 'world' });
  socket.on('my other event', function (data) {
    console.log(data);
  });
});
```

Client (index.html)
```js
<script src="/socket.io/socket.io.js"></script>
<script>
  var socket = io('http://localhost');
  socket.on('news', function (data) {
    console.log(data);
    socket.emit('my other event', { my: 'data' });
  });
</script>
```

注意力转到这两部分：

服务端
```js
  socket.emit('news', { hello: 'world' });
  socket.on('my other event', function (data) {
    console.log(data);
  });
```

客户端
```js
  var socket = io('http://localhost');
  socket.on('news', function (data) {
    console.log(data);
    socket.emit('my other event', { my: 'data' });
  });
```

使用事件，客户端通过on订阅'news'事件，并且当触发‘new’事件的时候通过emit发布'my other event'事件。服务端在用户连接的时候发布'news'事件，并且订阅'my other event'事件。

一般我们使用事件的时候，都是在同一个页面中on和emit。而socket.io的神奇之处就是同一事件的on和emit是分别在客户端和服务端，这就是跨进程的事件。

那么，在某一端emit某个事件的时候，另一端如果on监听了此事件，是如何知道这个事件emit(发布)了呢？

没有看socket.io源码之前，我设想应该是emit方法里做了某些事情。就像java或c#，实现rpc的时候，可以依据接口定义动态生成实现(也称为代理)，动态实现的(代理)方法中，就会将当前方法名称以及参数通过相应协议进行序列化，然后通过http或者tcp等网络协议传输到RPC服务端，服务端进行反序列化，通过反射等技术调用本地实现，并返回执行结果给客户端。客户端拿到结果后，整个调用完成，就像调用本地方法一样实现了远程方法的调用。

看了socket.io emit的代码实现后，思路也是大同小异，通过将当前emit的事件名和参数按一定规则组合成数据，然后将数据通过WebSocket的send方法发送出去。接收端按规则取到事件名和参数，然后本地触发emit。(注意远程emit和本地emit，socket.io中直接调用的是远程emit)。

下面是实现代码，事件直接用的[emitter](https://github.com/component/emitter)模块，并且为了能自定义emit事件名和参数组合规则，以中间件的方式提供处理方法：

```js
export default class EasySocket extends Emitter{//继承Emitter
    constructor(config) {
       this.url = config.url;
       this.openMiddleware = [];
       this.closeMiddleware = [];
       this.messageMiddleware = [];
       this.errorMiddleware = [];
       this.remoteEmitMiddleware = [];//新增的部分
       
       this.openFn = Promise.resolve();
       this.closeFn = Promise.resolve();
       this.messageFn = Promise.resolve();
       this.errorFn = Promise.resolve();
       this.remoteEmitFn = Promise.resolve();//新增的部分
    }
    openUse(fn) {
        this.openMiddleware.push(fn);
        return this;
    }
    closeUse(fn) {
        this.closeMiddleware.push(fn);
        return this;
    }
    messageUse(fn) {
        this.messageMiddleware.push(fn);
        return this;
    }
    errorUse(fn) {
        this.errorMiddleware.push(fn);
        return this;
    }
    //新增的部分
    remoteEmitUse(fn, runtime) {
        this.remoteEmitMiddleware.push(fn);
        if (runtime) {
            this.remoteEmitFn = compose(this.remoteEmitMiddleware);
        }
        return this;
    }
    connect(url) {
       ...
       //新增部分
       this.remoteEmitFn = compose(this.remoteEmitMiddleware);
    }
    //重写emit方法，支持本地调用以远程调用
    emit(event, args, isLocal = false) {
        let arr = [event, args];
        if (isLocal) {
            super.emit.apply(this, arr);
            return this;
        }
        let evt = {
            event: event,
            args: args
        }
        let remoteEmitContext = { client: this, event: evt };
        this.remoteEmitFn(remoteEmitContext).catch(error => { console.log(error) })
        return this;
    }
}
```

下面是一个简单的处理中间件：

```js
client.remoteEmitUse((context, next) => {
    let client = context.client;
    let event = context.event;
    if (client.socket.readyState !== 1) {
      alert("连接已断开!");
    } else {
      client.socket.send(JSON.stringify({
        type: 'event',
        event: event.event,
        args: event.args
      }));
      next();
    }
  })
```
意味着调用
```js
client.emit('chatMessage',{
    from:'admin',
    masg:"Hello WebSocket"
});
```
就会组合成数据
```js
{
    type: 'event',
    event: 'chatMessage',
    args: {
        from:'admin',
        masg:"Hello WebSocket"
    }
}
```
发送出去。

服务端接受到这样的数据，可以做相应的数据处理（后面会使用nodejs实现类似的编程模式），也可以直接发送给别的客户端。客户受到类似的数据，可以写专门的中间件进行处理，比如：
```js
client.messageUse((context, next) => {
    if (context.res.type === 'event') {
      context.client.emit(context.res.event, context.res.args, true);//注意这里的emit是本地emit。
    }
    next();
})
```

如果本地订阅的chatMessage事件，回到函数就会被触发。

在vue或react中使用，也会比之前使用路由的方式简单

```js
mounted() {
   let client = this.$wsClients.get("im");
   client.on("chatMessage", data => {
      let isSelf = data.from.id == this.user.id;
      let msg = {
        name: data.from.name,
        msg: data.msg,
        createdDate: data.createdDate,
        isSelf
      };
      this.broadcastMessageList.push(msg);
    });
}

```
组件销毁的时候移除相应的事件订阅即可，或者清空所有事件订阅

```js
destroyed() {
    let client = this.$wsClients.get("im");
    client.removeAllListeners();
}

```

## 心跳重连

核心代码直接从[websocket-heartbeat-js](https://github.com/zimv/websocket-heartbeat-js) copy过来的(用npm包，还得在它的基础上再包一层)，相关文章 [初探和实现websocket心跳重连](https://www.cnblogs.com/1wen/p/5808276.html)。

核心代码：
```js
    heartCheck() {
        this.heartReset();
        this.heartStart();
    }
    heartStart() {
        this.pingTimeoutId = setTimeout(() => {
            //这里发送一个心跳，后端收到后，返回一个心跳消息
            this.socket.send(this.pingMsg);
            //接收到心跳信息说明连接正常,会执行heartCheck(),重置心跳(清除下面定时器)
            this.pongTimeoutId = setTimeout(() => {
                //此定时器有运行的机会，说明发送ping后，设置的超时时间内未收到返回信息
                this.socket.close();//不直接调用reconnect，避免旧WebSocket实例没有真正关闭，导致不可预料的问题
            }, this.pongTimeout);
        }, this.pingTimeout);
    }
    heartReset() {
        clearTimeout(this.pingTimeoutId);
        clearTimeout(this.pongTimeoutId);
    }
```

## 最后

源码地址：[easy-socket-browser](https://github.com/wjkang/easy-socket-browser)

nodejs实现的类似的编程模式(有空再细说)：[easy-socket-node](https://github.com/wjkang/easy-socket-node)

实现的聊天室例子：[online chat demo](http://jaycewu.coding.me/easy-socket-chat/#/) 

聊天室前端源码：[lazy-mock-im](https://github.com/wjkang/lazy-mock-im)

聊天室服务端源码：[lazy-mock](https://github.com/wjkang/lazy-mock/tree/chat)















