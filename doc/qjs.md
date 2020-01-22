# qjs 服务器

mongols-1.7.1以后版本不再支持以duktape引擎实现的javascript服务器。

但是，自mongols-1.8.0开始，新增一个基于quickjs引擎的新的javascript服务器。

来看例子：
```cpp
#include <mongols/qjs_server.hpp>

int main(int, char**)
{
    int port = 9090;
    const char* host = "127.0.0.1";
    mongols::qjs_server
        server(host, port, 5000, 8192, 0 /*2*/);
    server.set_root_path("html/qjs");
    server.set_enable_bootstrap(true);
    server.set_enable_lru_cache(true);
    server.set_lru_cache_expires(1);
    //    if (!server.set_openssl("openssl/localhost.crt", "openssl/localhost.key")) {
    //        return -1;
    //    }

    server.set_shutdown([&]() {
        std::cout << "process " << getpid() << " exit.\n";
    });
    server.run();
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

