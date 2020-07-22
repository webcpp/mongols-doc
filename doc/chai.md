# chai 服务器

考虑到c++11的普遍性和chaiscript语言本身的小众性，mongols-v1.6.0之后将不在支持。此节可忽略。

chaiscript是一种对C++程序员非常友好的脚本语言。具体用法参考:[https://github.com/ChaiScript/ChaiScript/blob/develop/cheatsheet.md](https://github.com/ChaiScript/ChaiScript/blob/develop/cheatsheet.md)

它直接支持脚本模块和C++模块，极为方便。可惜普及程度不高，很多c++程序员甚至不知道它能干什么。现在，它能像lua或者js一样，服务于web开发了。


来看例子:

```cpp

#include <string>
#include <mongols/lib/hash/md5.hpp>
#include <mongols/lib/hash/sha1.hpp>
#include <mongols/chai_server.hpp>

class person {
public:

    person() : name("Tom"), age(0) {
    }
    virtual~person() = default;

    void set_name(const std::string& name) {
        this->name = name;
    }

    void set_age(unsigned int age) {
        this->age = age;
    }

    const std::string& get_name() {
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

    void set_score(double score) {
        this->score = score;
    }
private:
    double score;
};

int main(int, char**) {
    int port = 9090;
    const char* host = "127.0.0.1";
    mongols::chai_server
    server(host, port, 5000, 8096, 0/*2*/);
    server.set_root_path("html/chai");
    server.set_enable_bootstrap(true);
    server.set_enable_lru_cache(true);
    server.set_lru_cache_expires(1);

    server.add(chaiscript::user_type<person>(), "person");
    server.add(chaiscript::constructor < person()>(), "person");
    server.add(chaiscript::fun(&person::get_age), "get_age");
    server.add(chaiscript::fun(&person::set_age), "set_age");
    server.add(chaiscript::fun(&person::get_name), "get_name");
    server.add(chaiscript::fun(&person::set_name), "set_name");


    server.add(chaiscript::user_type < studest>(), "studest");
    server.add(chaiscript::constructor < studest()>(), "studest");
    server.add(chaiscript::fun(&studest::get_score), "get_score");
    server.add(chaiscript::fun(&studest::set_score), "set_score");


    server.add(chaiscript::base_class<person, studest>());

    server.add(chaiscript::fun(&mongols::md5), "md5");
    server.add(chaiscript::fun(&mongols::sha1), "sha1");

    std::vector<std::string> package_paths = {"html/chai/package"}
    , package_cpaths = {"html/chai/package/test"};

    for (auto& item : package_paths) {
        server.set_package_path(item);
    }
    for (auto& item : package_cpaths) {
        server.set_package_cpath(item);
    }
    server.run();
}

```

```shell

use("foo/foo.chai")
use("demo.chai")
load_module("test")

///*
{
    var a = foo()
    var b = demo()
    a.set_value("hello,")
    b.set_value("world")
    mongols_res.header("Content-Type","text/plain;charset=UTF-8")
    mongols_res.content(concat(a.get_value(),b.get_value()))
    mongols_res.status(200)
}
//*/


/*
{
var a = studest()
a.set_score(23.8)
a.set_name("Jerry")
a.set_age(14)
mongols_res.header("Content-Type","text/plain;charset=UTF-8")
mongols_res.content(concat(a.get_name(),a.get_age().to_string()))
mongols_res.status(200)
}
*/


```

脚本模块通过`use`引进，c++模块通过`load_module`引进。临时变量通过`{}`大括号自动清除。

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


## 模块
### 脚本模块
```shell

class demo {
    var value;
    def demo() { this.value = "demo"; }
    def set_value(x) { this.value = x;}
    def get_value() { return this.value; }
};

```

### C++ 模块
可以直接向服务器注入C/C++函数和类，也可以写动态库：

```cpp

#include <string>
#include <mongols/lib/chaiscript/chaiscript.hpp>

std::string concat(const std::string& a,const std::string& b){
    return a+b;
}

CHAISCRIPT_MODULE_EXPORT chaiscript::ModulePtr create_chaiscript_module_test() {
    chaiscript::ModulePtr m(new chaiscript::Module());
    m->add(chaiscript::fun(&concat), "concat");
    return m;
}

```

详细例子请参考:[https://github.com/webcpp/mongols/tree/master/example/html/chai](https://github.com/webcpp/mongols/tree/master/example/html/chai)

## 其他

比较于lua_server和js_server,chai_server的性能属于中等偏上。当然，开启lru缓存的话，三者没什么效率上的分别。

因此从开发者的角度来看，选择lua,chai还是js，由开发者的品味决定。
