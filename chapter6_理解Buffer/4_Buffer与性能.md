**Buffer 在文件 I/O 和网络 I/O 中运用广泛**，尤其在网络传输中，它的性能举足轻重。在应用中，我们通常会操作字符串，但一旦在网络中传输，都需要转换为 Buffer，以进行二进制数据传输。 在 Web 应用中，字符串转换到Buffer 是时时刻刻发生的，提高字符串到 Buffer 的转换效率，可以很大程度地提高网络吞吐率。

在展开Buffer与网络传输的关系之前，我们可以先来进行一次性能测试。下面的例子中构造 了一个10 KB大小的字符串。我们首先通过纯字符串的方式向客户端发送，代码如下：

```javascript
var http = require('http'); 
var helloworld = ""; 
for (var i = 0; i < 1024 * 10; i++) { 
 helloworld += "a"; 
} 
// helloworld = new Buffer(helloworld); 
http.createServer(function (req, res) { 
 res.writeHead(200); 
 res.end(helloworld); 
}).listen(8001); 

```

如果把里面的注释去掉，性能可以提高一倍。测量方法是 ab：

```javascript
ab -c 200 -t 100 http://127.0.0.1:8001/ 
```

通过预先转换静态内容为 Buffer 对象，可以有效地减少 CPU 的重复使用，节省服务器资源。 在 Node 构建的 Web 应用中，可以选择将页面中的动态内容和静态内容分离，静态内容部分可以通 过预先转换为 Buffer 的方式，使性能得到提升。由于文件自身是二进制数据，所以在不需要改变内容的场景下，尽量只读取 Buffer，然后直接传输，不做额外的转换，避免损耗。

#### 文件读取

Buffer 的使用除了与字符串的转换有性能损耗外，在文件的读取时，有一个 highWaterMark 设置对性能的影响至关重要。在 fs.createReadStream(path, opts) 时，我们可以传入一些参数，代码如下：

```javascript
{ 
 flags: 'r', 
 encoding: null, 
 fd: null, 
 mode: 0666, 
 highWaterMark: 64 * 1024 
} 
```

我们还可以传递start和end来指定读取文件的位置范围：

```javascript
{start: 90, end: 99} 
```

fs.createReadStream() 的工作方式是在内存中准备一段 Buffer，然后在 fs.read() 读取时逐步从磁盘中将字节复制到 Buffer 中。完成一次读取时，则从这个 Buffer 中通过 slice() 方法取出部分数据作为一个小 Buffer 对象，再通过 data 事件传递给调用方。如果 Buffer 用完，则重新分配一个；如果还有剩余，则继续使用。下面为分配一个新的 Buffer 对象的操作：

```javascript
var pool; 
function allocNewPool(poolSize) { 
 pool = new Buffer(poolSize); 
 pool.used = 0; 
} 
```

在理想的状况下，每次读取的长度就是用户指定的 highWaterMark。但是有可能读到了文件结尾，或者文件本身就没有指定的 highWaterMark 那么大，这个预先指定的 Buffer 对象将会有部分剩余，不过好在这里的内存可以分配给下次读取时使用。pool 是常驻内存的，只有当 pool 单元剩余数量小于 128（kMinPoolSpace）字节时，才会重新分配一个新的 Buffer 对象。Node 源代码中分配新的 Buffer 对象的判断条件如下所示：

```javascript
if (!pool || pool.length - pool.used < kMinPoolSpace) { 
 // discard the old pool 
 pool = null; 
 allocNewPool(this._readableState.highWaterMark); 
} 
```

**这里与Buffer的内存分配比较类似，highWaterMark的大小对性能有两个影响的点。**

- **highWaterMark 设置对 Buffer 内存的分配和使用有一定影响。**
- **highWaterMark 设置过小，可能导致系统调用次数过多。**

**文件流读取基于 Buffer 分配，Buffer 则基于 SlowBuffer 分配，这可以理解为两个维度的分配策略。如果文件较小（小于8 KB），有可能造成 slab 未能完全使用。**

由于 fs.createReadStream() 内部采用 fs.read() 实现，将会引起对磁盘的系统调用，对于大文件而言，highWaterMark 的大小决定会触发系统调用和 data 事件的次数。 以下为 Node 自带的基准测试，在benchmark/fs/read-stream-throughput.js 中可以找到：

```javascript
function runTest() { 
 assert(fs.statSync(filename).size === filesize); 
 var rs = fs.createReadStream(filename, { 
 highWaterMark: size, 
 encoding: encoding 
 }); 
 rs.on('open', function() { 
 bench.start(); 
 }); 
 var bytes = 0; 
 rs.on('data', function(chunk) { 
 bytes += chunk.length; 
 }); 
 rs.on('end', function() { 
 try { fs.unlinkSync(filename); } catch (e) {} 
 // MB/sec 
 bench.end(bytes / (1024 * 1024)); 
 }); 
} 
```

下面为某次执行的结果：

```javascript
fs/read-stream-throughput.js type=buf size=1024: 46.284 
fs/read-stream-throughput.js type=buf size=4096: 139.62 
fs/read-stream-throughput.js type=buf size=65535: 681.88 
fs/read-stream-throughput.js type=buf size=1048576: 857.98 
```

**从上面的执行结果我们可以看到，读取一个相同的大文件时，highWaterMark 值的大小与读取速度的关系：该值越大，读取速度越快。**