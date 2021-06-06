Buffer对象可以与字符串之间相互转换。目前支持的字符串编码类型有如下这几种。

- ASCII
- UTF-8
- UTF-16LE/UCS-2
- Base64
- Binary
- Hex

### 字符串转 Buffer 

字符串转Buffer对象主要是通过构造函数完成的：

```javascript
new Buffer(str, [encoding]); 
```

通过构造函数转换的Buffer对象，存储的只能是一种编码类型。**encoding参数不传递时，默 认按UTF-8编码进行转码和存储。**

一个 Buffer 对象可以存储不同编码类型的字符串转码的值，调用 write() 方法可以实现该目 的，代码如下：

```javascript
buf.write(string, [offset], [length], [encoding]) 
```

**由于可以不断写入内容到 Buffer 对象中，并且每次写入可以指定编码，所以 Buffer 对象中可 以存在多种编码转化后的内容。需要小心的是，每种编码所用的字节长度不同，将 Buffer 反转回字符串时需要谨慎处理。**

### Buffer转字符串

实现Buffer向字符串的转换也十分简单，Buffer对象的toString()可以将Buffer对象转换为字 符串，代码如下：

```javascript
buf.toString([encoding], [start], [end]) 
```

比较精巧的是，可以设置 encoding（默认为UTF-8）、start、end 这3个参数实现整体或局部的转换。如果 Buffer 对象由多种编码写入，就需要在局部指定不同的编码，才能转换回正常的编码。

### Buffer不支持的编码类型

目前比较遗憾的是，Node的Buffer对象支持的编码类型有限，只有少数的几种编码类型可以 在字符串和Buffer之间转换。为此，Buffer提供了一个isEncoding()函数来判断编码是否支持转换：

```javascript
Buffer.isEncoding(encoding)
```

将编码类型作为参数传入上面的函数，如果支持转换返回值为 true，否则为 false。很遗憾的是，在中国常用的GBK、GB2312 和 BIG-5 编码都不在支持的行列中。

对于不支持的编码类型，可以借助Node生态圈中的模块完成转换。

iconv 和 iconv-lite 两个模块可以支持更多的编码类型转换，包括 Windows 125 系列、ISO-8859 系列、IBM/DOS代码页系列、Macintosh 系列、KOI8 系列，以及 Latin1、US-ASCII，也支持宽字节编码 GBK 和 GB2312。