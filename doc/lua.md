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

### 其他

为了方便，我还在lua引擎中嵌入了一个可用于正则计算的表：`mongols_regex`，内含三个函数:

- full_match
- partial_match
- match

其具体用法可参考以下代码:

```lua

mongols_res:header('Content-Type','text/plain;charset=UTF-8')

if mongols_req:has_form('test') then
    local test=mongols_req:get_form('test')
    if mongols_regex.partial_match('^[0-9]+$',test) then
        mongols_res:content('int type: '..test)
    elseif mongols_regex.partial_match('^[a-zA-Z]+$',test) then
        mongols_res:content('string type: '..test)
    else
        local match=mongols_regex.match('(\\w+)[_:+-](\\w+)',test)
        if #match > 0 then
            local content=''
            for key,value in ipairs(match) do 
                content =content .. 'part '..key ..' = '.. value..'\n'
            end
            mongols_res:content('match: \n'..content)
        else
            mongols_res:content('not match: '.. test)
        end
    end
else 
    mongols_res:content('not found test variable')
end


mongols_res:status(200)




```