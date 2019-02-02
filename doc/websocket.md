# websocket 服务器

websocket在开发上有两条路径。第一条是自行处理消息，第二条是由服务器自动处理。前者需要开发者使用c/c++语言，后者则只需开发者使用javascript语言。

来看代码：

```cpp
#include <mongols/ws_server.hpp>

int main(int,char**){
	int port=9090;
	const char* host="127.0.0.1";
	mongols::ws_server server(host,port,5000,8096,4);

	auto f=[](const std::string& input
            , bool& keepalive
            , bool& send_to_other
            , mongols::tcp_server::client_t& client
            , mongols::tcp_server::filter_handler_function& send_to_other_filter){
			    keepalive = KEEPALIVE_CONNECTION;
			    send_to_other=true;
			if(input=="close"){
				keepalive = CLOSE_CONNECTION;
				send_to_other = false;
			}
			return input;
	};
	server.run(f);
	//server.run();
}
```
以上代码中的`f`是一个handler,它是实现开发者自行处理消息的关键。以上实现是说，如果客户端发送`close`消息到服务器，服务器就关闭它的连接，并且不转发该消息；除此而外，服务器会保持连接，并转发消息到所有在线客户端。这是走第一条路径的一个例子，它没有实现聊天群组功能和其他特性。

浏览器测试代码:

```html

<!DOCTYPE html>
<html>

<head>
  <meta charset="utf-8" />
  <title>WebSocket Test</title>
</head>

<body>
  <h2>WebSocket Test</h2>

  <div id="output"></div>
  <script language="javascript" type="text/javascript">

    var wsUri = "ws://127.0.0.1:9090/";
    var output;


    function init() {
      output = document.getElementById("output");
      testWebSocket();
      var i = 0, max_i = 1000;
      setInterval(function () {
        if (i < max_i) {
          doSend('websocket test ' + i)
          i++;
        }
      }, 500)
    }

    function testWebSocket() {
      websocket = new WebSocket(wsUri);
      websocket.onopen = function (evt) { onOpen(evt) };
      websocket.onclose = function (evt) { onClose(evt) };
      websocket.onmessage = function (evt) { onMessage(evt) };
      websocket.onerror = function (evt) { onError(evt) };
    }

    function onOpen(evt) {
      writeToScreen("CONNECTED");
      doSend("WebSocket rocks");
    }

    function onClose(evt) {
      writeToScreen("DISCONNECTED");
    }

    function onMessage(evt) {
      writeToScreen('<span style="color: blue;">RESPONSE: ' + evt.data + '</span>');
    }

    function onError(evt) {
      writeToScreen('<span style="color: red;">ERROR:</span> ' + evt.data);
    }

    function doSend(message) {
      writeToScreen("SENT: " + message);
      websocket.send(message);
    }

    function writeToScreen(message) {
      var pre = document.createElement("p");
      pre.style.wordWrap = "break-word";
      pre.innerHTML = message;
      if (output.childNodes.length == 0) {
        output.appendChild(pre);
      } else {
        output.replaceChild(pre, output.childNodes[0]);
      }
    }

    window.addEventListener("load", init, false);

  </script>
</body>

</html>


```

走第二条路径就简单多了。没有参数的 `run`可调用内置的群组管理机制，开发者仅仅在前端使用javascript即可完成对群组的控制和管理。是不是太方便！

怎么用？每一次用javascript服务器发送的消息应该是一个用`JSON.stringify`处理的json数据。该数据应该包含以下是个字段：

- uid，表示连接id，默认是0。
- gid, 表示连接所属群组id或群组id数组，默认是0。如果需退出某群组，用负的gid表示即可。
- gfilter，表示可接受该消息的群组gid数组。如果为空数组，则表示转发至所有在线群组。
- ufilter，表示可接受该消息的的uid数组。如果为空数组，则表示转发至所有在线连接。

其他字段为自选。服务器收到数据后，会解析该数据，根据以上四个字段作相应处理，并返回该数据。

基本实现，可参考[demo](https://github.com/webcpp/fusheng),基本体验可访问：[浮生beta](https://fusheng.hi-nginx.com/)


## 安全

ws_server自带安全检查方法。
通过`set_enable_origin_check`方法启用`Origin`HTTP头部检查，默认不检查。该方法需配合`set_origin`方法来配置连接请求需要匹配的`Origin`值。

另外，`set_max_send_limit`方法可配置每一连接每秒发送消息的最大频率，默认值是5，即最多每秒5次消息发送。