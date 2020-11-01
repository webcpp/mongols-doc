# lua 服务器

这是为方便使用脚本语言LUA进行web应用开发准备的。


来看代码：

```cpp

#include <mongols/lua_server.hpp>

int main(int, char**) {
    int port = 9090;
    const char* host = "127.0.0.1";
    mongols::lua_server
    server(host, port, 5000, 8096, 0);
    server.set_root_path("html/lua");
    server.set_enable_bootstrap(true);
    server.run("html/lua/package/?.lua;", "html/lua/package/?.so;");
}

```

```lua

local echo =  {}

function echo.concat(...)
    local text=''
    for i,v in ipairs({...}) do
        text=text..tostring(v)
    end
   return text
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

为了方便json处理，我内置了一个基于json.hpp的类mongols_json，API列表如下：
- set
- get
- append
- parse_string
- parse_file
- as_xxx
    - string
    - bool
    - double
    - long
- is_xxx
    - string
    - bool
    - double
    - long
    - object
    - array
- size
- to_string

具体用法参考下例：

```lua
local cjson=require('cjson')
local j = mongols_json.new()
local json_str=[[
{
"employees": [
{ "firstName":"John" , "lastName":"Doe" },
{ "firstName":"Anna" , "lastName":"Smith" },
{ "firstName":"Peter" , "lastName":"Jones" }
]
}
]]

j:parse_string(json_str)
print(j:to_string())

local employees=j:get('employees')
local item=mongols_json.new()
item:set('firstName','Hello')
item:set('lastName','World')
employees:append(item)

if employees:is_array() then 
    local size=employees:size()
    for i= 0,size-1 do
        local iter=employees:get(i)
        print(iter:get('firstName'):as_string()..' : ' ..iter:get('lastName'):as_string())
    end
end

j:set('employees',employees)
j:set('node1','test')
j:set('node2',true)
j:set('node3',3.1415926)
j:set('node4',100)
print(j:to_string())


local value = { true, { foo = "bar" } ,{1,2,3,'test',3.2}}
local json_text = cjson.encode(value)
print(json_text)

local mj=mongols_json.new()
mj:parse_string(json_text)
print(mj:to_string())


```


mongols_json不认识Lua的表类型，但lua-cjson认识。所以我内置了lua-cjson。以后，还会内置一些好用常用的第三方插件。

已内置的第三方插件包括:cjson和lfs。

此外，我还内置了一个tcp客户端类`mongols_tcp_client`,支持openssl。其api列表如下:
- ok
- send
- recv

具体用法可参考：

```lua

mongols_res:header('Content-Type','text/plain;charset=UTF-8')
mongols_res:status(200)
local host="www.baidu.com"
local port=443
local enable_ssl =true
local cli=mongols_tcp_client.new(host,port,enable_ssl)
local req="GET / HTTP/1.1\r\n\r\n"

if cli:ok() then
    if cli:send(req) then
        mongols_res:content(cli:recv(8096))
    else
        mongols_res:content("send failed.")
    end
else
    mongols_res:content("connection failed.")
end

```


## 单文件入口

单文件入口模式是web编程中常见的模式。默认情况下，lua_server没有开启对该模式的支持。欲开启此一支持，使用方法`set_enable_bootstrap`。

`example`下有例子代码，可参考：

```lua 
--index.lua

local echo=require('echo')
local route=require('route'):get_instance()


route:add({'GET'},'^/hello/([0-9a-zA-Z]+)/?$'
,function(req,res,param)
    res:content('hello,'..param[2])
    res:header('Content-Type','text/plain;charset=UTF-8')
    res:status(200)
end)

route:add({'GET'},'^/([0-9]+)/?$'
,function(req,res,param)
    res:content(param[2]) 
    res:header('Content-Type','text/plain;charset=UTF-8')
    res:status(200)
end)


route:add({'GET','POST','PUT'},'^/(\\w+)/?$'
,function(req,res,param) 
    local text= echo.concat('uri: ',req:uri(),'\n','method: ',req:method(),'\n',param[2])
    res:content(text) 
    res:header('Content-Type','text/plain;charset=UTF-8')
    res:status(200)
end)

route:run(mongols_req,mongols_res)


```

### c/c++ 模块

lua_server支持直接向服务器注册c/c++函数和类。例子:

```cpp

class person {
public:

    person() : name("Tom"), age(0) {
    }
    virtual~person() = default;

    person* set_name(const std::string& name) {
        this->name = name;
        return this;
    }

    person* set_age(unsigned int age) {
        this->age = age;
        return this;
    }

    const std::string& get_name() const{
        return this->name;
    }

    unsigned int get_age() {
        return this->age;
    }
private:
    std::string name;
    unsigned int age;
};

class studest : public person {
public:

    studest() : person() {
    }
    virtual~studest() = default;

    double get_score() {
        return this->score;
    }

    studest* set_score(double score) {
        this->score = score;
        return this;
    }
private:
    double score;
};

//some code

server.set_function(&mongols::sha1, "sha1");
server.set_function(&mongols::md5, "md5");

server.set_class(
            kaguya::UserdataMetatable<person>()
            .setConstructors < person()>()
            .addFunction("get_age", &person::get_age)
            .addFunction("get_name", &person::get_name)
            .addFunction("set_age", &person::set_age)
            .addFunction("set_name", &person::set_name)
            , "person");

server.set_class(
            kaguya::UserdataMetatable<studest, person>()
            .setConstructors < studest()>()
            .addFunction("get_score", &studest::get_score)
            .addFunction("set_score", &studest::set_score)
            , "studest");

```

```lua


local template = require "resty.template"
local view=template.new('name: {{name}}\
age: {{age}}\
score: {{score}}\
text:{{text}}\
text_md5: {{md5}}\
text_sha1: {{sha1}}')

local text='hello,world'

local s=studest.new()
s:set_score(74.6):set_name("Jerry"):set_age(14)

view.name=s:get_name()
view.age=s:get_age()
view.score=s:get_score()
view.text=text
view.md5=md5(text)
view.sha1=sha1(text)

mongols_res:header('Content-Type','text/plain;charset=UTF-8')
mongols_res:content(tostring(view))
mongols_res:status(200)

```

所以，无需写动态库扩展了。是不是太方便？