# openssl 支持

mongols所包含的所有服务器设施均支持openssl化。也就是说，开发者可以为（tcp|http|resp）协议“一键”开启openssl支持。

开启方法很简单，调用`set_openssl`方法即可。该方法第一个参数为crt文件，第二个参数是key文件。第三个参数选择openssl协议版本(默认tlsv1.2),第四个参数是ciphers(默认`ECDHE-ECDSA-AES128-SHA256:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:RSA+AES128:!aNULL:!eNULL:!LOW:!ADH:!RC4:!3DES:!MD5:!EXP:!PSK:!SRP:!DSS`),第五个参数是flags(默认``SSL_OP_NO_COMPRESSION |SSL_OP_NO_TICKET| SSL_OP_CIPHER_SERVER_PREFERENCE | SSL_OP_SINGLE_ECDH_USE | SSL_OP_SINGLE_DH_USE`)。通常只需设置前两个参数即可。

一个例子：

```cpp

    if (!server.set_openssl("openssl/localhost.crt", "openssl/localhost.key"
            ,mongols::openssl::version_t::TLSv12,"AES128-GCM-SHA256")) {
        return -1;
    }

```


