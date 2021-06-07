适合在分布式网络中扮演各种各样的角色。

在 Web 领域，大多数的编程语言需要专门的 Web 服务器作为容器，如 ASP、ASP.NET 需要 IIS 作为服务器， PHP 需要搭载 Apache 或 Nginx 环境等， JSP 需要 Tomcat 服务器等。但对于Node 而言，只需要几行代码即可构建服务器，无需额外的容器。

**Node提供了 net、dgram、http、https这4个模块，分别用于处理 TCP、UDP、HTTP、HTTPS， 适用于服务器端和客户端。**

### TCP 是什么

略

### 创建 TCP 服务器端

创建一个 TCP 服务器端来接受网络请求，代码如下：

```javascript
var net = require('net'); 
var server = net.createServer(function (socket) { 
 // 新的连接
 socket.on('data', function (data) { 
 socket.write("你好"); 
 }); 
 socket.on('end', function () { 
 console.log('连接断开'); 
 }); 
 socket.write("欢迎光临《深入浅出Node.js》示例：\n"); 
}); 
server.listen(8124, function () { 
 console.log('server bound'); 
});
```

我们通过 net.createServer(listener) 即可创建一个TCP服务器，listener 是连接事件 connection 的侦听器，也可以采用如下的方式进行侦听：

```javascript
var server = net.createServer(); 
server.on('connection', function (socket) { 
 // 新的连接
}); 
server.listen(8124);
```

**利用各种工具作为客户端对刚才创建的简单服务器进行会话交流：**

略

### TCP服务的事件

代码分为服务器事件和连接事件。

#### 服务器事件

net.createServer() 创建的服务器而言，它是一个 EventEmitter 实例，它的自定义 事件有如下几种

-  listening：在调用 server.listen() 绑定端口或者 Domain Socket 后触发，简洁写法为 server.listen(port,listeningListener)，通过 listen() 方法的第二个参数传入。
- connection：每个客户端套接字连接到服务器端时触发，简洁写法为通过 net.createServer()，最后一个参数传递。
- close：当服务器关闭时触发，在调用 server.close() 后，服务器将停止接受新的套接字连接，但保持当前存在的连接，等待所有连接都断开后，会触发该事件。
- error：当服务器发生异常时，将会触发该事件。比如侦听一个使用中的端口，将会触发 一个异常，如果不侦听 error 事件，服务器将会抛出异常。

#### 连接事件

**服务器可以同时与多个客户端保持连接，对于每个连接而言是典型的可写可读Stream对象。** Stream对象可以用于服务器端和客户端之间的通信，既可以通过data事件从一端读取另一端发来 的数据，也可以通过write()方法从一端向另一端发送数据。它具有如下自定义事件。

- data：当一端调用write()发送数据时，另一端会触发data事件，事件传递的数据即是 write()发送的数据。
- end：当连接中的任意一端发送了FIN数据时，将会触发该事件。
- connect：该事件用于客户端，当套接字与服务器端连接成功时会被触发。
- drain：当任意一端调用write()发送数据时，当前这端会触发该事件。
- error：当异常发生时，触发该事件。
- close：当套接字完全关闭时，触发该事件。
- timeout：当一定时间后连接不再活跃时，该事件将会被触发，通知用户当前该连接已经被闲置了。

由于TCP套接字是可写可读的Stream对象，可以利用pipe()方法巧妙地实现管道操作， 如下代码实现了一个 echo 服务器：

```javascript
var net = require('net'); 
var server = net.createServer(function (socket) { 
 socket.write('Echo server\r\n'); 
 socket.pipe(socket); 
}); 
server.listen(1337, '127.0.0.1'); 
```

值得注意的是，TCP针对网络中的小数据包有一定的优化策略：Nagle算法。如果每次只发送一个字节的内容而不优化，网络中将充满只有极少数有效数据的数据包，将十分浪费网络资源。 Nagle算法针对这种情况，要求缓冲区的数据达到一定数量或者一定时间后才将其发出，所以小 数据包将会被Nagle算法合并，以此来优化网络。这种优化虽然使网络带宽被有效地使用，但是 数据有可能被延迟发送。

在Node中，由于TCP默认启用了Nagle算法，可以调用socket.setNoDelay(true)去掉Nagle算 法，使得write()可以立即发送数据到网络中。

另一个需要注意的是，尽管在网络的一端调用write()会触发另一端的data事件，但是并不 意味着每次write()都会触发一次data事件，在关闭掉Nagle算法后，另一端可能会将接收到的多 个小数据包合并，然后只触发一次data事件。