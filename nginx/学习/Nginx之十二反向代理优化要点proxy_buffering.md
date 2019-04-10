# Nginx之十二反向代理优化要点proxy_buffering

当nginx用于反向代理时，每个客户端将使用两个连接：一个用于响应客户端的请求，另一个用于到后端的访问；

那么，可以从如下配置起步:

```
# One worker per CPU-core.
worker_processes  2;
events {
    worker_connections  8096;
    multi_accept        on;
    use                 epoll;
}
worker_rlimit_nofile 40000;
http {
    sendfile           on;
    tcp_nopush         on;
    tcp_nodelay        on;
    keepalive_timeout  15;
}
```

## 标准的代理配置

下面是一个基本的反向代理配置模板，将所有请求都转发给指定的后端应用。

例如，到[http://your.ip:80/](http://your.ip/)的请求都将重定向到 <http://127.0.0.1:4433/> 私有服务器:

```
# One process for each CPU-Core
worker_processes  2;
# Event handler.
events {
    worker_connections  8096;
    multi_accept        on;
    use                 epoll;
}
http {
     # Basic reverse proxy server
     upstream backend  {
           server 127.0.0.1:4433;
     }
     # *:80 -> 127.0.0.1:4433
     server {
            listen       80;
            server_name  example.com;
            ## send all traffic to the back-end
            location / {
                 proxy_pass        http://backend;
                 proxy_redirect    off;
                 proxy_set_header  X-Forwarded-For $remote_addr;
            }
     }
}
```



## 缓冲控制

如果禁止缓冲，那么当Nginx一收到后端的反馈就同时传给客户端。

`nginx` 不会从被代理的服务器读取整个反馈信息。

nginx可从服务器一次接收的最大数据大小由  proxy_buffer_size 控制。

```
proxy_buffering    off;
proxy_buffer_size  128k;
proxy_buffers 100  128k;
```

**相关参数**
**proxy_buffer_size**
语法: proxy_buffer_size the_size
默认值: proxy_buffer_size 4k/8k
上下文: http, server, location

该指令设置缓冲区大小,从代理后端服务器取得的第一部分的响应内容,会放到这里.小的响应header通常位于这部分响应内容里边.
默认来说,该缓冲区大小等于指令 proxy_buffers所设置的;但是,你可以把它设置得更小。



**proxy_buffering**
语法: proxy_buffering on|off
默认值: proxy_buffering on
上下文: http, server, location

这个参数用来控制是否打开后端响应内容的缓冲区，如果这个设置为off，那么proxy_buffers和proxy_busy_buffers_size这两个指令将会失效。 但是无论proxy_buffering是否开启，对proxy_buffer_size都是生效的。

proxy_buffering开启的情况下，nignx会把后端返回的内容先放到缓冲区当中，然后再返回给客户端(边收边传，不是全部接收完再传给客户端)。 临时文件由proxy_max_temp_file_size和proxy_temp_file_write_size这两个指令决定的。如果响应内容无法放在内存里边,那么部分内容会被写到磁盘上。

如果proxy_buffering关闭，那么nginx会立即把从后端收到的响应内容传送给客户端，每次取的大小为proxy_buffer_size的大小，这样效率肯定会比较低。
nginx不尝试计算被代理服务器整个响应内容的大小,nginx能从服务器接受的最大数据,是由指令proxy_buffer_size指定的。

注：1、 proxy_buffering启用时，要提防使用的代理缓冲区太大。这可能会吃掉你的内存，限制代理能够支持的最大并发连接数。

2、对于基于长轮询(long-polling)的Comet 应用来说,关闭 proxy_buffering 是重要的,不然异步响应将被缓存导致Comet无法工作。



**proxy_buffers**
语法: proxy_buffers the_number is_size;
默认值: proxy_buffers 8 4k/8k;
上下文: http, server, location

该指令设置缓冲区的大小和数量,从被代理的后端服务器取得的响应内容,会放置到这里. 默认情况下,一个缓冲区的大小等于内存页面大小,可能是4K也可能是8K,这取决于平台。
 proxy_buffers由缓冲区数量和缓冲区大小组成的。总的大小为number*size。
若某些请求的响应过大,则超过_buffers的部分将被缓冲到硬盘(缓冲目录由_temp_path指令指定), 当然这将会使读取响应的速度减慢, 影响用户体验. 可以使用proxy_max_temp_file_size指令关闭磁盘缓冲。



**proxy_busy_buffers_size**
语法: proxy_busy_buffers_size size;
默认值: proxy_busy_buffers_size proxy_buffer_size * 2;
上下文: http, server, location, if


proxy_busy_buffers_size不是独立的空间，他是proxy_buffers和proxy_buffer_size的一部分。nginx会在没有完全读完后端响应的时候就开始向客户端传送数据，所以它会划出一部分缓冲区来专门向客户端传送数据(这部分的大小是由proxy_busy_buffers_size来控制的，建议为proxy_buffers中单个缓冲区大小的2倍)，然后它继续从后端取数据，缓冲区满了之后就写到磁盘的临时文件中。



**proxy_max_temp_file_size**和**proxy_temp_file_write_size**

临时文件由proxy_max_temp_file_size和proxy_temp_file_write_size这两个指令决定。 proxy_temp_file_write_size是一次访问能写入的临时文件的大小，默认是proxy_buffer_size和proxy_buffers中设置的缓冲区大小的2倍，Linux下一般是8k。
proxy_max_temp_file_size指定当响应内容大于proxy_buffers指定的缓冲区时, 写入硬盘的临时文件的大小. 如果超过了这个值, Nginx将与Proxy服务器同步的传递内容, 而不再缓冲到硬盘. 设置为0时, 则直接关闭硬盘缓冲.



**总结：**

**buffer工作原理**
首先第一个概念是所有的这些proxy buffer参数是作用到每一个请求的。每一个请求会安按照参数的配置获得自己的buffer。proxy buffer不是global而是per request的。

proxy_buffering 是为了开启response buffering of the proxied server，开启后proxy_buffers和proxy_busy_buffers_size参数才会起作用。

无论proxy_buffering是否开启，proxy_buffer_size（main buffer）都是工作的，proxy_buffer_size所设置的buffer_size的作用是用来存储upstream端response的header。

在proxy_buffering 开启的情况下，Nginx将会尽可能的读取所有的upstream端传输的数据到buffer，直到proxy_buffers设置的所有buffer们被写满或者数据被读取完(EOF)。此时nginx开始向客户端传输数据，会同时传输这一整串buffer们。同时如果response的内容很大的话，Nginx会接收并把他们写入到temp_file里去。大小由proxy_max_temp_file_size控制。如果busy的buffer传输完了会从temp_file里面接着读数据，直到传输完毕。

一旦proxy_buffers设置的buffer被写入，直到buffer里面的数据被完整的传输完（传输到客户端），这个buffer将会一直处在busy状态，我们不能对这个buffer进行任何别的操作。所有处在busy状态的buffer size加起来不能超过proxy_busy_buffers_size，所以proxy_busy_buffers_size是用来控制同时传输到客户端的buffer数量的。



![img](http://img.blog.csdn.net/20170804104819020?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY3ltbV9saXU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)



## 缓存和过期控制

上面的配置是将所有请求都转发给后端应用。为避免静态请求给后端应用带来的过大负载，我们可以将nginx配置为缓存那些不变的响应数据。

这就意味着nginx不会向后端转发那些请求。

下面示例，将 `*.html`, `*.gif`, 等文件缓存30分钟。:

```
http {
     #
     # The path we'll cache to.
     #
     proxy_cache_path /tmp/cache levels=1:2 keys_zone=cache:60m max_size=1G;
}
## send all traffic to the back-end
location / {
    proxy_pass  http://backend;
    proxy_redirect off;
    proxy_set_header        X-Forwarded-For $remote_addr;
    location ~* \.(html|css|jpg|gif|ico|js)$ {
        proxy_cache          cache;
        proxy_cache_key      $host$uri$is_args$args;
        proxy_cache_valid    200 301 302 30m;
        expires              30m;
        proxy_pass  http://backend;
	}
}
```

这里，我们将请求缓存到 `/tmp/cache，并定义了其大小限制为1G。同时只允许缓存有效的返回数据，例如`:

```
proxy_cache_valid  200 301 302 30m;
```

所有响应信息的返回代码不是 "`HTTP (200|301|302) OK`" 的都不会被缓存。

对于例如workpress的应用，需要处理cookies 和缓存的过期时间，通过只缓存静态资源来避免其带来的问题。