---
layout: post
tags: phantom websocket
date: 2018-03-01 10:00
title: phantom无法追踪websocket问题解决
published: true
---

### 1.禁用websocket, 使使用websocket的请求转为使用xhr
```javascript
page.on('onInitialized', function () {
  page.evaluate(function() {
    (function(w) {
        w.WebSocket = undefined;
    })(window);
  })
})
```
<!--more-->

### 2.修改原WebSocket类，添加phantom的回调
```javascript
page.on('onInitialized', function () {
  page.evaluate(function() {
    (function(w){
        var oldWS = w.WebSocket;
        w.WebSocket = function(url, protocols){
            this.ws = new oldWS(url, protocols);
        };
        w.WebSocket.prototype.send = function(msg){
            w.callPhantom({type: "websocket", event: "send", msg: msg});
            this.ws.send(msg);
        };
      })(window);
  })
})

page.on('onCallback', function (data) {
  console.log(data)
})
```
