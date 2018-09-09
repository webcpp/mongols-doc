# sqlite 服务器

如果需要一个类似于mysql之类的小型关系数据库服务器，这个就是了。


来看代码：

```cpp
#include <mongols/sqlite_server.hpp>

int main(int,char**){
	int port = 9090;
	const char* host="127.0.0.1";
	mongols::sqlite_server 
	//server(host,port,5000,8096,2);
	server(host,port);
	server.run("html/sqlite/test.db");

}

```

## 用法
sqlite_server和leveldb_server一样，采用http协议进行交互，成功返回200，失败返回500。它把SQL语句分为三类：
- cmd
- query
- transaction

使用时通过POST方法提交SQL语句表单，服务器返回json。

`curl -X POST -d'sql_type=x' -d'sql=sql_statement' http://host/`

### 例子
请求：

`curl -X POST  -d'sql_type=query' -d'sql=select * from test limit 3;' http://127.0.0.1:9090/`

返回：

`{"error":null,"result":[{"id":1,"name":"a"},{"id":2,"name":"b"},{"id":3,"name":"c"}]}`
