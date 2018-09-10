# web 服务器

这个服务器专门处理静态文件。来看代码:

```cpp

#include <mongols/web_server.hpp>

int main(int,char**)
{
	auto f=[](const mongols::request& req){
		if(req.method=="GET"&&req.uri.find("..")==std::string::npos){
			return true;
		}
		return false;
	};
	int port=9090;
	const char* host="127.0.0.1";
	mongols::web_server 
	server(host,port,5000,512000,2);
	//server(host,port);
	server.set_root_path("html");
	server.set_mime_type_file("mime.conf");
	server.set_list_directory(true);
	server.set_enable_mmap(false);
	server.run(f);
}

```

web_server可以通过`set_enable_mmap`来启用内存映射读取，在某些情况下可提升性能。

函数`f`可用来根据http请求过滤客户端。

关于并发性能，可参考下图：
![压力测试](image/mongols_2.png)

以上测试并不是说web_server一定比nginx更适合静态文件服务。更多的测试——包括ab和wrk——使我相信，在大规模并发(并发数5000以上)的情况下，如果文件较小，则web_server比nginx更快；如果文件较大，则web_server比nginx更稳定。比如nginx的欢迎页面，用wrk跑9000并发，web_server比nginx更快，但用ab 跑3000(nginx基本跑不动5000+并发,4000并发已经频繁报错)并发，nginx比web_server更快，但报错也较多。如图:

![wrk](image/wrk.png)