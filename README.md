# mongols 简介

mongols是一个高性能C++网络库，提供基于TCP/UDP/HTTP(S)/WebSocket(S)/RESP等协议为基础的多种服务商基础设施。

mongols不依赖于任何事件库，其并发性能却远远强于著名的libevent、libev和libuv。

mongols不仅内嵌支持两种脚本语言lua和quickjs,还提供一个python绑定：[pymongols](https://github.com/webcpp/pymongols)。

mongols提供的接口非常友好，使得任何试图基于tcp、resp、http和websocket协议开发高性能网络服务器的开发者都能够轻易地完成工作。

而且，它提供的所有服务器均能像nginx一样实现master-worker多进程模型，而且比nginx更高效。


下载：[https://github.com/webcpp/mongols](https://github.com/webcpp/mongols)
