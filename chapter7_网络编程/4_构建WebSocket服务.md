提到Node，不能错过的是WebSocket协议。它与Node之间的配合堪称完美，其理由有两条：

- WebSocket 客户端基于事件的编程模型与 Node 中自定义事件相差无几。 
- WebSocket 实现了客户端与服务器端之间的长连接，而 Node 事件驱动的方式十分擅长与大 量的客户端保持高并发连接。

WebSocket 与传统 HTTP 有如下好处：

- 客户端与服务器端只建立一个 TCP 连接，可以使用更少的连接。 
- WebSocket 服务器端可以推送数据到客户端，这远比 HTTP 请求响应模式更灵活、更高效。 
- 有更轻量级的协议头，减少数据传送量。

### WebSocket 握手

### WebSocket 数据传输