# mongols 简介

mongols是C++ 服务器基础设施库， 它的主要特性如下：

- tcp 服务器
- http 服务器
- websocket 服务器
- web 服务器
- leveldb 服务器
- lua 服务器
- javascript 服务器
- chaiscript 服务器
- sqlite 服务器
- medis 服务器
- proxy 服务器

以上所有服务器均通过epoll机制实现，并且支持多线程化和多进程化：

- 单进程单线程
- 单进程多线程
- 多进程单线程
- 多进程多线程

这些模型统统支持，而且非常易于支持。

![mongols.png](doc/image/mongols.png)

mongols不依赖于任何事件库，其并发性能却远远强于著名的libevent、libev和libuv——这三个库已经过时啦！

mongols支持三种脚本语言lua,js和chai,你的品味决定你的选择。对mongols而言，它们是无差异的。

它还提供非常友好的开发接口，使得任何试图基于tcp、resp或http协议开发高性能网络服务器的开发者都能够轻易地完成工作。

下载：[https://github.com/webcpp/mongols](https://github.com/webcpp/mongols)