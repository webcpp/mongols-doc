# qjs 服务器

mongols-1.7.1以后版本不再支持以duktape引擎实现的javascript服务器。

但是，自mongols-1.8.0开始，新增一个基于quickjs引擎的新的javascript服务器。性能较之以前，有较大提升。

来看例子：
```cpp
#include <mongols/qjs_server.hpp>
#include <mongols/util.hpp>

int main(int, char**)
{
    int port = 9090;
    const char* host = "127.0.0.1";
    mongols::qjs_server
        server(host, port, 5000, 8192, 0 /*2*/);
    server.set_root_path("html/qjs");
    server.set_enable_bootstrap(true);
    server.set_enable_lru_cache(false);
    server.set_lru_cache_expires(1);
    //    if (!server.set_openssl("openssl/localhost.crt", "openssl/localhost.key")) {
    //        return -1;
    //    }

    // server.set_shutdown([&]() {
    //     std::cout << "process " << getpid() << " exit.\n";
    // });
    // server.run();

    std::function<void(pthread_mutex_t*, size_t*)> ff = [&](pthread_mutex_t* mtx, size_t* data) {
        server.set_shutdown([&]() {
            std::cout << "process " << getpid() << " exit.\n";
        });
        server.run();
    };

    std::function<bool(int)> g = [&](int status) {
        return false;
    };

    mongols::multi_process main_process;
    main_process.run(ff, g);
}
```

```js
import * as std from "std";
import * as os from "os";
import * as bjson from "bjson";
import * as mongols from "mongols";

mongols.status(200);
mongols.header('Content-Type', 'text/plain');
mongols.content("hello,world\n");

```

简单的`hello,world`测试如下：
![qjs_server_ab](image/qjs_server_ab.png)
![qjs_server_wrk](image/qjs_server_wrk.png)


## 优化

qjs_server通过两个静态变量调整服务器工作状态。
### `qjs_server::memory_limit`
默认值是1GB。它负责配置quickjs运行时能够使用的最大内存量。因为每一工作进程配有一个运行时，所以配置此值时需要考虑机器的硬件约束。通常默认值即可。
### `qjs_server::ctx_called_limit`
默认值是10240。它负责配置每一quickjs运行时上下文被使用的次数。值过小会导致效率下降，值过大则可能导致内存使用上升。通常默认值即可。配置时，需与`qjs_server::memory_limit`协调。
### 开启lru缓存机制
调用`set_enable_lru_cache`方法，参数为`true`即可。此法甚妙。1秒的过期时间，即可令服务器功力大增。如下图：
![qjs_server_wrk_lru](image/qjs_server_wrk_lru.png)

## api

### request
- uri
- method
- client
- param
- user_agent
- has_header
- get_header
- has_form
- get_form
- has_session
- get_session
- has_cookie
- get_cookie
- has_cache
- get_cache
### response
- status
- content
- header
- session
- cache

## 模块

quickjs原版内置三个模块：std、os和bjson，并且启用了数学扩展。我为其添加了三个内置模块：hash、crypto和mongols。qjs_server建立在mongols模块之上。hash模块提供md5、sha1、sha256和sha512计算，crypto模块提供AES加密服务。

## 路由

qjs_server提供一个默认的路由器。使用方法如下：

```javascript
import * as mongols from "mongols";
import { crypto } from "crypto";
import route from "./lib/route.mjs"

var r = route.get_instance();

r.get('^/$', function (m, param) {
  m.status(200);
  m.header('Content-Type', 'text/plain;charset=utf-8');
  m.content('hello,world\n');
});

r.get('^/(.*)/?$', function (m, param) {
  m.header('Content-Type', 'text/plain;charset=UTF8')
  m.content(m.method() + '\n' + m.uri() + '\n' + param.toString())
  m.status(200)
});

var route_test = function (m) {
  r.run(m)
};


export default route_test;
```
路由器支持各种HTTP方法，具体可参考`example/html/qjs/test/lib/route.mjs`。

如果有更好的路由器实现，欢迎替换。

## 比较

quickjs引擎是一个新的javascript引擎。它具备很多优点和新特性。其综合性能不及V8。此点自然会在一定程度上拖累qjs_server。

不过，qjs_server仍可优于nodejs——以号称最快的nodejs框架fastify为例:

```javascript
const cluster = require('cluster');
const numCPUs = require('os').cpus().length;
// Require the framework and instantiate it
const fastify = require('fastify')({ logger: false })



if (cluster.isMaster) {
  console.log(`Master ${process.pid} is running`);

  // Fork workers.
  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }

} else {

  // Declare a route
  fastify.get('/', async (request, reply) => {
    reply.header('Content-Type', 'text/plain;charset=utf-8')
    return 'hello,world\n';
  })

  // Run the server!
  const start = async () => {
    try {
      await fastify.listen(3000)
      fastify.log.info(`server listening on ${fastify.server.address().port}`)
    } catch (err) {
      fastify.log.error(err)
    }
  }
  start()

}
```
使用nodejs-v13驱动,同样使用默认路由器的情况下，ab压测对比于qjs_server:
![mongolsVSfastify_ab](image/mongolsVSfastify_ab.png)
若以wrk压测，则如下:
![mongolsVSfastify_wrk](image/mongolsVSfastify_wrk.png)
fastify不仅不如qjs_server快，而且不如后者轻。完成上述测试，fastify每一进程至少需要25至30mb内存消耗，而对qjs_server而言，每一进程使用2mb已经绰绰有余。

比较于hi-nginx-qjs，qjs_server亦可占上风(均未使用路由器,均使用几乎完全一致的测试代码)：
![qjs_serverVShi-nginx-qjs](image/qjs_serverVShi-nginx-qjs.png)