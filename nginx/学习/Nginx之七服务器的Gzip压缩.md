# Nginx 之七 服务器的Gzip压缩

Gzip压缩可以在http块，server块和location块中配置

  Nginx服务器是通过ngx_http_gzip_module模块、ngx_http_gzip_static_module模块、ngx_http_gunzip_module模块对这些指令进行解析和处理。

## 1.ngx_http_gzip_module模块（压缩）处理的指令

这个模块主要是负责Gzip功能的开启和设置，对响应数据进行在线实时压缩。指令如下：

| 语法                                                         | 说明                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| 是否开启压缩<br>gzip on \| off                               | 该指令用于开启或者关闭Gzip功能，<br/>默认情况下该指令设置为off,只有为on时，下面指令才有效 |
| 压缩需要申请的空间<br/>gzip_buffers number size              | number:指定Nginx服务器需要向系统申请缓存空间的个数 <br/>size :指定缓存空间的大小;<br/>根据该配置，nginx在对响应输出数据经压缩时需要向系统申请number*size 大小的空间用来存储压缩数据.<br/>默认大小是128,其中size的值是取系统内存页一页的大小，为4kb或者8kb 即： gzip_buffers 32 4k |
| 压缩等级<br/>gzip_comp_level level                           | 该指令用于设置Gzip压缩程度，包括级别1-9，<br/>级别1表示压缩程度最低效率最高，
级别9表示 压缩程度最高，效率最低 。
默认值是1 |
| 根据客户端类型动态开启压缩<br/>gzip_disabel regex…           | 针对不同的客户端的请求，可以选择性的开启和关闭Gzip的功能 ，Nginx服务器在响应这种种类的客户端请求时，不使用Gzip功能缓存响应输出数据。regex根据客户端的浏览器标志（User-Agent,UA）设置，支持正则表达式。<br/>例如 gzip_disable MSIE [4-6] 表示MSIE4、MSIE5、MSIE6、这些客户端请求后，nginx不进行Gzip压缩返回 |
| 根据浏览器版本是否开启压缩<br/>gzip_http_version version     | 早期的浏览器也许不支持Gzip的自解压，因此客户端有可能会看到乱码，所以针对不同的HTTP协议版本，需要选择性地开启或者关闭Gzip功能。<br/> 默认设置为1.1版本，即只有客户端1.1及以上版本的HTTP协议，才使用Gzip压缩功能，目前来看大部分浏览器都支持Gzip自解压，所以使用默认值即可。 |
| 根据响应数据大小是否开启压缩<br/>gzip_min_length length      | Gzip压缩功能对大数据的压缩效果明显，但是如果压缩的数据很小，就可能数据越压缩越大，因此我们也应该有选择的压缩。当响应页面的大小大于该值才开启Gzip压缩功能。响应页面的大小通过HTTP响应头部中的Content-Length指令获取，如果我们使用了Chunk编码压缩，Content-Length或不存在或被忽略，这条指令这不起作用。<br>默认值是20，设置为0表示不管页面多大都要压缩。建议大于1kb的压缩；比如：gzip_min_length 1024 |
| 根据响应头部是否开启压缩<br/>gzip_proxied off\|expired\|<br/>no-cache\|no-store\|<br/>private\|no_last_modified\|<br/>no_etag\|auth\|any …; | **off**:表示关闭Nginx服务器对后端服务器返回结果的压缩，这是默认设置<br/>**expired**:表示当后端服务器响应页头部包含用于指示响应数据过期时间的expired头域时，启用对响应数据的Gzip压缩
**no-cache**:表示当后端服务器响应页头部包含用于通知缓存机制是否缓存的Cache-Control头域，且其指令值为no-cache时，启用对响应数据的Gzip压缩
**no-store**:表示当后端服务器响应页头部包含用于通知缓存机制是否缓存的Cache-Control头域，且其指令值为no-store时，启用对响应数据的Gzip压缩
**private**:表示当后端服务器响应页头部包含用于通知缓存机制是否缓存的Cache-Control头域，且其指令值为private时，启用对响应数据的Gzip压缩
**no_last_modified**:表示当后端服务器响应页头部不包含用于指明获取数据最后修改时间的Last-Modified头域时，启动对响应数据的Gzip压缩
**no_etag**:表示当后端服务器响应页头部不包含用于标示被请求变量的实体值的ETag头域时，启动对响应数据的Gzip压缩
**auth**:表示当后端服务器响应页头部包含用于标示HTTP授权证书的Authorization头域时时，启动对响应数据的Gzip压缩
**any**:无条件启动对响应数据的Gzip压缩 |
| 根据响应页的MIME类型开启是否压缩<br/>gzip_types mime-type …; | 这个mime-type的默认值是text/html，实际上当gzip设置为on的时候，Nginx服务器会对所有的text/html类型的页面进行Gzip压缩，该变量还可以取值“*”，表示对所有的MIME类型页面进行压缩，一般我们如下设置：<br/>gzip_types text/plain application/x-javascript text/css text/html application/html; |
| 用与设置Gzip功能是否发送带有“Vary:Accept-Encoding”头域的响应头部<br/>gzip_vary on \| off | 该头域的主要功能主要是告诉接收方发送的数据时通过压缩处理，开启后的效果是在响应头部添加Accept-Encoding:gzip.这对于本身不止Gzip压缩的客户端浏览器是很有用的。<br/>默认是off的，当然我也可以通过nginx配置的add_header指令强制Nginx服务器在响应头部添加“Vary:Accept-Encoding”头域：
add_header Vary Accept-Encoding gzip; |



## 2.ngx_http_gzip_static_module模块 -（预压缩） 处理的指令

**使用前提**： 如果要使用这个指令，必须在Nginx程序配置时添加–with-http_gzip_static_module指令
    这个模块主要是负责搜索和发送Gzip功能预压缩的数据（这些数据是“.gz”作为后缀名存储在服务器上）。如果客户端请求的数据在之前被压缩过，而且客户端浏览器支持Gzip压缩，就直接返回压缩的数据。

    **和ngx_http_gzip_module模块不同点**： 该模块和ngx_http_gzip_module模块不同处在于该模块使用静态压缩，在HTTP响应头部包含Content-Length头域来指明报文体的长度，用于服务器可确定响应数据长度的情况而后者默认使用Chunked编码动态压缩，主要使用服务器无法确定响应数据长度的情况，比如大文件下载的情形，这时需要实时生成数据长度。

     **指令**： 和该模块的指令有gzip_static、gzip_http_version、gzip_proxied、gzip_disable和gzip_vary等。

```
gzip_static 指令用来开启和关闭该模块指令 语法结构：gzip_static on | off |always
on:开启该模块功能
off:关闭该模块功能
always：一直发送gzip预压缩文件，而不检查客户端浏览器是否支持Gzip压缩。

其他指令和上面的相同。需要注意的是gzip_proxied指令只接受如下设置： gzip_proxied expired |no-cache | no-store | private | auth  

另外对于该模块下的gzip_vary指令，开启以后只给未压缩的内容添加“Vary: Accept_Encoding”头域，而不是所有的内容都添加。
```



## 3.ngx_http_gunzip_module模块（解压）处理的指令

这个指令是在客户端浏览器不支持Gzip解压能力而在Nginx服务器在向其发送数据之前现将其解压，这些压缩数据可能来自于后端服务器压缩产生的或者Nginx服务器预压缩产生的。

**指令**： gunzip、gunzip_buffers、gzip_httpversion、gzip_proxied、gzip_disable 和gzip_vary等

| 语法                                                         | 说明                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| 用于开启或者关闭该模块<br>gunzip_static on \| off            | **on**:表示开启该模块的功能<br/>**off**:表示关闭该模块功能呢。
默认是关闭功能。如果客户端浏览器不支持Gzip处理，Nginx浏览器将返回解压后的数据，如果客户端浏览器支持Gzip功能，Nginx服务器忽略这个指令的设置，任然返回压缩数据 |
| 用于解压需要使用缓存空间的大小<br/>gunzip_bufffers number size | number:指定Nginx服务器需要向系统申请缓存空间的个数<br/>size：指定每个缓存空间的大小。<br/>默认值number*size=128k ,例如：gunzip_buffers 32 4k \|16 8k<br/>该指令用法和ngx_http_gzip_moduel模块的使用方法相同 |



## 4.案例

```
user nobody;
worker_processes 3;
error_log logs/error.log;
pid nginx.pid;
events{
  user epoll;
  worker_connections 1024;
}
http{
   include mime.types;
   default_type application/octet-stream;
   sendfile on;
   keepalive_timeout 65;
   log_format access.log '\$remote_addr-[\$time_local]-"\$request"-"\$http_user_agent"';
   gzip on;    #开启gzip压缩功能
   gzip_min_length 1024;          #最小压缩文件为1k
   gzip buffers 4 16k;                #压缩缓存空间大小
   gzip_comp_level 2;                 #压缩级别为2
   gzip_types text/plain appliaton/x-javascript text/css application/xml                  #压缩文件类型
   gzip vary on;               #开启压缩表示 
   gunzip_static on;         #开启（在浏览器不支持解压功能，nginx提前解压）解压功能
   server{
    ....
    gzip off;   &emsp; &emsp; #在某一个server不需要压缩，可以关闭压缩功能
   }
    server{
    ....
   }
}
```



## 5.常见问题

1.如果nginx和后端服务器都开启了gzip压缩功能会出现报错：第一次访问不会出现错误，但是刷新报错304，建议关闭后端服务器的gzip压缩功能

2.Nginx服务器作为后端服务器和前端服务器加护，两类服务器对Gzip压缩功能支持不同也会导致问题。这个和Context-Length还有Chunked编码有关，建议查看有关书籍。

