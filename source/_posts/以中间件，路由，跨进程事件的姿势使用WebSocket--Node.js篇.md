title: 以中间件，路由，跨进程事件的姿势使用WebSocket--Node.js篇
author: 若邪
tags:
  - WebSocket
categories:
  - 前端
  - 后端
date: 2018-11-09 00:00:00
---
上一篇文章介绍了在浏览器端以中间件，路由，跨进程事件的姿势使用原生WebSocket。这篇文章将介绍如何使用Node.js以相同的编程模式来实现WebSocket服务端。

Node.js中比较流行的两个WebSocket库分别是[socket.io](https://github.com/socketio/socket.io)与[ws](https://github.com/websockets/ws)。其中socket.io已经实现了跨进程事件，广播，群发等功能，并且服务端与浏览器端是配套的，在不支持WebSocket技术的浏览器会降级为使用ajax轮询。所以。这里选择使用相对而言较为底层或原始的ws,在其基础上实现文章标题所提到的编程模式。

## WS

使用ws简简单单就可以启动一个WebSocket服务：
```js
const WebSocket = require('ws');

const wss = new WebSocket.Server({ port: 8080 });

wss.on('connection', function connection(ws) {
  ws.on('message', function incoming(message) {
    console.log('received: %s', message);
  });

  ws.send('something');
});
```

上面的wss支持的事件有`connection`,`close`,`error`,`headers`,`listening`,ws支持的事件有`message`,`close`,`error`。更多详情可以[这里](https://github.com/websockets/ws/blob/master/doc/ws.md)。

## 中间件

对ws进行封装，上面提到的事件：WSS的`connection`，ws的`message`,`close`,`error`。分别提供注册中间件的接口

```js
class EasySocket {
    constructor() {
        this.connectionMiddleware = [];
        this.closeMiddleware = [];
        this.messageMiddleware = [];
        this.errorMiddleware = [];

        this.connectionFn = Promise.resolve();
        this.closeFn = Promise.resolve();
        this.messageFn = Promise.resolve();
        this.errorFn = Promise.resolve();
    }
    connectionUse(fn, runtime) {
        this.connectionMiddleware.push(fn);
        if (runtime) {
            this.connectionFn = compose(this.connectionMiddleware);
        }
        return this;
    }
    closeUse(fn, runtime) {
        this.closeMiddleware.push(fn);
        if (runtime) {
            this.closeFn = compose(this.closeMiddleware);
        }
        return this;
    }
    messageUse(fn, runtime) {
        this.messageMiddleware.push(fn);
        if (runtime) {
            this.messageFn = compose(this.messageMiddleware);
        }
        return this;
    }
    errorUse(fn, runtime) {
        this.errorMiddleware.push(fn);
        if (runtime) {
            this.errorFn = compose(this.errorMiddleware);
        }
        return this;
    }
    
}
```

通过xxxUse注册相应的中间件。 xxxMiddleware中就是相应的中间件。xxxFn是中间件通过compose处理后的结构。使用runtime参数可以在运行时注册中间件。

再添加一个listen方法，处理相应的中间件并且实例化WebSocket.Server

```js
listen(config) {
        this.socket = new WebSocket.Server(config);
        this.connectionFn = compose(this.connectionMiddleware);
        this.messageFn = compose(this.messageMiddleware);
        this.closeFn = compose(this.closeMiddleware);
        this.errorFn = compose(this.errorMiddleware);
        this.socket.on('connection', (client, req) => {
            let context = { server: this, client, req };
            this.connectionFn(context).catch(error => { console.log(error) });

            client.on('message', (message) => {
                let req;
                try {
                    req = JSON.parse(message);
                } catch (error) {
                    req = message;
                }
                let messageContext = { server: this, client, req }
                this.messageFn(messageContext).catch(error => { console.log(error) })
            });

            client.on('close', (code, message) => {
                let closeContext = { server: this, client, code, message };
                this.closeFn(closeContext).catch(error => { console.log(error) })
            });

            client.on('error', (error) => {
                let errorContext = { server: this, client, error };
                this.errorFn(errorContext).catch(error => { console.log(error) })
            });
        })
    }
```

使用[koa-compose](https://github.com/koajs/compose)模块处理中间件。注意xxContext传入了哪些东西，后续定义中间件的时候都可以使用。

>compose的作用可看这篇文章  [傻瓜式解读koa中间件处理模块koa-compose](https://juejin.im/post/5bd7238d51882579201028b0)

使用：
```js
import EasySocket from 'easy-socket-node';

const config = {
    port: 3001,
    perMessageDeflate: {
        zlibDeflateOptions: { // See zlib defaults.
            chunkSize: 1024,
            memLevel: 7,
            level: 3,
        },
        zlibInflateOptions: {
            chunkSize: 10 * 1024
        },
        // Other options settable:
        clientNoContextTakeover: true, // Defaults to negotiated value.
        serverNoContextTakeover: true, // Defaults to negotiated value.
        //clientMaxWindowBits: 10,       // Defaults to negotiated value.
        serverMaxWindowBits: 10,       // Defaults to negotiated value.
        // Below options specified as default values.
        concurrencyLimit: 10,          // Limits zlib concurrency for perf.
        threshold: 1024,               // Size (in bytes) below which messages
        // should not be compressed.
    }
}
const easySocket = new EasySocket();
//使用中间件获取token
easySocket
    .connectionUse((context,next)=>{
       console.log("new Connected");
       let location = url.parse(context.req.url, true);
       let token=location.query.token;
       if(!token){
           client.send("invalid token");
           client.close(1003, "invalid token");
           return;
       }
       context.client.token=token;
       next();
    });
easySocket
    .listen(config)

console.log('Now start WebSocket server on port ' + config.port + '...')
```

使用messageUse可以注册多个处理消息的中间件，比如

```js
 easySocket.messageUse((context, next) => {
    //群聊处理中间件
    if (context.req.action === 'roomChatMessage') {
      //可以在这里持久化消息，将消息发送给其它群聊客户端
      console.log(context.req);
    }
    next();
  })
  .messageUse((context, next) => {
    //私聊处理中间件
    if (context.req.action === 'privateChatMessage') {
      //可以在这里持久化消息，将消息发送给私聊客户端
      console.log(context.req);
    }
    next();
  })
```
每个中间件都要判断context.req.action，而这个context.res就是浏览器端或客户端发送的数据。怎么消除这个频繁的if判断呢? 我们实现一个简单的消息处理路由。

## 路由

定义消息路由中间件

messageRouteMiddleware.js

```js
export default (routes) => {
    return async (context, next) => {
        if (routes[context.req.action]) {
            await routes[context.req.action](context,next);
        } else {
            console.log(context.req)
            next();
        }
    }
}

```

定义路由

router.js
```js
export default {
    roomChatMessage:function(context,next){
        //可以在这里持久化消息，将消息发送给其它群聊客户端，以及其它业务逻辑
        console.log(context.req);
        next();
    },
    privateChatMessage:function(context,next){
        //可以在这里持久化消息，将消息发送给私聊客户端，以及其它业务逻辑
        console.log(context.req);
        next();
    }
}
```

使用：
```js
easySocket.messageUse(messageRouteMiddleware(router))
```

## 跨进程事件

上一篇文章已经介绍了跨进程事件，这里直接说实现。

使用Node的原生事件模块
```js
import compose from './compose';
const WebSocket = require('ws');
var EventEmitter = require('events').EventEmitter;
export default class EasySocket extends EventEmitter {
    constructor() {
        ...
        this.remoteEmitMiddleware = [];

        ...
        this.remoteEmitFn = Promise.resolve();
    }
    ...
    remoteEmitUse(fn, runtime) {
        this.remoteEmitMiddleware.push(fn);
        if (runtime) {
            this.remoteEmitFn = compose(this.remoteEmitMiddleware);
        }
        return this;
    }
    listen(config) {
        this.socket = new WebSocket.Server(config);
        ...
        this.remoteEmitFn = compose(this.remoteEmitMiddleware);

        ...
    }
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
        let remoteEmitContext = { server: this, event: evt };
        this.remoteEmitFn(remoteEmitContext).catch(error => { console.log(error) })
        return this;
    }
}
```

## 最后

源码地址：[easy-socket-node](https://github.com/wjkang/easy-socket-node)

基于[easy-socket-node](https://github.com/wjkang/easy-socket-node)与[easy-socket-browser](https://github.com/wjkang/easy-socket-browser)一个完整例子：

index.html
```html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Document</title>
</head>

<body>
</body>
<script src="https://unpkg.com/easy-socket-browser@1.1.1/lib/easy-socket.min.js"></script>
<script>
    <!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Document</title>
</head>

<body>
</body>
<script src="https://unpkg.com/easy-socket-browser@1.1.1/lib/easy-socket.min.js"></script>
<script>
    var client = new EasySocket({
        name: 'demo',
        autoReconnect: true,
        pingMsg: '{"type":"event","event":"ping","args":"ping"}'//模拟emit 消息体
    });
    client.openUse((context, next) => {
        console.log("open");
        next();
    })
        .closeUse((context, next) => {
            console.log("close");
            next();
        }).errorUse((context, next) => {
            console.log("error", context.event);
            next();
        }).messageUse((context, next) => {
            if (context.res.type === 'event') {
                context.client.emit(context.res.event, context.res.args, true);
            }
            next();
        })
        .reconnectUse((context, next) => {
            console.log('正在进行重连')
            next();
        })
        .remoteEmitUse((context, next) => {
            let client = context.client;
            let event = context.event;
            if (client.socket.readyState !== 1) {
                console.log("连接已断开");
            } else {
                client.socket.send(JSON.stringify({
                    type: 'event',
                    event: event.event,
                    args: event.args
                }));
                next();
            }
        });


    client.connect('ws://localhost:3001');
    var msg = 1;
    setInterval(() => {
        client.emit('chatMessage', msg++)
    }, 3000);
    client.on("serverMessage", (data) => {
        console.log("serverMessage:" + data)
    });

</script>

</html>

</script>

</html>
```
server.js
```js
var EasySocket = require('easy-socket-node').default;
var config = {
    port: 3001,
    perMessageDeflate: {
        zlibDeflateOptions: { // See zlib defaults.
            chunkSize: 1024,
            memLevel: 7,
            level: 3,
        },
        zlibInflateOptions: {
            chunkSize: 10 * 1024
        },
        // Other options settable:
        clientNoContextTakeover: true, // Defaults to negotiated value.
        serverNoContextTakeover: true, // Defaults to negotiated value.
        //clientMaxWindowBits: 10,       // Defaults to negotiated value.
        serverMaxWindowBits: 10,       // Defaults to negotiated value.
        // Below options specified as default values.
        concurrencyLimit: 10,          // Limits zlib concurrency for perf.
        threshold: 1024,               // Size (in bytes) below which messages
        // should not be compressed.
    }
}

var remoteEmitMiddleware = (context, next) => {
    var server = context.server;
    var event = context.event;
    for (let client of server.clients.values()) {
        client.readyState == 1 && client.send(makeEventMessage(event));
    }
}
function makeEventMessage(event) {
    return JSON.stringify({
        type: 'event',
        event: event.event,
        args: event.args
    })
}
var messageRouteMiddleware = (routes) => {
    return (context, next) => {
        if (context.req.type === 'event') {
            if (routes[context.req.event]) {
                routes[context.req.event](context, next);
            } else {
                context.server.emit(context.req.event, context.req.args);//将会直接触发remoteEmitMiddleware 中间件的调用
                next();
            }
        } else {
            next();
        }
    }
}
var router = {
    chatMessage: (context, next) => {
        var req = context.req;
        context.server.emit('serverMessage', req.args);
    }
}
var server = new EasySocket();
server
    .connectionUse((context, next) => {
        context.server.clients.set(1, context.client)
        console.log('new connection')
    })
    .closeUse((context, next) => {
        console.log('close')
    })
    .messageUse(messageRouteMiddleware(router))
    .remoteEmitUse(remoteEmitMiddleware)
    .listen(config)

console.log('Now start WebSocket server on port ' + config.port + '...')
```
运行过程，可以停止后端服务，然后再启动，测下心跳重连


实现的聊天室例子：[online chat demo](http://jaycewu.coding.me/easy-socket-chat/#/) 

聊天室前端源码：[lazy-mock-im](https://github.com/wjkang/lazy-mock-im)

聊天室服务端源码：[lazy-mock](https://github.com/wjkang/lazy-mock/tree/chat)







