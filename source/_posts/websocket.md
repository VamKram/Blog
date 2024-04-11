title: Html5 WebSocket
date: 2019/11/21
categories:
- html
tags:
- webSocket

---
# Html5 WebSocket

一，WebSocket创建过程

- 创建监听事件
- 断开连接
- 消息时间
- 发送消息返回服务器
- 关闭连接

```javascript
// 创建一个Socket实例
var socket = new WebSocket('ws://localhost:8080'); 
// 打开Socket 
socket.onopen = function(event) { 
  // 发送一个初始化消息
  socket.send('I am the client and I\'m listening!'); 
  // 监听消息
  socket.onmessage = function(event) { 
    console.log('Client received a message',event); 
  }; 
  // 监听Socket的关闭
  socket.onclose = function(event) { 
    console.log('Client notified socket has closed',event); 
  }; 
  // 关闭Socket.... 
  //socket.close() 
};
```

## NodeJS和Socket.IO联合开发

Socket.IO提供的服务器端解决方案，允许统一的客户端和服务器端的API。使用Node，你可以**创建一个典型的HTTP服务器**，然后**把服务器的实例传递到Socket.IO**。从这里，你创建连接、断开连接、建立消息监听器，跟在客户端一样。

```javascript
// 需要HTTP 模块来启动服务器和Socket.IO
var http= require('http'), io= require('socket.io'); 
// 在8080端口启动服务器
var server= http.createServer(function(req, res){ 
  // 发送HTML的headers和message
  res.writeHead(200,{ 'Content-Type': 'text/html' }); 
  res.end('<h1>Hello Socket Lover!</h1>'); 
}); 
server.listen(8080); 
// 创建一个Socket.IO实例，把它传递给服务器
var socket= io.listen(server); 
// 添加一个连接监听器
socket.on('connection', function(client){ 
  // 成功！现在开始监听接收到的消息
  client.on('message',function(event){ 
    console.log('Received message from client!',event); 
  }); 
  client.on('disconnect',function(){ 
    clearInterval(interval); 
    console.log('Server has disconnected'); 
  }); 
});
```

你可以运行服务器部分，假定已安装了NodeJS，从命令行执行：

> node socket-server.js

现在客户端和服务器都能来回推送消息了！在NodeJS脚本内，可以使用简单的JavaScript创建一个定期消息发送器：

```javascript
// 创建一个定期（每5秒）发送消息到客户端的发送器
var interval= setInterval(function() { 
  client.send('This is a message from the server! ' + new Date().getTime()); 
},5000);
```

------

------

服务器端将会每5秒推送消息到客户端！

## 指南介绍

**WebSocket内容**：全双工，一旦建立WebSocket连接，客户端和服务端可以在任何时候互相传送消息（不是采用响应请求的方式），每个基于WebSockt的服务都必须定义自己的子协议用于传输数据（兼容性差多版本），WebSocket包含一种协商机制（用于挑选）可以传一个字符串数组给WebSocket构造函数服务器端会生成一个子协议列表传递给客户端，客户端通过protocol检查是哪种协议

1. 首先创建一个socket

```
 var socket = new WebSocket("ws:/ws.example.com:1234/resource")
```

1. 创建套接字通常需要注册一个事件处理程序

```javascript
socket.onopen = function(e){/* connection*/ };
socket.onclose = function(e){/*close */};
socket.onerror = function(e){ /* error*/}
socket.onmessage = function(e){
        var mes = e.data//向服务器发送一条消息
}
```

1. 调用send方法

```
 scocket.send("Hello");//send();
```

1. close 关闭