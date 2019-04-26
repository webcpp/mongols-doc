# websocket 服务器

websocket在开发上有两条路径。第一条是自行处理消息，第二条是由服务器自动处理。前者需要开发者使用c/c++语言，后者则只需开发者使用javascript语言。

## 路径一


```cpp

#include <mongols/ws_server.hpp>

int main(int, char**)
{
    int port = 9090;
    const char* host = "127.0.0.1";
    mongols::ws_server server(host, port, 5000, 8096, std::thread::hardware_concurrency() /*0*/);
    //    if (!server.set_openssl("openssl/localhost.crt", "openssl/localhost.key")) {
    //        return -1;
    //    }

    auto f = [](const std::string& input, bool& keepalive, bool& send_to_other, mongols::tcp_server::client_t& client, mongols::tcp_server::filter_handler_function& send_to_other_filter, mongols::ws_server::ws_message_t& ws_msg_type) -> std::string {
        keepalive = KEEPALIVE_CONNECTION;
        send_to_other = true;
        if (ws_msg_type == mongols::ws_server::ws_message_t::BINARY) {
            ws_msg_type = mongols::ws_server::ws_message_t::TEXT;
            goto ws_close;
        }
        if (input == "close") {
        ws_close:
            keepalive = CLOSE_CONNECTION;
            send_to_other = false;
            return "close";
        }
        return input;
    };
    //server.set_enable_origin_check(true);
    //server.set_origin("http://localhost");
    //server.set_max_send_limit(5);
    server.run(f);
    //server.run();
}

```
以上代码中的`f`是一个handler,它是实现开发者自行处理消息的关键。以上实现是说，如果客户端发送`close`或者二进制消息到服务器，服务器就关闭它的连接，并且不转发该消息；除此而外，服务器会保持连接，并转发消息到所有在线客户端。这是走第一条路径的一个例子，它没有实现聊天群组功能和其他特性。

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


    <div id="req_output"></div><br />
    <div id="res_output"></div><br />
    <div id="err_output"></div><br />

    <script language="javascript" type="text/javascript">

        var wsUri = "ws://127.0.0.1:9090";
        var req_output, res_output, err_output;
        var j;

        function ab2str(buf) {
            return String.fromCharCode.apply(null, new Uint8Array(buf));
        }

        function str2ab(str) {
            var buf = new ArrayBuffer(str.length);
            var bufView = new Uint8Array(buf);
            for (var i = 0; i < str.length; i++) {
                bufView[i] = str.charCodeAt(i);
            }
            return buf;
        }

        function init() {
            req_output = document.getElementById("req_output");
            res_output = document.getElementById('res_output');
            err_output = document.getElementById('err_output')
            testWebSocket();
            var i = 0, max_i = 1000;
            j = setInterval(function () {
                if (i < max_i) {
                    var data = {};
                    data.gid = 0;
                    data.uid = 0;
                    data.gfilter = [];
                    data.ufilter = [];
                    data.name = 'Bugger';
                    data.message = "websocket test: " + i;
                    data.room = 'cpp';
                    data.time = ((new Date()).toLocaleString());
                    doSend(JSON.stringify(data))
                    i++;
                }
            }, 10)
        }

        function testWebSocket() {
            websocket = new WebSocket(wsUri);
            websocket.onopen = function (evt) { onOpen(evt) };
            websocket.onclose = function (evt) { onClose(evt) };
            websocket.onmessage = function (evt) { onMessage(evt) };
            websocket.onerror = function (evt) { onError(evt) };
        }

        function onOpen(evt) {
            writeToScreen("CONNECTED", err_output);
            doSend("WebSocket rocks");
        }

        function onClose(evt) {
            clearInterval(j);
            writeToScreen("DISCONNECTED", err_output);
        }

        function onMessage(evt) {
            writeToScreen('<span style="color: blue;">RESPONSE: ' + (evt.data) + '</span>', res_output);
        }

        function onError(evt) {
            writeToScreen('<span style="color: red;">ERROR:</span> ' + evt.data, err_output);
        }

        function doSend(message) {
            writeToScreen("SENT: " + message, req_output);
            websocket.send((message));
        }

        function writeToScreen(message, ele) {
            var pre = document.createElement("p");
            pre.style.wordWrap = "break-word";
            pre.innerHTML = message;
            if (ele.childNodes.length == 0) {
                ele.appendChild(pre);
            } else {
                ele.replaceChild(pre, ele.childNodes[0]);
            }
        }

        window.addEventListener("load", init, false);

    </script>
</body>

</html>

```

## 路径二

走第二条路径就简单多了。没有参数的 `run`可调用内置的群组管理机制，开发者仅仅在前端使用javascript即可完成对群组的控制和管理。是不是太方便！

怎么用？每一次用javascript服务器发送的消息应该是一个用`JSON.stringify`处理的json数据。该数据应该包含以下是个字段：

- uid，表示连接id，默认是0。
- gid, 表示连接所属群组id或群组id数组，默认是0。如果需退出某群组，用负的gid表示即可。
- gfilter，表示可接受该消息的群组gid数组。如果为空数组，则表示转发至所有在线群组。
- ufilter，表示可接受该消息的的uid数组。如果为空数组，则表示转发至所有在线连接。

其他字段为自选。服务器收到数据后，会解析该数据，根据以上四个字段作相应处理，并返回该数据。

基本实现，可参考[demo](https://github.com/webcpp/fusheng),基本体验可访问：[浮生beta](https://fusheng.hi-nginx.com/)

## 利用websocket上传文件

首先是服务器代码:

```cpp

#include <fstream>
#include <mongols/util.hpp>
#include <mongols/ws_server.hpp>

int main(int, char**)
{
    int port = 9090;
    const char* host = "127.0.0.1";
    mongols::ws_server server(host, port, 5000, 8192, std::thread::hardware_concurrency() /*0*/);
    //    if (!server.set_openssl("openssl/localhost.crt", "openssl/localhost.key")) {
    //        return -1;
    //    }

    std::unordered_map<size_t, std::pair<std::string, std::ofstream>> file_manage;

    auto f = [&](const std::string& input, bool& keepalive, bool& send_to_other, mongols::tcp_server::client_t& client, mongols::tcp_server::filter_handler_function& send_to_other_filter, mongols::ws_server::ws_message_t& ws_msg_type) -> std::string {
        keepalive = KEEPALIVE_CONNECTION;
        send_to_other = false;
        if (ws_msg_type == mongols::ws_server::ws_message_t::BINARY) {
            file_manage[client.sid].second << input;
            ws_msg_type = mongols::ws_server::ws_message_t::TEXT;
            return "continue";
        }
        std::vector<std::string> v = mongols::split(input, ':');
        if (v.size() > 1) {
            file_manage[client.sid].first = v.back();
            file_manage[client.sid].second = std::ofstream("upload/" + file_manage[client.sid].first, std::ios::binary | std::ios::out | std::ios::ate);
            return "start upload";
        }
        if (input == "upload success") {
            file_manage.erase(client.sid);
        }
        return input;
    };
    //server.set_enable_origin_check(true);
    //server.set_origin("http://localhost");
    //server.set_max_send_limit(5);
    server.run(f);
    //server.run();
}

```

接着是客户端代码:

```html

<!DOCTYPE html>
<html>

<head>
    <meta charset="utf-8" />
    <title>WebSocket Upload Test</title>
</head>

<body>
    <h2>WebSocket Update File Test</h2>

    <form id="frm1">
        <input type="file" id="myFile">
    </form>
    <div id='tag'>上传进度: <strong>0%</strong></div>
    <div id="log"></div>
    <script src="http://cdn.bootcss.com/jquery/3.4.0/jquery.min.js"></script>
    <script language="javascript" type="text/javascript">

        var ws, ws_uri = "ws://127.0.0.1:9090";

        var tag = $('#tag>strong');
        var default_step = 5;
        var upload_info = {
            size: 0, name: '', uploaded: 0, step: default_step
        }
        var file;

        function upload(upload_info) {
            if (upload_info.uploaded + upload_info.step > upload_info.size) {
                upload_info.step = upload_info.size - upload_info.uploaded;
            }
            var buffer = file.slice(upload_info.uploaded, upload_info.uploaded + upload_info.step)

            var reader = new FileReader();
            reader.readAsArrayBuffer(buffer);

            reader.onload = function (e) {
                ws.send(e.target.result);
                // log('<<正在上传数据...');
            }

            reader.onloadend = function (e) {
                tag.html(upload_info.uploaded / upload_info.size * 100 + '%');
                upload_info.uploaded = upload_info.uploaded + upload_info.step;
            }
        }
        $('#myFile').on('change', function (event) {
            file = document.getElementById("myFile").files[0];
            upload_info.size = file.size;
            upload_info.name = file.name;
            upload_info.uploaded = 0;
            upload_info.step = default_step;
            // console.log(upload_info);
            ws = new WebSocket(ws_uri);
            ws.onopen = function () {
                log('<<已连接上！');
                ws.send("name:" + upload_info.name);
            }

            ws.onclose = function () {
                log('<<连接已关闭！');
            }

            ws.onmessage = function (e) {
                // log(">>收到服务器消息:" + e.data + "\n");
                if (e.data == 'start upload') {
                    log('<<开始上传文件');

                    upload(upload_info);

                } else if (e.data == 'continue') {
                    if (upload_info.size > upload_info.uploaded) {
                        upload(upload_info);
                    } else {
                        ws.send('upload success');
                    }
                } else if (e.data == 'upload success') {
                    log('<<上传完成');
                    tag.html(upload_info.uploaded / upload_info.size * 100 + '%');
                    ws.close();
                }
            }

        });
        function log(text) {
            $("#log").append(text + "<br/>");
        }

    </script>
</body>

</html>

```
以上代码给出一个利用websocket进行文件分片上传的基本实现。该实现不限文件大小，只要合理配置`default_step`即分片大小，再大的文件也可轻松上传。

注意：分片大小应该小于服务器缓冲大小，一般不能超过(服务器缓冲大小-100)。分片大小越大，上传越快。

## 安全

ws_server自带安全检查方法。
通过`set_enable_origin_check`方法启用`Origin`HTTP头部检查，默认不检查。该方法需配合`set_origin`方法来配置连接请求需要匹配的`Origin`值。

另外，所有对tcp_server进行的安全配置对ws_server都是有效的。

