# daemon

mongols中包含的所有服务器都没有daemon化。这意味着：如果需要服务器安装为系统服务并自动启动，那么你好需要写写东西。这种事情对熟悉linux daemon api的人来说，说轻而易举的。

如果不熟悉，请参考[fusheng](https://github.com/webcpp/fusheng/tree/master/fusheng)。