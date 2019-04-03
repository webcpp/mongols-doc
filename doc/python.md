# python 绑定

为了在python中运用mongols，我提供了一个pymongols。它现在仅仅包括http_server。

仓库在[pymongols](https://github.com/webcpp/pymongols)


## 依赖
- mongols
- python2,3 devel

## 安装

很简单，`cd pymongols && make clean && make && sudo make install`

修改`Makefile`中的`PYVERSION`变量即可轻松适配开发者版本。



## API

参考`pymongols.cpp`，大体含义跟c++版http_server差不多，略有不同。

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
#server.set_pidfile("test1.pid")
#server.set_enable_lru_cache(True)
#server.set_lru_cache_expires(1)

server.run(req_filter,res_filter)

```

## 性能

pymongols单进程已极快，还原生支持多进程(从verion-0.6.7开始)。在python界，它应该是最快的了。

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

## 路由组件

这个组件依赖于pymongols,在`mongols_route.py`目录中，用`sudo python3 setup.py install`或者`sudo python2 setup.py install`安装。


## example

`test`目录下有三个例子，可做参考。