# 杂项

mongols主要是网络库，其他一切设施均直接或间接为此而存在。

使用是`pkg-config --libs --cflags mongols openssl` 即可。

## 线程池

多线程化核心依赖。

不要迷信多线程；对真正高性能的服务器来说，使用多线程的结果通常就是将并发性能打8折——线程越多，打折越多；
多线程的合适场景是后台多任务而非提高并发吞吐率。

这也是为什么libevent、libev、nodejs、nginx、redis等项目只使用单线程的真正原因。如果多线程在提供并发吞吐率方面是有效的，那么你很难解释以上项目的作者为什么不使用多线程。

实际上，mongols所提供的所有服务器在单线程情况下都会有更好的性能表现。因此，mongols在version-1.3.4之后,对多线程机制的使用做了调整，仅当需要将消息在各个连接之间转发时，才把相关任务作为后台任务由线程池执行。

## 线程安全队列

它是线程池的一个依赖。

## RE2

正则表达式处理。

## zlib

解、压缩处理。

## jsoncpp 

已经用到ws_server中。还有个json11，看喜欢哪个吧，都能用。我更喜欢jsoncpp。

## fmt

字符串格式化。

## msgpack

序列化。

## http-parser

http 协议处理

## simple_resp

RESP协议处理

## util

满足一些零碎而必要的需求。

## 性能优化建议

开启缓存：`set_enable_cache`或`set_enable_lru_cache`。这个不是一般有效，是非常有效。

## 关于http 服务器和web 服务器的压力测试

在使用ab或者wrk等压力测试软件时，请务必严格区分短连接和长连接。mongols默认短连接。对长连接仅仅通过HEAD:`Connection: keep-alive`识别,任何字母或者字母大小写方面的差异都将不能建立长连接而自动改为短连接。

因此，对于ab,好的例子是这样的：

- 短连接： `ab -c2000 -n50000  http://localhost:9090/nginx.html`
- 长连接： `ab -kc2000 -n50000 -H'Connection: keep-alive' http://localhost:9090/nginx.html`

对于wrk，则是如下:

- 短连接:  `wrk -t4 -d30s -c1000  http://localhost:9090/nginx.html`
- 长连接: `wrk -t4 -d30s -c1000 -H'Connection: keep-alive' http://localhost:9090/nginx.html`

等你需要测试长连接性能时，务必添加HEAD：`Connection: keep-alive`,否则你测得的就是短连接性能。