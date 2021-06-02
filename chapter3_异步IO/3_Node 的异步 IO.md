### 事件循环

在**进程启动**时，Node 便会创建一个类似于while(true)的循环，每执行一次循环体的过程我们称为Tick。

**每个 Tick 的过程就是查看是否有事件待处理，如果有，就取出事件及其相关的回调 函数。如果存在关联的回调函数，就执行它们。**

如果不再有事件处理，就**退出进程**。

### 观察者

在每个Tick的过程中，如何判断是否有事件需要处理呢？这里必须要引入的概念是观察者。

**每个事件循环中有一个或者多个观察者，而判断是否有事件要处理的过程就是向这些观察者询问是否有要处理的事件。**

**在 Node 中，事件主要来源于网络请求、文件 I/O 等**，这些事件对应的观察者有**文件 I/O 观察者、网络 I/O 观察者**等。观察者将事件进行了分类。

在Windows下，事件循环基于 IOCP 创建，而在 *nix 下则基于多线程创建。

### 请求对象

在这一节中，我们将通过解释Windows下异步I/O（利用IOCP实现）的简单例子来探寻从 JavaScript代码到系统内核之间都发生了什么。

一般的、非异步的回调函数，由我们自行调用：

```javascript
var forEach = function (list, callback) { 
 for (var i = 0; i < list.length; i++) { 
 callback(list[i], i, list); 
 } 
}; 
```

对于 Node 中的异步 I/O 调用而言，回调函数却不由开发者来调用。**那么从我们发出调用后， 到回调函数被执行，中间发生了什么呢？**

事实上，**从 JavaScript 发起调用到内核执行完 I/O 操作的 过渡过程中，存在一种中间产物，它叫做请求对象。**

**下面我们以最简单的 fs.open() 方法来作为例子，探索Node与底层之间是如何执行异步 I/O 调用以及回调函数究竟是如何被调用执行的。**

```javascript
fs.open = function(path, flags, mode, callback) { 
 // ... 
 binding.open(pathModule._makeLong(path), 
 stringToFlags(flags), 
 mode, 
 callback); 
}; 
```

fs.open() 的作用是根据指定路径和参数去打开一个文件，从而得到一个文件描述符，这是 后续所有 I/O 操作的初始操作。

libuv 作为封装层，有两个平台的实现，实质上是调 用了 uv_fs_open() 方法。在 uv_fs_open() 的调用过程中，我们创建了一个 FSReqWrap 请求对象。**从 JavaScript层传入的参数和当前方法都被封装在这个请求对象中**，其中我们最为关注的**回调函数则被设置在这个对象的oncomplete_sym属性**上：

```javascript
req_wrap->object_->Set(oncomplete_sym, callback);
```

对象包装完毕后，在 Windows 下，则调用 QueueUserWorkItem() 方法将这个 FSReqWrap 对象推入线程池中等待执行，该方法的讲解略。

至此，JavaScript 调用立即返回，由 JavaScript 层面发起的异步调用的第一阶段就此结束。 JavaScript 线程可以继续执行当前任务的后续操作。**当前的 I/O 操作在线程池中等待执行，不管它是否阻塞 I/O，都不会影响到 JavaScript 线程的后续执行，如此就达到了异步的目的。**

**请求对象是异步 I/O 过程中的重要中间产物，所有的状态都保存在这个对象中，包括送入线程池等待执行以及I/O操作完毕后的回调处理。**

### 执行回调

组装好请求对象、送入 I/O 线程池等待执行，实际上完成了异步 I/O 的第一部分，回调通知是第二部分。

线程池中的 I/O 操作调用完毕之后，会将获取的结果储存在 req->result 属性上，然后调用   PostQueuedCompletionStatus() 通知 IOCP，告知当前对象操作已经完成。

PostQueuedCompletionStatus() 方法的作用是向 IOCP 提交执行状态，**并将线程归还线程池**。通过PostQueuedCompletionStatus() 方法**提交的状态**，**可以**通过 GetQueuedCompletionStatus() **提取**。

在这个过程中，我们其实还动用了**事件循环的 I/O 观察者**。**在每次 Tick 的执行中，它会调用 IOCP 相关的 GetQueuedCompletionStatus() 方法检查线程池中是否有执行完的请求，如果存在，会将请求对象加入到 I/O 观察者的队列中，然后将其当做事件处理。**

I/O 观察者回调函数的行为就是取出请求对象的 result 属性作为参数，取出 oncomplete_sym 属 性作为方法，然后调用执行，以此**达到调用 JavaScript 中传入的回调函数的目的**。 至此，整个异步I/O的流程完全结束。

**事件循环、观察者、请求对象、I/O 线程池这四者共同构成了 Node 异步 I/O 模型的基本要素。**

异步I/O的几个关键词：单线程、事件循环、观察者 和 I/O线程池。

这里单线程与I/O线程池之间看起来有些悖论的样子。由于我们知道 JavaScript是单线程的，所以按常识很容易理解为它不能充分利用多核CPU。

**事实上，在 Node 中， 除了 JavaScript 是单线程外，Node 自身其实是多线程的，只是 I/O 线程使用的 CPU 较少。另一个需要重视的观点则是，除了用户代码无法并行执行外，所有的 I/O（磁盘I/O和网络I/O等）则是可以并行起来的。**

