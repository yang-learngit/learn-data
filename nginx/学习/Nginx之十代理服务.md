# Nginx之十代理服务

## 一.正向代理

### **1.概念**

   正向代理最大的特点是客户端非常明确要访问的服务器地址；服务器只清楚请求来自哪个代理服务器，而不清楚来自哪个具体的客户端；正向代理模式屏蔽或者隐藏了真实客户端信息。正向代理是访问外部网络。比如国内访问不到的网址，通过代理访问。

### **2.指令**

| 指令语法                         | 说明                                                         |
| -------------------------------- | ------------------------------------------------------------ |
| resolver address … [valid =time] | 该指令用于指定DNS服务器的IP地址，DNS服务器的主要功能解析域名，将域名解析为IP地址。<br/>**address：** 表示 DNS服务器的IP地址，如果不指定端口，默认是35 
**time：** 设置数据包在网络中的有效时间 |
| resolver_timeout time            | 该指令用于设置DNS服务器域名解析超时的时间                    |
| proxy_pass URL                   | 这个指令用来设置代理服务器的协议和地址，这个不仅仅用于Nginx服务器的代理服务，更主要的是应用于反向代理服务器<br/>URL即为设置的代理服务器的协议和地址 |

### **3.正向代理例子**

```
...
server {
   resolver 8.8.8.8；  #设置的DNS服务器为8.8.8.8，使用默认端口53
   listen 82;              #代理服务监听的端口是82
   location / {
     proxy_pass http://$http_host$request_uri;   #这里是代理服务器地址，$http_host$request_uri这两个是Nginx配置自动获取的主机和URI的变量，一般配置不要改变该指令的配置，意思就是这一行就是写死的。
   } 
}
注意点：正向代理不支持代理https站点，这里不能使用server_name指令，并且必须使用resolver指令，用来处理解析接收到的域名。
```



## 二、反向代理

### 1.概念

  反向代理（Reverse Proxy）方式是指以代理服务器来接受Internet上(客户端)的连接请求，然后将请求转发给内部网络上的服务器；并将从服务器上得到的结果返回给Internet上请求连接的客户端，此时代理服务器对外就表现为一个服务器。

### 2.反向代理基本设置的21个指令

| 指令语法       | 说明                                                         |
| -------------- | ------------------------------------------------------------ |
| proxy_pass URL | 设置被代理服务器的地址，URL可以主机名，IP加端口号的形式，传输协议通常是"http"、“https://”，URL也可以是upstream设置的一组服务器。 注意的是如果upstream组中的服务器没有使用“http://”或者“https://”那么 proxy_pass就需要添加这个协议。<br/>注意点一：https/http的写法
upstream proxy_sers { 
 server 192.168.123.1/uri;
 server 192.168.123.2/uri;
 server 192.168.123.3/uri;
}
server{
 listen 80;
 server_name www.form1.cn;
 location / {
  proxy_pass http://proxy_sers; #server中指明 http:// 在proxy_pass就不需要指定,这里写了，那么upstream里面就不需要了
 }
}
注意点二：
被代理的网址是否包含URI，如果没有包含就不会改变原来的URI，否则就会用新的URI代替原来的URI
server{
 listen 80;
 server_name www.myweb.com; 
 location /server/ { 
  proxy_pass http://123.11.11.1; # 这里网址没有使用URI，所以访问www.myweb.com/server/的时候，会代理到 http://123.11.11.1/server/。 如果这里时候使用的是proxy_pass http://123.11.11.1/user/ 那么当同样访问的原来这个网址的时候，就直接会替换掉原来的URI(/server/),代理到 http://123.11.11.1/user/ 上去
 }
} |

1）**proxy_pass**
设置被代理服务器的地址，可以主机名，IP加端口号的形势，语法位：proxy_pass URL，下面举例说明：

```
`upstream proxy_sers {``    ``server 192.168.123.1/URI;``    ``server 192.168.123.2/URI;``    ``server 192.168.123.3/URI;``}``server{``    ``listen 80;``    ``server_name www.form1.cn;``    ``location / {``    ``proxy_pass http:``//proxy_sers;          #server中指明 http:// 在proxy_pass就不需要指定``    ``}``}`
```

proxy_pass中URL是否包含URI的问题，当访问 www.form1.cn/server

```
`location /server/ {``    ``server_name www.form1.cn;``    ``proxy_pass http:``//192.168.123.2;``}`
```

由于proxy_pass中URL不包含URI，所以转向的地址为 http://192.168.123.2/server

```
`location /server/ {``    ``server_name www.form1.cn;``    ``proxy_pass http:``//192.168.123.2/loc;``}`
```

由于proxy_pass中URL指明了URI，所以转向的地址为 http://192.168.123.2/loc

```
`proxy_pass http:``//192.168.123.2;   www.form1.cn/server/  http://192.168.123.2/server/``proxy_pass http:``//192.168.123.2/;  www.form1.cn/server/  http://192.168.123.2/`
```


2）**proxy_hide_header**
Nginx在发送HTTP响应时，可以去掉相关的响应头信息： 

```
`proxy_hide_header Set-Cookie;`
```


3）**proxy_pass_header**
默认情况下，Nginx服务器在发送响应报头时，报文头中不包含"Date 、Server、X-Accel"等来自代理服务器的头域信息。proxy_pass_header可以调置这些头域信息以被发送
语法为

```
`proxy_pass_header field;`
```

field 为需要发送的头域

4）**proxy_pass_request_body**
配置是否将客户端请求的请求体发送给代理服务器。
语法为

```
`proxy_pass_request_body on | off;`
```

默认为开启状态(on)

5）**proxy_pass_request_headers**
配置是否将客户端请求的请求头发送给代理服务器。
语法为

```
`proxy_pass_request_headers on | off;`
```

默认为开启状态(on)

6）**proxy_set_header**
可以更改nginx服务接收到的客户端请求的请求头信息，然后将新头发送给被代理服务器。
语法为

```
`proxy_set_header field value;`
```

默认值：

```
`proxy_set_header Host ``$proxy_host``;``proxy_set_header Connection close;`
```


7）**proxy_set_body**
可以更改nginx服务接收到的客户端请求的请求体信息，然后将新体发送给被代理服务器。
语法为

```
`proxy_set_body value;`
```


8）**proxy_bind**
强制将与代理主机的连接绑定到指定IP地址。
语法为

```
`proxy_bind address; 其中 address为IP地址`
```


9）**proxy_connect_timeout**
配置Nginx服务器与后端被代理服务器尝试建立连接的超时时间，
语法为

```
`proxy_connect_timeout time; 其中time为设置的超时时间，默认为 60s;`
```


10）**proxy_read_timeout**
配置Nginx服务器向后端被代理服务器（组）发出read请求后，等待响应的超时时间，
语法为

```
`proxy_read_timeout time; 其中time为设置的超时时间，默认为 60s;`
```


11）**proxy_send_timeout**
配置Nginx服务器向后端被代理服务器（组）发出write请求后，等待的响应超时时间，
语法为

```
`proxy_send_timeout time; 其中time为设置的超时时间，默认为 60s;`
```


12）**proxy_http_version**
设置用于Nginx服务器提供代理服务的HTTP协议版本，
语法为

```
`proxy_http_version 1.0 | 1.1;`
```

默认为 1.0版本。1.1版本支持upsteam服务器组设置中的keepalive指令

13，proxy_method
设置Nginx服务器请求被代理服务器时使用的请求方法，一般为 POST 或者 GET ，设置了该值，客户端的请求方法将被忽略。
语法为

```
`proxy_method POST | GET;`
```


14）**proxy_ignore_client_abort**
用于设置在客户端中断网络请求时，Nginx服务器是否中断对被代理服务器的请求，
语法为

```
`proxy_ignore_client_abort on | off;`
```

默认为 off ，客户端断了，nginx对被代理服务器也断

15）**proxy_ignore_headers**
用于设置一些HTTP响应头中的头域，Nginx服务器接收到被代理服务器的响应数据后不会处理被设置的头域
语法为

```
`proxy_ignore_headers field....`
```

field为要设置的HTTP响应头的头域，例如 "X-Accel-Redirect 、X-Accel-Expires 、EXpires 、Cache-Control 、Set-Cookie"

16）**proxy_redirect**
假设前端url是example.com。后端server域名是csdn123.com，那么后端server在返回refresh或location的时候，host为csdn123.com，显然这个信息直接返回给客户端是不行的，需要nginx做转换，这时可以设置:

```
`proxy_redirect http:``//csdn123.com  /`
```

nginx会将host及port部分替换成自身的server_name及listen port。不过这种配置对server_name有多个值的情况下支持不好。
我们可以用nginx内部变量来解决这一问题：

```
`proxy_redirect http:``//csdn123.com http://$host:$server_port`
```


17）**proxy_intercept_errors**
配置一个状态是开启还是关闭。
开启时：被代理服务器返回状态码为 400或400以上，Nginx服务器使用自己的 error_page。
关闭时：Nginx直接将代理服务器返回的状态码返回给客户端

语法为

```
`proxy_intercept_errors on | off;`
```


18）**proxy_headers_hash_max_size**
存放HTTP报文头的哈希表的容量
默认为

```
`proxy_headers_hash_max_size 512;`
```


19）**proxy_headers_hash_bucket_size**
Nginx服务器申请存放HTTP报文头的哈希表容量的单位大小
默认为

```
`proxy_headers_hash_max_size 64;`
```

对(18和19)：在启动 Nginx 的时候，有时候会遇到这样的一个错误：
解决办法就是在配置文件中新增以下配置项：

```
`proxy_headers_hash_max_size 51200;``proxy_headers_hash_bucket_size 6400;`
```


20）**proxy_next_upstream**
如果Nginx定义了 upstream 后端服务器组，如果组内有异常情况，将请求顺次交给下一个组内服务器处理
语法为

```
`proxy_next_upstream status...`
```

可以是一个也可以是多个

```
`status = error,timeout,invalid_header,http_500 502 503 504 404,off`
```


21）**proxy_ssl_session_reuse**
该指令用于配置是否使用基于SSL安全协议的会话连接（htts://）被代理服务器，
语法为

```
`proxy_ssl_session_reuse on | off;`
```

默认为开启 on，如果在错误日志中发现“SSL3_GET_FINISHED:digest check fiailed”的情况，关闭该指令即可







