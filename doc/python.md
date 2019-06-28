# python 绑定

为了在python中运用mongols，我提供了一个pymongols。它包括http_server和web_server。

仓库在[pymongols](https://github.com/webcpp/pymongols)


## 依赖
- mongols
- python2,3 devel

## 安装

很简单，`cd pymongols && make clean && make && sudo make install`

修改`Makefile`中的`PYVERSION`变量即可轻松适配开发者版本。



## API

参考`pymongols.cpp`，大体含义跟c++版http_server和web_server差不多，略有不同。

## 用法

```python

import pymongols


def req_filter(req):
    return True

def res_filter(req,res):
    res.content='hello,world'
    res.status=200

config={}
config['host']='127.0.0.1'
config['port']=9090
config['timeout']=5000
config['buffer_size']=8192
config['thread_size']=0
config['max_body_size']=4096
config['max_event_size']=64



server = pymongols.http_server(config['host'],config['port'],config['timeout'],config['buffer_size'],config['thread_size'],config['max_body_size'],config['max_event_size'])

#server.set_enable_daemon(True)
#server.set_enable_multiple_processes(True)
#server.set_pidfile("test1.pid")
#server.set_enable_lru_cache(True)
#server.set_lru_cache_expires(1)

server.run(req_filter,res_filter)

```

## 性能

### http_server 

pymongols单进程已极快，还原生支持多进程(从verion-0.1.7开始)。在python界，它应该是最快的了。

```text

Server Software:        mongols/1.6.8
Server Hostname:        localhost
Server Port:            9090

Document Path:          /
Document Length:        11 bytes

Concurrency Level:      1000
Time taken for tests:   0.898 seconds
Complete requests:      100000
Failed requests:        0
Keep-Alive requests:    100000
Total transferred:      13600000 bytes
HTML transferred:       1100000 bytes
Requests per second:    111341.22 [#/sec] (mean)
Time per request:       8.981 [ms] (mean)
Time per request:       0.009 [ms] (mean, across all concurrent requests)
Transfer rate:          14787.51 [Kbytes/sec] received


```

我都不屑于与tornado之流相比较，它们太太太太太次了——恕我不能理解那些高级python程序员的心情。

据说japronto很快，但它既不如pymongols快，也不如pymongols稳定，更不如pymongols轻（指内存消耗）。


### web_server

例子:

```python

import pymongols


config = {}
config['host'] = '127.0.0.1'
config['port'] = 9090
config['timeout'] = 5000
config['buffer_size'] = 8192
config['thread_size'] = 0
config['max_body_size'] = 4096
config['max_event_size'] = 64


server = pymongols.web_server(config['host'], config['port'], config['timeout'], config['buffer_size'],
                              config['thread_size'], config['max_body_size'], config['max_event_size'])

server.set_root_path("html")
server.set_mime_type_file("html/mime.conf")
server.set_list_directory(True)
server.set_enable_mmap(True)
#server.set_enable_daemon(True)
#server.set_enable_multiple_processes(True)
#server.set_pidfile("test4.pid")

server.run()


```

够快够轻：

```txt

Server Software:        mongols/1.6.8
Server Hostname:        localhost
Server Port:            9090

Document Path:          /nginx.html
Document Length:        1165 bytes

Concurrency Level:      1000
Time taken for tests:   0.908 seconds
Complete requests:      100000
Failed requests:        0
Keep-Alive requests:    100000
Total transferred:      127800000 bytes
HTML transferred:       116500000 bytes
Requests per second:    110073.36 [#/sec] (mean)
Time per request:       9.085 [ms] (mean)
Time per request:       0.009 [ms] (mean, across all concurrent requests)
Transfer rate:          137376.72 [Kbytes/sec] received


```

## 路由组件

这个组件依赖于pymongols,在`mongols_route.py`目录中，用`sudo python3 setup.py install`或者`sudo python2 setup.py install`安装。

例子：

```python

import pymongols
from mongols_route import application, template

import os

app = application()


@app.route(r'^/test/?$', ['GET'])
@app.route(r"^/$", ['GET'])
def hello_world(req, res, param):
    res.set_header('Content-Type', 'text/plain;charset=utf-8')
    res.content = 'hello,world'
    res.status = 200


@app.route(r"^/client/?$", ['GET', 'POST'])
def client(req, res, param):
    res.content = '{}<br>{}<br>{}<br>{}<br>{}'.format(
        req.client, req.method, req.uri, req.user_agent, req.param)
    res.status = 200


@app.route(r"^/hello/(?P<who>\w+)?$", ['GET'])
def hello(req, res, param):
    res.content = '{}={}'.format('who', param['who'])
    res.status = 200


@app.route(r'^/template/(?P<name>\w+)/(?P<age>\d+)/?$', ['GET'])
def tpl(req, res, param):
    param['title'] = '测试 jinja2 template'
    tpl_engine = template(os.path.join(os.getcwd(), 'python/templates'))
    res.content = tpl_engine.file_render('b.html', param)
    res.status = 200


@app.route(r'^/session/?$', ['GET'])
def sess(req, res, param):
    if 'test' in req.session:
        res.set_session('test', str(int(req.session['test'])+1))
        res.content = req.session['test']
    else:
        res.set_session('test', '0')
        res.content = 'null'
    res.status = 200


def req_filter(req):
    return True


def res_filter(req, res):
    app.run(req, res)


config = {}
config['host'] = '127.0.0.1'
config['port'] = 9090
config['timeout'] = 5000
config['buffer_size'] = 8192
config['thread_size'] = 0
config['max_body_size'] = 4096
config['max_event_size'] = 64


server = pymongols.http_server(config['host'], config['port'], config['timeout'], config['buffer_size'],
                               config['thread_size'], config['max_body_size'], config['max_event_size'])

# server.set_enable_daemon(True)
# server.set_pidfile("test2.pid")
# server.set_enable_lru_cache(True)
# server.set_lru_cache_expires(1)
server.set_enable_session(True)

server.run(req_filter, res_filter)



```


## example

`test`目录下有几个例子，可做参考。