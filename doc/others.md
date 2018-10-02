# 杂项

mongols主要是网络库，其他一切设施均直接或间接为此而存在。

使用是`pkg-config --libs --cflags mongols` 即可。

## 线程池

多线程化核心依赖。

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
