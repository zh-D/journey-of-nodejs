Node中其实还存在一些与 I/O 无关的异步 API，这一部分也值得略微关注一下，它们分别是 setTimeout()、setInterval()、 setImmediate() 和 process.nextTick()

### 定时器

setTimeout() 和 setInterval() 与浏览器中的 API 是一致的，分别用于单次和多次定时执行任务。它们的实现原理与异步 IO 比较类似，**但它们不需要I/O线程池的参与。**

定时器会被插入到**定时器观察者**内部的一个红黑树中。每次 Tick 执行时，会 从该红黑树中迭代取出定时器对象，检查是否超过定时时间，如果超过，就形成一个事件，它的回调函数将立即执行。

定时器的问题在于，它并非精确的（在容忍范围内）。

###  process.nextTick()

由于事件循环自身的特点，定时器的精确度不够。而事实上，采用定时器需要动用红黑树， 创建定时器对象和迭代等操作，而 setTimeout(fn, 0) 的方式较为浪费性能。实际上， process.nextTick() 方法的操作相对较为轻量，具体代码如下：

```javascript
process.nextTick = function(callback) { 
 // on the way out, don't bother. 
 // it won't get fired anyway 
 if (process._exiting) return; 
 if (tickDepth >= process.maxTickDepth) 
 maxTickWarn(); 
 var tock = { callback: callback }; 
 if (process.domain) tock.domain = process.domain; 
 nextTickQueue.push(tock); 
 if (nextTickQueue.length) { 
 process._needTickCallback(); 
 } 
}; 

```

每次调用 **process.nextTick() 方法，只会将回调函数放入队列**中，在下一轮 Tick 时取出执行。 定时器中采用红黑树的操作时间复杂度为 O(lg(n))，nextTick() 的时间复杂度为 O(1)。相较之下， process.nextTick() 更高效。

###  setImmediate()

setImmediate()方法与process.nextTick()方法十分类似，都是将回调函数延迟执行.但是两者之间其实是有细微差别的。将它们放在一起时，又会是怎样的优先级呢。示例代码如下：

```javascript
process.nextTick(function () { 
 console.log('nextTick延迟执行'); 
}); 
setImmediate(function () { 
 console.log('setImmediate延迟执行'); 
}); 
console.log('正常执行'); 
```

其执行结果如下： 

```
正常执行
nextTick延迟执行
setImmediate延迟执行
```

**process.nextTick() 中的回调函数执行的优先级要高于 setImmediate()。**

**事件循环对观察者的检查是有先后顺序的**，process.nextTick() 属于 idle 观察者， setImmediate() 属于 check 观察者。在每一个轮循环检查中，idle 观察者先于 I/O 观察者，I/O 观察者先于 check 观察者。

- 在具体实现上，process.nextTick()的回调函数保存在一个数组中，setImmediate()的结果则是保存在链表中。
- 在行为上，process.nextTick() 在每轮循环中会将数组中的回调函数全部执行完，而 setImmediate() 在每轮循环中**执行链表中的一个回调函数**。

如：

```javascript
// 加入两个nextTick()的回调函数
process.nextTick(function () { 
 console.log('nextTick延迟执行1'); 
}); 
process.nextTick(function () { 
 console.log('nextTick延迟执行2'); 
}); 
// 加入两个setImmediate()的回调函数
setImmediate(function () { 
 console.log('setImmediate延迟执行1'); 
 // 进入下次循环
 process.nextTick(function () { 
 console.log('强势插入'); 
 }); 
}); 
setImmediate(function () { 
 console.log('setImmediate延迟执行2'); 
}); 
console.log('正常执行'); 
```

其执行结果如下：

```javascript
正常执行
nextTick延迟执行1 
nextTick延迟执行2 
setImmediate延迟执行1 
强势插入
setImmediate延迟执行2 
```

从执行结果上可以看出，**当第一个 setImmediate() 的回调函数执行后，并没有立即执行第二 个，而是进入了下一轮循环，再次按 process.nextTick() 优先、setImmediate() 次后的顺序执行。** 之所以这样设计，是为了保证每轮循环能够较快地执行结束，防止 CPU 占用过多而阻塞后续 I/O 调用的情况。

