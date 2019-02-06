# 压缩

所有基于http协议的服务器组件均支持压缩，请求头`Accept-Encoding`的值需要包含`deflate`或者`gzip`。

即便请求头`Accept-Encoding`的值包含`deflate`或者`gzip`，服务器还会参考静态变量`http_server::zip_min_size`(默认1024,即1KB)和`http_server::zip_max_size`(默认307200,即300KB)来决定是否压缩。仅当响应体大小应该介于以上两值之间。

此外，服务器还会参考`http_server::zip_mime_type`静态变量来决定是否压缩。该列表变量是给出需要压缩的`Content-Type`的`mime-type`类型设定。默认类型包括:
- text/html
- text/css
- text/plain
- application/x-javascript
- text/xml

压缩层次由`http_server::zip_level`静态变量指定，默认是`Z_BEST_SPEED`即1，其值最大为9。一般情况下，指定1或者2即可。

压缩会消耗服务器资源，也会降低服务器的并发率，所以需谨慎配置。