# mongols 简介

mongols是C++ 服务器基础设施库， 它的主要特性如下：

- tcp 服务器
- http 服务器
- websocket 服务器
- web 服务器
- leveldb 服务器
- lua 服务器
- sqlite 服务器

以上所有服务器均通过epoll机制实现，并且支持多线程化。

mongols不依赖于任何事件库，其并发性能却强于著名的libevent、libev和libuv。

而且，它提供非常友好的开发接口，使得任何试图基于tcp或者http协议开发高性能网络服务器的开发者都能够轻易地完成工作。