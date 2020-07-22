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
