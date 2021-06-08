前文介绍了 child_process 模块中的大多数细节，以及如何通过这个模块构建强大的单机集群。如果熟知 Node，也许你会惊讶为何迟迟不谈 cluster 模块。上述提及的问题，Node在v0.8版 本时新增的 cluster 模块就能解决。

在 v0.8 版本之前，实现多进程架构必须通过 child_process 来实现，要创建单机 Node 集群，由于有这么多细节需要处理，对普通工程师而言是一件相对较难的工作，于是 **v0.8 时直接引入了 cluster 模块，用以解决多核 CPU 的利用率问题，同时也提供了较完善的 API，用以处理进程的健壮性问题。**

对于本章开头提到的创建 Node 进程集群，cluster 实现起来也是很轻松的事情，如下所示：

```javascript
// cluster.js 
var cluster = require('cluster'); 
cluster.setupMaster({ 
 exec: "worker.js" 
}); 
var cpus = require('os').cpus(); 
for (var i = 0; i < cpus.length; i++) { 
 cluster.fork(); 
} 
```

执行 node cluster.js 将会得到与前文创建子进程集群的效果相同。就官方的文档而言，它更喜欢如下的形式作为示例：

```javascript
var cluster = require('cluster'); 
var http = require('http'); 
var numCPUs = require('os').cpus().length; 
if (cluster.isMaster) { 
 // Fork workers 
 for (var i = 0; i < numCPUs; i++) { 
 cluster.fork(); 
 } 
 cluster.on('exit', function(worker, code, signal) { 
 console.log('worker ' + worker.process.pid + ' died'); 
 }); 
} else { 
 // Workers can share any TCP connection 
 // In this case its a HTTP server 
 http.createServer(function(req, res) { 
 res.writeHead(200); 
 res.end("hello world\n"); 
 }).listen(8000); 
} 
```

在进程中判断是主进程还是工作进程，主要取决于环境变量中是否有 NODE_UNIQUE_ID，如下 所示：

```javascript
cluster.isWorker = ('NODE_UNIQUE_ID' in process.env); 
cluster.isMaster = (cluster.isWorker === false); 
```

但是官方示例中忽而判断 cluster.isMaster、忽而判断 cluster.isWorker，对于代码的可读 性十分差。我建议用cluster.setupMaster() 这个API，将主进程和工作进程从代码上完全剥离， 如同 send() 方法看起来直接将服务器从主进程发送到子进程那样神奇，剥离代码之后，甚至都感觉不到主进程中有任何服务器相关的代码。

通过 cluster.setupMaster() 创建子进程而不是使用 cluster.fork()，程序结构不再凌乱，逻辑分明，代码的可读性和可维护性较好。

### Cluster 工作原理

事实上cluster模块就是child_process和net模块的组合应用。

cluster 启动时，如同我们在 9.2.3 节里的代码一样，它会在内部启动 TCP 服务器，在 cluster.fork() 子进程时，将这个 TCP 服 务器端 socket 的文件描述符发送给工作进程。如果进程是通过 cluster.fork() 复制出来的，那么它的环境变量里就存在 NODE_UNIQUE_ID，如果工作进程中存在 listen() 侦听网络端口的调用，它将拿到该文件描述符，通过 SO_REUSEADDR 端口重用，从而实现多个子进程共享端口。

对于普通方式启动的进程，则不存在文件描述符传递共享等事情。

在 cluster 内部隐式创建 TCP 服务器的方式对使用者来说十分透明，但也正是这种方式使得它无法如直接使用 child_process 那样灵活。在 cluster 模块应用中，一个主进程只能管理一组工作进程。

对于自行通过 child_process 来操作时，则可以更灵活地控制工作进程，甚至控制多组工作进程。其原因在于自行通过 child_process 操作子进程时，可以隐式地创建多个 TCP 服务器，使得子进程可以共享多个的服务器端 socket

### Cluster 事件

对于健壮性处理，cluster模块也暴露了相当多的事件。

- fork：复制一个工作进程后触发该事件。 
- online：复制好一个工作进程后，工作进程主动发送一条online消息给主进程，主进程收 到消息后，触发该事件。 
- listening：工作进程中调用listen()（共享了服务器端Socket）后，发送一条listening 消息给主进程，主进程收到消息后，触发该事件。 
- disconnect：主进程和工作进程之间IPC通道断开后会触发该事件。 
- exit：有工作进程退出时触发该事件。 
- setup：cluster.setupMaster()执行后触发该事件。

这些事件大多跟 child_process 模块的事件相关，在进程间消息传递的基础上完成的封装。 这些事件对于增强应用的健壮性已经足够了。

### 总结

尽管 Node 从单线程的角度来讲它有够脆弱的：既不能充分利用多核 CPU 资源，稳定性也无法得到保障。但是群体的力量是强大的，通过简单的主从模式，就可以将应用的质量提升一个档次。在实际的复杂业务中，我们可能要启动很多子进程来处理任务，结构甚至远比主从模式复杂，但是每个子进程应当是简单到只做好一件事，然后通过进程间通信技术将它们连接起来即可。这符合 Unix 的设计理念，每个进程只做一件事，并做好一件事，将复杂分解为简单，将简单组合成强大。

尽管通过 child_process 模块可以大幅提升 Node 的稳定性，但是一旦主进程出现问题，所有子进程将会失去管理。在 Node 的进程管理之外，还需要用监听进程数量或监听日志的方式确保整个系统的稳定性，即使主进程出错退出，也能及时得到监控警报，使得开发者可以及时处理故障。