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
        server(host, port, 5000, 8192, 0);
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



## 优化

qjs_server通过两个静态变量调整服务器工作状态。
### `qjs_server::memory_limit`
默认值是1GB。它负责配置quickjs运行时能够使用的最大内存量。因为每一工作进程配有一个运行时，所以配置此值时需要考虑机器的硬件约束。通常默认值即可。
### `qjs_server::ctx_called_limit`
默认值是10240。它负责配置每一quickjs运行时上下文被使用的次数。值过小会导致效率下降，值过大则可能导致内存使用上升。通常默认值即可。配置时，需与`qjs_server::memory_limit`协调。
### 开启lru缓存机制
调用`set_enable_lru_cache`方法，参数为`true`即可。此法甚妙。1秒的过期时间，即可令服务器功力大增。
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

## 压力测试比较
同样开4个工作进程，比较于nodejs(v13)下的fastify框架，qjs_server能够在每工作进程内存消耗仅仅十分之一的优势下，获得近两倍吞吐率：

```js
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
```txt
apib -c30000 -t4 -d30 http://localhost:3000/

(5 / 30) 5311.890 93% cpu
(10 / 30) 35488.653 100% cpu
(15 / 30) 39301.124 99% cpu
(20 / 30) 38975.863 100% cpu
(25 / 30) 40288.632 99% cpu
(30 / 30) 39911.255 100% cpu
Duration:             30.024 seconds
Attempted requests:   997245
Successful requests:  997245
Non-200 results:      0
Connections opened:   29996
Socket errors:        0

Throughput:           33214.740 requests/second
Average latency:      596.694 milliseconds
Minimum latency:      281.847 milliseconds
Maximum latency:      27382.753 milliseconds
Latency std. dev:     488.977 milliseconds
50% latency:          526.840 milliseconds
90% latency:          730.986 milliseconds
98% latency:          1915.567 milliseconds
99% latency:          3387.646 milliseconds

Client CPU average:    99%
Client CPU max:        100%
Client memory usage:    83%

Total bytes sent:      60.74 megabytes
Total bytes received:  144.56 megabytes
Send bandwidth:        16.18 megabits / second
Receive bandwidth:     38.52 megabits / second


```

```txt
apib -c30000 -t4 -d30 http://localhost:9090/

(5 / 30) 45417.124 95% cpu
(10 / 30) 63895.888 100% cpu
(15 / 30) 64130.772 100% cpu
(20 / 30) 64421.669 100% cpu
(25 / 30) 58932.007 100% cpu
(30 / 30) 64635.808 100% cpu
Duration:             30.024 seconds
Attempted requests:   1808772
Successful requests:  1808772
Non-200 results:      0
Connections opened:   29998
Socket errors:        0

Throughput:           60243.352 requests/second
Average latency:      483.053 milliseconds
Minimum latency:      150.099 milliseconds
Maximum latency:      2049.196 milliseconds
Latency std. dev:     131.720 milliseconds
50% latency:          469.488 milliseconds
90% latency:          574.891 milliseconds
98% latency:          804.549 milliseconds
99% latency:          1042.740 milliseconds

Client CPU average:    99%
Client CPU max:        100%
Client memory usage:    74%

Total bytes sent:      108.72 megabytes
Total bytes received:  238.05 megabytes
Send bandwidth:        28.97 megabits / second
Receive bandwidth:     63.43 megabits / second

```

所以不要轻易说quickjs不够快！