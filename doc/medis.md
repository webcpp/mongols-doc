# medis 服务器


medis_server 是一个使用RESP协议进行交互的混合型缓存服务器。

它像redis一样，提供多种复杂数据结构——字符串、哈希、列表、集合、队列和堆栈——但是它提供两种保存选择:内存或者硬盘(通过leveldb)。如果这些数据结构还不能满足需要，它还提供sqlite，用户可以自由通过SQL定义数据结构。

所有这些功能，都能通过RESP协议进行访问。也就是说，可以用redis-cli实现与medis_server的交互。

来看例子：

```cpp
#include <mongols/medis_server.hpp>

int main(int,char**){
	int port = 9090;
	const char* host="127.0.0.1";
	mongols::medis_server
	server(host,port,5000,8096,2/*0*/);
	server.ready();
	server.run("html/leveldb","html/sqlite/test.db");
}

```

# 压力测试

![medis_server](image/medis.png)

# 提示

- 高热数据应该保存放在内存，低热数据应该保存在硬盘，中热数据按读多还是写多相机选择内存或硬盘。

- 对于非字符串的高级数据结构，首选在内存。

- 特殊情况下，可以用[messagepack](https://msgpack.org/)将高级数据结构转换为字符串类型进行存储。

- 保存在内存中的数据按LRU算法更新。

- 服务器退出时，只有内存数据会消失。

# 命令列表

所有命令均为大写，前缀为`_`下划线的表示保存在内存中。所有保存在内存中的数据均可以通过`_FLUSHALL`命令全部清除。

所有命令暂不支持过期时间机制。

## 常规

- ECHO value

- PING [value]

- _FLUSHALL



## 字符串

- SET key value or _SET key value

- GET key or _GET key

- DEL key or _DEL key

- EXISTS key or _EXISTS key

- MSET key value [key value ...] or _MSET key value [key value ...]

- MGET key [key ...] or _MGET key [key ...]

- GETSET key value or _GETSET key value

- STRLEN key or _STRLEN key

- APPEND key value or _APPEND key value

- INCR key or _INCR key

- INCRBY key increment or _INCRBY key increment

- DECR key or _DECR key

- DECRBY key decrement or _DECRBY key decrement

## 哈希

- HSET key field value or _HSET key field value

- HGET key field or _HGET key field

- HDEL key field [field ...] or _HDEL key field [field ...]

- HEXISTS key field or _HEXISTS key field

- _HERASE key

## 列表

- LFRONT key or _LFRONT key

- LBACK key or _LBACK key

- LPUSH_FRONT key [value ...]  or _LPUSH_FRONT key [value ...] 

- LPUSH_BACK key [value ...]  or _LPUSH_BACK key [value ...] 

- LPOP_FRONT key  or _LPOP_FRONT key 

- LPOP_BACK key   or _LPOP_BACK key

- LLEN key or _LLEN key 

- LRANGE key start stop or _LRANGE key start stop

- _LEXISTS key value 

- _LERASE key

## 集合(有序)

- SADD SADD key member [member ...] or _SADD key member [member ...]

- SDEL key member [member ...] or _SDEL key member [member ...]

- SMEMBERS key or _SMEMBERS key

- SEXISTS key or _SEXISTS key

- SCARD key or _SCARD key

- SDIFF key1 key2 or _SDIFF key1 key2

- SINTER key1 key2 or _SINTER key1 key2

- SUNION key1 key2 or _SUNION key1 key2

- SSYDIFF key1 key2 or _SSYDIFF key1 key2

- _SERASE key

## 队列

- QPUSH key member [member ...] or _QPUSH key member [member ...]

- QPOP key or _QPOP key

- QEMPTY key or _QEMPTY key

- QFRONT key or _QFRONT key

- QBACK key or _QBACK key

- _QERASE key

## 堆栈

- ZPUSH  key member [member ...] or _ZPUSH key member [member ...]

- ZPOP key or _ZPOP key

- ZTOP key or _ZTOP key

- ZEMPTY key or _ZEMPTY key

- _ZERASE key

## SQLite

- SQLCMD statement

    `SQLCMD 'update a set name="test" where id=4 or id=5'`

- SQLBINDCMD statement

    `SQLBINDCMD "INSERT INTO A(ID,NAME)VALUES(:ID,:NAME);" ":ID" 4 ":NAME" "CPP"`

- SQLTRANSACTION statement

    `SQLTRANSACTION 'INSERT INTO A(ID,NAME)VALUES(14,"14");UPDATE A SET NAME="TEST" WHERE ID=14;'`

- SQLQUERY statement

    `SQLQUERY 'SELECT * FROM A ORDER BY ID desc limit 3'`




