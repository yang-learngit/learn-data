# Nginx在请求头request中增加header参数

应用场景：

1、切流，一个线上服务，新增了一个功能，需要公网内部测试，这时候可以根据来源ip & path设置不同的header信息，配合业务逻辑实现内外网流量切分，保证内测功能只有内网可见。

2、使用非标准标头向上游服务器通知用户的IP地址和其他请求属性。



有两种方式可以实现这一操作：

1：nginx反向代理（需要两个nginx服务）

在nginx反向代理服务器通过使用proxy_set_header实现。

​      e.    proxy_set_header X-Forwarded-For 127.0.0.1;

2：安装模块Passenger

​      [Passenger用户手册] https://github.com/biti/passenger-doc-zh/wiki/Passenger%E7%94%A8%E6%88%B7%E6%89%8B%E5%86%8Cnginx%E7%89%88

安装完成后使用：

passenger_set_cgi_param HTTP_X_FORWARDED_FOR 127.0.0.1; 

来增加请求header中的参数



