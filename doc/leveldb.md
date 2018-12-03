# leveldb 服务器

它是专门的缓存服务器。关键是它使用文件系统存储缓存数据，不仅速度非常快而且不用大量消耗内存。


来看代码：

```cpp
#include <mongols/leveldb_server.hpp>

int main(int, char**) {
    int port = 9090;
    const char* host = "127.0.0.1";
    mongols::leveldb_server
    server(host, port, 5000, 8096, 0/*2*/);
    server.run("html/leveldb");
}

````

## 用法
leveldb_server使用http协议处理请求。成功返回200，失败返回500:
- POST `curl -d'key=value' http://host/key`
- GET `curl http://host/key`
- DELETE `curl -X DELETE http://host/key`

