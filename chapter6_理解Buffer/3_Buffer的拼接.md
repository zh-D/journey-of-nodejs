Buffer在使用场景中，通常是以一段一段的方式传输。以下是常见的从输入流中读取内容的 示例代码：

```javascript
var fs = require('fs'); 
var rs = fs.createReadStream('test.md'); 
var data = ''; 
rs.on("data", function (chunk){ 
 data += chunk; 
}); 
rs.on("end", function () { 
 console.log(data); 
}); 
```

上面这段代码常见于国外，用于流读取的示范，data 事件中获取的 chunk 对象即是 Buffer 对象。 对于初学者而言，容易将 Buffer 当做字符串来理解，所以在接受上面的示例时不会觉得有任何异 常。

一旦输入流中有宽字节编码时，问题就会暴露出来。如果你在通过Node开发的网站上看到 乱码符号，那么该问题的起源多半来自于这里。

这里潜藏的问题在于如下这句代码：

```javascript
data += chunk; 
```

这句代码里隐藏了toString()操作，它等价于如下的代码：

```javascript
data = data.toString() + chunk.toString();
```

值得注意的是，外国人的语境通常是指英文环境，在他们的场景下，这个 toString() 不会造成任何问题。但对于宽字节的中文，却会形成问题。为了重现这个问题，下面我们模拟近似的场景，将文件可读流的每次读取的 Buffer 长度限制为 11，代码如下：

```javascript
var rs = fs.createReadStream('test.md', {highWaterMark: 11}); 
```

搭配该代码的测试数据为李白的《静夜思》。执行该程序，将会得到以下输出：

```javascript
床前明���光，疑���地上霜；举头���明月，���头思故乡。
```

### 乱码是如何产生的

产 生这个输出结果的原因在于文件可读流在读取时会逐个读取Buffer。

这首诗的原始Buffer应存 储为：

```javascript
<Buffer e5 ba 8a e5 89 8d e6 98 8e e6 9c 88 e5 85 89 ef bc 8c e7 96 91 e6 98 af e5 9c b0 e4 b8 8a e9 
9c 9c ef bc 9b e4 b8 be e5 a4 b4 e6 9c 9b e6 98 8e e6 9c 88 ...>
```

由于我们限定了 Buffer 对象的长度为 11，因此只读流需要读取 7 次才能完成完整的读取，结果是以下几个 Buffer 对象依次输出：

```javascript
<Buffer e5 ba 8a e5 89 8d e6 98 8e e6 9c> 
<Buffer 88 e5 85 89 ef bc 8c e7 96 91 e6> 
... 
```

上文提到的 buf.toString() 方法默认以 UTF-8 为编码，中文字在 UTF-8 下占 3 个字节。所以第 一个Buffer对象在输出时，只能显示 3 个字符，Buffer 中剩下的 2 个字节（e6 9c）将会以乱码的形式显示。第二个Buffer对象的第一个字节也不能形成文字，只能显示乱码。于是形成一些文字无法正常显示的问题。

在这个示例中我们构造了11这个限制，但是对于任意长度的Buffer而言，宽字节字符串都有 可能存在被截断的情况，只不过Buffer的长度越大出现的概率越低而已，但该问题依然不可忽视。

### setEncoding()与string_decoder()

在看过上述的示例后，也许我们忘记了可读流还有一个设置编码的方法 setEncoding() ，示 例如下：

```javascript
readable.setEncoding(encoding) 
```

该方法的作用是让data事件中传递的不再是一个Buffer对象，而是编码后的字符串。为此， 我们继续改进前面诗歌的程序，添加 setEncoding() 的步骤如下：

```javascript
var rs = fs.createReadStream('test.md', { highWaterMark: 11}); 
rs.setEncoding('utf8'); 
```

重新执行程序，得到输出：

```javascript
床前明月光，疑是地上霜；举头望明月，低头思故乡。
```

这是令人开心的输出结果，说明输出不再受 Buffer 大小的影响了。那 Node 是如何实现这个输出结果的呢？

事实上，在调用 setEncoding() 时，可读流对象在内部设置了一个 decoder 对象。每次 data 事件都通过该 decoder 对象进行 Buffer 到字符串的解码，然后传递给调用者。古诗（原文是“是故”，我觉得是古诗？）设置编码后，data 不再收到原始的 Buffer 对象。**但是这依旧无法解释为何设置编码后乱码问题被解决掉了，因为在前述分析中，无论如何转码，总是存在宽字节字符串被截断的问题。**

StringDecoder 在得到编码后，知道宽字节字符串在 UTF-8 编码下是以 3 个字节的方式存储的，所以第一次 write() 时，只输出前9个字节转码形成的字符，“月”字的 前两个字节被保留在 StringDecoder 实例内部。第二次write() 时，会将这2个剩余字节和后续 11 个字节组合在一起，再次用3的整数倍字节进行转码。于是乱码问题通过这种中间形式被解决了。

虽然 string_decoder 模块很奇妙，但是它也并非万能药，**它目前只能处理UTF-8、Base64和 UCS-2/UTF-16LE这3种编码。**所以，通过 setEncoding() 的方式不可否认能解决大部分的乱码问 题，但并不能从根本上解决该问题。

### 正确拼接Buffer 

**淘汰掉 setEncoding() 方法后**，剩下的解决方案只有将多个小 Buffer 对象拼接为一个 Buffer 对 象，然后通过iconv-lite 一类的模块来转码这种方式。+= 的方式显然不行，那么正确的 Buffer 拼接方法应该如下面展示的形式：

```javascript
var chunks = []; 
var size = 0; 
res.on('data', function (chunk) { 
 chunks.push(chunk); 
 size += chunk.length; 
}); 
res.on('end', function () { 
 var buf = Buffer.concat(chunks, size); 
 var str = iconv.decode(buf, 'utf8'); 
 console.log(str); 
});
```

**正确的拼接方式是用一个数组来存储接收到的所有 Buffer 片段并记录下所有片段的总长度， 然后调用Buffer.concat() 方法生成一个合并的Buffer对象。**Buffer.concat() 方法封装了从小 Buffer 对象向大 Buffer 对象的复制过程，实现十分细腻，值得围观学习：

```javascript
Buffer.concat = function(list, length) { 
 if (!Array.isArray(list)) { 
 throw new Error('Usage: Buffer.concat(list, [length])'); 
 } 
 if (list.length === 0) { 
 return new Buffer(0); 
 } else if (list.length === 1) { 
 return list[0]; 
 } 
 if (typeof length !== 'number') { 
 length = 0; 
 for (var i = 0; i < list.length; i++) { 
 var buf = list[i]; 
 length += buf.length; 
 } 
 } 
 var buffer = new Buffer(length); 
 var pos = 0; 
 for (var i = 0; i < list.length; i++) { 
 var buf = list[i]; 
 buf.copy(buffer, pos); 
 pos += buf.length; 
 } 
 return buffer; 
}; 
```

