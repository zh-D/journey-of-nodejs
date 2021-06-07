TCP与UDP都属于网络传输层协议，如果要构造高效的网络应用，就应该从传输层进行着手。

**但是对于经典的应用场景，则无须从传输层协议入手构造自己的应用，比如 HTTP 或 SMTP 等，这 些经典的应用层协议对于普通应用而言绰绰有余。Node 提供了基本的 http 和 https 模块用于 HTTP 和 HTTPS 的封装，对于其他应用层协议的封装，也能从社区中轻松找到其实现。**

在 Node 中构建 HTTP 服务极其容易，Node 官网上的经典例子就展示了如何用寥寥几行代码实现一个 HTTP 服务器，代码如下：

```javascript
var http = require('http'); 
http.createServer(function (req, res) { 
 res.writeHead(200, {'Content-Type': 'text/plain'}); 
 res.end('Hello World\n'); 
}).listen(1337, '127.0.0.1'); 
console.log('Server running at http://127.0.0.1:1337/'); 

```

### HTTP 是什么

略

### http 模块

Node 的 http 模块包含对 HTTP 处理的封装。**在 Node 中，HTTP 服务继承自 TCP 服务器（net 模块），它能够与多个客户端保持连接，由于其采用事件驱动的形式，并不为每一个连接创建额外的线程或进程，保持很低的内存占用，所以能实现高并发。**HTTP 服务与 TCP 服务模型有区别的地方在于，在开启 keepalive 后，一个 TCP 会话可以用于多次请求和响应。**TCP 服务以 connection 为单位进行服务，HTTP 服务以 request 为单位进行服务。http 模块即是将 connection 到 request 的过程进行了封装**。

除此之外，**http 模块将连接所用套接字的读写抽象为 ServerRequest 和 ServerResponse 对象， 它们分别对应请求和响应操作。**在请求产生的过程中，http 模块拿到连接中传来的数据，调用二 进制模块 http_parser 进行解析，在解析完请求报文的报头后，触发 request 事件，调用用户的业 务逻辑。

### HTTP请求

报文头第一行GET / HTTP/1.1被解析之后分解为如下属性:

- req.method属性：值为GET，是为请求方法，常见的请求方法有GET、POST、DELETE、PUT、 CONNECT等几种。 
- req.url属性：值为/。
- req.httpVersion属性：值为1.1。

其余报头是很规律的 Key: Value 格式，被解析后放置在 req.headers 属性上传递给业务逻辑以供调用，如下所示：

```javascript
headers: 
 { 'user-agent': 'curl/7.24.0 (x86_64-apple-darwin12.0) libcurl/7.24.0 OpenSSL/0.9.8r zlib/1.2.5', 
 host: '127.0.0.1:1337', 
 accept: '*/*' }, 
```

报文体部分则抽象为一个只读流对象，如果业务逻辑需要读取报文体中的数据，则要在这个数据流结束后才能进行操作，如下所示：

```javascript
function (req, res) { 
 // console.log(req.headers); 
 var buffers = []; 
 req.on('data', function (trunk) { 
 buffers.push(trunk); 
 }).on('end', function () { 
 var buffer = Buffer.concat(buffers); 
 // TODO
 res.end('Hello world'); 
 }); 
} 
```

HTTP 请求对象和 HTTP 响应对象是相对较底层的封装，现行的 Web 框架如 Connect 和 Express 都是在这两个对象的基础上进行高层封装完成的。

### HTTP响应

再来看看HTTP响应对象。HTTP响应相对简单一些，它封装了对底层连接的写操作，可以将 其看成一个可写的流对象。它影响响应报文头部信息的 API 为 res.setHeader() 和 res.  writeHead()。在上述示例中：

```javascript
res.writeHead(200, {'Content-Type': 'text/plain'});
```

其分为 setHeader() 和 writeHead() 两个步骤。它在http模块的封装下，实际生成如下报文：

```javascript
< HTTP/1.1 200 OK 
< Content-Type: text/plain 
```

我们可以调用setHeader进行多次设置，但只有调用 writeHead 后，报头才会写入到连接中。 除此之外，http模块会自动帮你设置一些头信息，如下所示：

```javascript
< Date: Sat, 06 Apr 2013 08:01:44 GMT 
< Connection: keep-alive 
< Transfer-Encoding: chunked 
```

报文体部分则是调用res.write()和res.end()方法实现，后者与前者的差别在于res.end()会 先调用write()发送数据，然后发送信号告知服务器这次响应结束，响应结果如下所示：

```javascript
hello world
```

报头是在报文体发送前发送的，一旦开始了数据的发送，writeHead()和setHeader()将不 再生效。这由协议的特性决定。

另外，无论服务器端在**处理业务逻辑时是否发生异常，务必在结束时调用 res.end() 结束请求**，否则客户端将一直处于等待的状态。当然，也可以通过延迟 res.end() 的方式实现客户端与 服务器端之间的长连接，但**结束时务必关闭连接。**

### HTTP服务的事件

如同TCP服务一样，HTTP 服务器也抽象了一些事件，以供应用层使用，同样典型的是，服务器也是一个EventEmitter 实例。

- connection事件：在开始HTTP请求和响应前，客户端与服务器端需要建立底层的TCP连 接，这个连接可能因为开启了keep-alive，可以在多次请求响应之间使用；当这个连接建 立时，服务器触发一次connection事件。
-  request事件：建立TCP连接后，http模块底层将在数据流中抽象出HTTP请求和HTTP响 应，当请求数据发送到服务器端，在解析出HTTP请求头后，将会触发该事件；在res.end() 后，TCP连接可能将用于下一次请求响应。
- close事件：与TCP服务器的行为一致，调用server.close()方法停止接受新的连接，当已 有的连接都断开时，触发该事件；可以给server.close()传递一个回调函数来快速注册该 事件。
-  checkContinue事件：某些客户端在发送较大的数据时，并不会将数据直接发送，而是先 发送一个头部带Expect: 100-continue的请求到服务器，服务器将会触发checkContinue 事件；如果没有为服务器监听这个事件，服务器将会自动响应客户端100 Continue的状态 码，表示接受数据上传；如果不接受数据的较多时，响应客户端400 Bad Request拒绝客 户端继续发送数据即可。需要注意的是，当该事件发生时不会触发request事件，两个事 件之间互斥。当客户端收到100 Continue后重新发起请求时，才会触发request事件。
-  connect事件：当客户端发起CONNECT请求时触发，而发起CONNECT请求通常在HTTP代理时 出现；如果不监听该事件，发起该请求的连接将会关闭。
- upgrade事件：当客户端要求升级连接的协议时，需要和服务器端协商，客户端会在请求 头中带上Upgrade字段，服务器端会在接收到这样的请求时触发该事件。这在后文的 WebSocket部分有详细流程的介绍。如果不监听该事件，发起该请求的连接将会关闭。
-  clientError事件：连接的客户端触发error事件时，这个错误会传递到服务器端，此时触 发该事件。

### HTTP客户端

http 模块提供了一个底层 API：http.request(options, connect)，用于构造 HTTP 客户端。（模拟一个浏览器的 HTTP 请求）

```javascript
var options = { 
 hostname: '127.0.0.1', 
 port: 1334, 
 path: '/', 
 method: 'GET' 
}; 
var req = http.request(options, function(res) { 
 console.log('STATUS: ' + res.statusCode); 
 console.log('HEADERS: ' + JSON.stringify(res.headers)); 
 res.setEncoding('utf8'); 
 res.on('data', function (chunk) { 
 console.log(chunk); 
 }); 
}); 
req.end(); 

```

执行上述代码得到以下输出：

```javascript
$ node client.js 
STATUS: 200 
HEADERS: {"date":"Sat, 06 Apr 2013 11:08:01 
GMT","connection":"keep-alive","transfer-encoding":"chunked"} 
Hello World 
```

其中options参数决定了这个HTTP请求头中的内容，它的选项有如下这些

- host：服务器的域名或IP地址，默认为localhost。 
- hostname：服务器名称。 
- port：服务器端口，默认为80。 
- localAddress：建立网络连接的本地网卡。 
- socketPath：Domain套接字路径。 
- method：HTTP请求方法，默认为GET。 
- path：请求路径，默认为/。
- headers：请求头对象。 
- auth：Basic认证，这个值将被计算成请求头中的Authorization部分。

#### HTTP响应

HTTP客户端的响应对象与服务器端较为类似，在ClientRequest对象中，它的事件叫做 response。ClientRequest在解析响应报文时，一解析完响应头就触发response事件，同时传递一 个响应对象以供操作ClientResponse。后续响应报文体以只读流的方式提供，如下所示：

```javascript
function(res) { 
 console.log('STATUS: ' + res.statusCode); 
 console.log('HEADERS: ' + JSON.stringify(res.headers)); 
 res.setEncoding('utf8'); 
 res.on('data', function (chunk) { 
 console.log(chunk); 
 }); 
} 
```

#### HTTP 代理

。为了重用TCP连接，http模块包含一 个默认的客户端代理对象http.globalAgent。它对每个服务器端（host + port）创建的连接进行了 管理，默认情况下，通过ClientRequest对象对同一个服务器端发起的HTTP请求最多可以创建5 个连接。它的实质是一个连接池.。

调用HTTP客户端同时对一个服务器发起10次HTTP请求时，其实质只有5个请求处于并发状 态，后续的请求需要等待某个请求完成服务后才真正发出。这与浏览器对同一个域名有下载连接数的限制是相同的行为。

如果你在服务器端通过 ClientRequest 调用网络中的其他 HTTP 服务，记得关注代理对象对网络请求的限制。一旦请求量过大，连接限制将会限制服务性能。如需要改变，可以在 options 中传递 agent 选项。默认情况下，请求会采用全局的代理对象，默认连接数限制的为 5。

我们既可以自行构造代理对象，代码如下：

```javascript
var agent = new http.Agent({ 
 maxSockets: 10 
}); 
var options = { 
 hostname: '127.0.0.1', 
 port: 1334, 
 path: '/', 
 method: 'GET', 
 agent: agent 
}; 

```

Agent对象的sockets和requests属性分别表示当前连接池中使用中的连接数和处于等待状态 的请求数，在业务中监视这两个值有助于发现业务状态的繁忙程度。

HTTP客户端事件

- response：与服务器端的request事件对应的客户端在请求发出后得到服务器端响应时， 会触发该事件。 
- socket：当底层连接池中建立的连接分配给当前请求对象时，触发该事件。 
- connect：当客户端向服务器端发起CONNECT请求时，如果服务器端响应了200状态码，客 户端将会触发该事件。 
- upgrade：客户端向服务器端发起Upgrade请求时，如果服务器端响应了101 Switching  Protocols状态，客户端将会触发该事件。 
- continue：客户端向服务器端发起Expect: 100-continue头信息，以试图发送较大数据量， 如果服务器端响应100 Continue状态，客户端将触发该事件。