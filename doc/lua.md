# lua 服务器

这是为方便使用脚本语言LUA进行web应用开发准备的。


来看代码：

```cpp
#include <mongols/lua_server.hpp>

int main(int,char**){
	int port = 9090;
	const char* host="127.0.0.1";
	mongols::lua_server 
	//server(host,port,5000,8096,2);
	server(host,port);
	server.set_root_path("html/lua");
	server.run("html/lua/package/?.lua;","html/lua/package/?.so;");
}

```

```lua
local echo =  {}

function echo.concat(a,b)
   return (a..b)
end

return echo	

````

```lua

local echo=require('echo')
mongols_res:header('Content-Type','text/plain;charset=UTF-8')
mongols_res:content(echo.concat('hello,','world'))
mongols_res:status(200)

```

## API

### mongols_req
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
### mongols_res
- status
- content
- header
- session
- cache
