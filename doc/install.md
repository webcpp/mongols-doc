# 依赖

- linux
- gcc(-std=c11) 
- g++(-std=c++11) 
- openssl 开发库

# 安装



mongols的安装非常简单:

`make clean && make -j2 && sudo make install && sudo ldconfig`，即可。

需要注意的是，mongols同时需要c(11)编译器和c++(11)编译器，也就是常见的gcc和g++。如果使用的clang和clang++，请自行修改Makefile中的相关部分，即：

```
CC=gcc
CXX=g++

```

改为:

```
CC=clang
CXX=clang++

```

## 使用

很简单：`pkg-config --libs --cflags mongols openssl`。


## 例子
 
安装后，`cd example && make`。`example`目录下有几个基本例子。可参考。
