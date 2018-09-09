# 安装

mongols的安装非常简单。

先安装三个最普通不过的依赖：
- pcre-devel
- zlib-devel
- openssl-devel

对于ubuntu: `sudo apt-get install libpcre3-dev zlib1g-dev libssl-dev`。

对于centos: `yum install pcre-devel zlib-devel openssl-devel`。

然后`make clean && make -j2 && sudo make install && sudo ldconfig`，即可。

唯一需要注意的是，mongols同时需要c编译器和c++编译器，也就是常见的gcc和g++。如果使用的clang和clang++，请自行修改Makefile中的相关部分，即：

```
CC=gcc
CXX=g++

```

改为:

```
CC=clang
CXX=clang++

```
