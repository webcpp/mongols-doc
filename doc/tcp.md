# tcp 服务器

tcp 服务器由两个类组成，一个是tcp_server,另一个是它的子类tcp_threading_server。



先看代码：

```cpp

#include <mongols/tcp_server.hpp>
#include <mongols/tcp_threading_server.hpp>

int main(int,char**)
{
	auto f=[](const std::pair<char*,size_t>& input
					 , bool & keepalive
                     , bool& send_to_other
                     , mongols::tcp_server::client_t& client
                    , mongols::tcp_server::filter_handler_function& send_to_other_filter){
					keepalive= KEEPALIVE_CONNECTION;
					send_to_other=true;
					return std::string(input.first,input.second);
				};
	int port=9090;
	const char* host="127.0.0.1";
	
	mongols::tcp_server
    //mongols::tcp_threading_server
	server(host,port);
	
	server.run(f);

}

```



`run`方法需要一个handler函数，可以是lambda。这个函数给予开发者任意处置客户端及其输入和输出的自由。通过这个函数，开发者几乎可以完全控制服务器的每一次I/O。