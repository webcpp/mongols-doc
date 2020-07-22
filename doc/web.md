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
	server.set_enable_mmap(true);
	server.run(f);
}

```

