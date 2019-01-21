# js 服务器

这是为了方便使用javascript进行web开发准备的。

它与nodejs的不同在于它使用duktape引擎。虽然该引擎的执行效率比不上v8引擎，但是它是轻量级的，效率虽不够好，但也不太差。其执行效率方面的劣势可通过多进程化服务器、开启lru缓存等多种途径来弥补。压测显示，多进程化后，其并发性能高于nodejs。


![duktapeVSnode](image/duktapeVSnode.png)

上图是一个简单的hello,world测试，node在8888端口,js_server在9090端口。第一次压测js_server为单进程，第三次压测时采用多进程化js_server。

特别需要说明的是，在测试比较中发现，nodejs(v11)完成上述测试通常需要很多的内存消耗——50MB-120MB,与并发数量正相关。而js_server的内存消耗近通常不会超过1.8MB，与并发数量没什么关系。

来看例子：

```cpp

#include <mongols/js_server.hpp>

int main(int, char**) {
    int port = 9090;
    const char* host = "127.0.0.1";
    mongols::js_server
    server(host, port, 5000, 8096, 0/*2*/);
    server.set_root_path("html/js");
    server.set_enable_bootstrap(true);
    server.run("html/js/package", "html/js/package");
}


```

`run`方法的第一个参数表示js模块的搜索路径，第二个参数则表示c/c++模块的搜索路径。


```js
//index.js

/*
var foo = require('foo')
mongols_res.header('Content-Type','text/plain;charset=UTF-8')
mongols_res.content(foo.hello())
mongols_res.status(200)
*/

/*
var math = require('math/math')
var v=math.minus(1,3)/math.add(100,3)
mongols_res.header('Content-Type','text/plain;charset=UTF-8')
mongols_res.content(v.toString())
mongols_res.status(200)
*/


/*
var loaded = mongols_module.require('adder/libadder','adder')
mongols_res.header('Content-Type','text/plain;charset=UTF-8')
mongols_res.content(loaded?adder(1,2).toString():'failed load c module.')
mongols_res.status(200)
*/

/*
var loaded = mongols_module.require('concat/libconcat','concat')
mongols_res.header('Content-Type','text/plain;charset=UTF-8')
mongols_res.content(loaded?concat('Hello,','world'):'failed load c module.')
mongols_res.status(200)
*/



///*
mongols_res.header('Content-Type','text/plain;charset=UTF-8')
mongols_res.content('hello,world')
mongols_res.status(200)
//*/

```

## 模块

### js 模块

用`require`加载

### c/c++ 模块
#### 动态库模块

动态库可以用`mongols_module.require`加载:
```cpp
#include <mongols/lib/dukglue/duktape.h>

#ifdef __cplusplus
extern "C" {
#endif

    duk_ret_t adder(duk_context *ctx) {
        int i;
        int n = duk_get_top(ctx); /* #args */
        double res = 0.0;

        for (i = 0; i < n; i++) {
            res += duk_to_number(ctx, i);
        }

        duk_push_number(ctx, res);
        return 1; /* one return value */
    }


#ifdef __cplusplus
}
#endif

```

编译为libadder.so——编译时需使用`pkg-config --libs --cflags mongols`，然后使用：

```js

 var loaded = mongols_module.require('adder/libadder','adder')
 mongols_res.header('Content-Type','text/plain;charset=UTF-8')
 mongols_res.content(loaded?adder(1,2).toString():'failed load c module.')
 mongols_res.status(200)

```

`mongols_module.require`方法的第一个参数加载路径，第二个参数是函数名。返回布尔值。

也可以在动态库中注册c++函数或者类。方法是先如上注册普通的c函数，然后在该函数中注册c++函数和类。例如：

```cpp

#include <string>
#include <mongols/lib/dukglue/dukglue.h>
#include <mongols/lib/dukglue/duktape.h>
#include <mongols/js_server.hpp>

#include "dukglue/duktape.h"

class demo : public mongols::js_object {
public:
    demo() = default;
    virtual~demo() = default;

    std::string echo(const std::string& text) {
        return text;
    }

    static bool loaded;
};

bool demo::loaded = false;

#ifdef __cplusplus
extern "C" {
#endif

    duk_ret_t cpp(duk_context *ctx) {
        if (!demo::loaded) {
            dukglue_register_constructor<demo>(ctx, "demo");
            dukglue_register_method(ctx, &demo::echo, "echo");
            demo::loaded = true;
        }
        if (demo::loaded) {
            duk_push_true(ctx);
        } else {
            duk_push_false(ctx);
        }
        return 1; /* one return value */
    }

#ifdef __cplusplus
}
#endif

```

调用方法，先用`mongols_module.require`加载动态库，然后调用c函数注册c++类和函数：
```javascript

var loaded = mongols_module.require('cpp/libcpp', 'cpp')
var registered = cpp()
if (loaded && registered) {
    var a = new demo()
    mongols_res.header('Content-Type', 'text/plain;charset=UTF-8')
    mongols_res.content(a.echo('hello,cpp class'))
    mongols_res.status(200)
    mongols_module.free(a)
}


```

#### 类和函数注入
js_server还支持直接注入c/c++函数和类到服务器：

```cpp

class person : public mongols::js_object {
public:

    person() : mongols::js_object(), name("Tom"), age(0) {
    }
    virtual~person() = default;

    void set_name(const std::string& name) {
        this->name = name;
    }

    void set_age(unsigned int age) {
        this->age = age;
    }

    const std::string& get_name() const {
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
// some code


server.register_class_constructor<person>("person");
server.register_class_method(&person::set_age, "set_age");
server.register_class_method(&person::get_age, "get_age");
server.register_class_method(&person::set_name, "set_name");
server.register_class_method(&person::get_name, "get_name");

server.register_class_constructor<studest>("studest");
server.register_class_method(&studest::get_score, "get_score");
server.register_class_method(&studest::set_score, "set_score");
server.set_base_class<mongols::js_object, person>();
server.set_base_class<person, studest>();

server.register_function(&mongols::md5, "md5");
server.register_function(&mongols::sha1, "sha1");


```

```javascript

var handlebars = require('handlebars')
var s=new studest()
s.set_name("Jerry")
s.set_age(14)
s.set_score(74.6)
var text='hello,world'
var tpl=handlebars.compile('name: {{name}}\nage: {{age}}\nscore: {{score}}\ntext:{{text}}\ntext_md5: {{md5}}\ntext_sha1: {{sha1}}')
var content=tpl({name:s.get_name(),age:s.get_age(),score:s.get_score(),text:text,md5:md5(text),sha1:sha1(text)})
mongols_module.free(s)
mongols_res.header('Content-Type','text/plain;charset=UTF-8')
mongols_res.content(content)
mongols_res.status(200)

```

就这一点而言，js_server与lua_server是基本一致的。不过，lua的执行效率远高于duktape——如果不开启lru缓存。

特别注意:

- 所有需要注册的c++类必须继承`mongols::js_object`类且不允许多重继承。
- 所有注册的c++类，用`new`创建、使用完毕后，一定要使用`mongols_module.free`方法进行垃圾回收，因为duktape不管理c++类实例生命期。


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

## 单文件入口
js_server支持单文件入口模式。方法是引入`route`模块，与lua_server的使用方法类似：

```js

var route = require('route').get_instance()

route.add(['GET'], '^\/(hello|test)?\/?$', function (req, res, param) {
    res.header('Content-Type', 'text/plain;charset=UTF8')
    res.content(param.toString())
    res.status(200)
})

route.add(['POST', 'PUT'], '^\/.*$', function (req, res, param) {
    res.header('Content-Type', 'text/plain;charset=UTF8')
    res.content(req.uri())
    res.status(200)
})


route.run(mongols_req, mongols_res)



```


## 其他

`mongols_module`对象实例下有个`read`方法，可完整读取文件。

## 比较

v8比duktape快，这是毫无疑义的。单纯执行复杂一点的代码，前者的效率可能是后者的几倍甚至十几倍。

然而，对web应用而言，承载js引擎的服务器并发性能，具有更大的意义。例如，当js_server和nodejs都使用LRU缓存提升吞吐率时，js_server的吞吐率就会数倍于nodejs。这是由服务器性能决定的。比如：

```js

var http = require('http');
var md5=require('md5')
var sha1=require('sha1')
var studest=require('./studest')
var handlebars = require('./handlebars')
const QuickLRU = require('quick-lru');
const lru = new QuickLRU({maxSize: 1000});


http.createServer(function (request, response) {
    response.writeHead(200, {'Content-Type': 'text/plain;charset=UTF-8'});

	if(lru.has('key')){
    	response.end(lru.get('key'));
	}else{
		var s=new studest()
		s.set_name("Jerry")
		s.set_age(14)
		s.set_score(74.6)
		var text='hello,world'
		var tpl=handlebars.compile('name: {{name}}\nage: {{age}}\nscore: {{score}}\ntext:{{text}}\ntext_md5: {{md5}}\ntext_sha1: {{sha1}}')
		var content=tpl({name:s.get_name(),age:s.get_age(),score:s.get_score(),text:text,md5:md5(text),sha1:sha1(text)})
		lru.set('key',content)
    	response.end(content)
	}
}).listen(8888);

```

以上是个略复杂的nodejs例子。开启LRU缓存可使得吞吐率2千多提升至2万多。如果开启多进程模式，nodejs的吞吐率可达到4万多。

但是，同样的逻辑，用js_server实现：

```js

var handlebars = require('handlebars')
var s=new studest()
s.set_name("Jerry")
s.set_age(14)
s.set_score(74.6)
var text='hello,world'
var tpl=handlebars.compile('name: {{name}}\nage: {{age}}\nscore: {{score}}\ntext:{{text}}\ntext_md5: {{md5}}\ntext_sha1: {{sha1}}')
var content=tpl({name:s.get_name(),age:s.get_age(),score:s.get_score(),text:text,md5:md5(text),sha1:sha1(text)})
mongols_module.free(s)
mongols_res.header('Content-Type','text/plain;charset=UTF-8')
mongols_res.content(content)
mongols_res.status(200)

```

开启LRU缓存（仅仅1秒的缓存期）即可使得吞吐率从3百多提升至8万多——nodejs多进程也没有js_server轻快。如果同时多进程化，吞吐率可高达13万以上。如果再比较内存消耗，则js_server的优势会更加明显：每个nodejs进程都需要40MB+的内存，而js_server每个进程只需3MB+的内存。
因此，完全不必纠结于v8与duktape的比较：决定性的因素是承载脚本引擎的服务器，而非脚本引擎本身。
