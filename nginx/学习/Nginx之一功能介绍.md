# Nginx之一 功能介绍

Nginx提供的基本服务大体分为三类：基本Http服务，高级Http服务和邮件服务。

- 基本Http服务：可以作为Http代理服务器和反向代理服务器，支持通过缓存加速访问，完成简单的负载均衡和熔池，包括过滤功能，支持SSL等
- Nginx提供高级Http服务，可以自定义配置，支持虚拟主机，支持URL重定向，支持网路监控，支持流媒体传输。
- Nginx可以作为邮件代理服务器，支持IMAP/POP3代理服务功能，支持内部SMTP代理服务功能

## 1、基本Http服务

​    这个服务包含以下功能特性：

 1）处理静态文件(如HTML静态网页的请求)；处理索引文件以及支持自动索引

2）打开并自行管理文件描述符缓存

3）提供反向代理服务，并且可以使用缓存加速反向代理，同时完成简单负载均衡和容错

4）提供远程FastCGI服务的缓存机制，加速访问，同时完成简单的负载均衡和容错

5）使用Nginx的模块化特性提供过滤器功能。Nginx基本过滤器包含gzip压缩、rangs支持、chunked响应、XSLT、SSI以及图像缩放等。其中，针对包含说个SSI的页面，经过FastCGI或反向代理，SSI过滤器可以并行处理

6）支持HTTP下的安全套接层安全协议SSL

 

## 2、高级Http服务

这个服务包含以下功能特性：

1）支持基于名字和IP的虚拟机的设置

2）支持HTTP/1.0中的KEEP-Alive的模式和管线（PipeLined）模型连接

3）支持重新加载配置以及在线升级时，无须中断正在处理的请求

4）自定义访问日志格式、带缓存的日志写操作以及快速日志轮转

5）提供3xx-5xx 错误代码重定向功能

6）支持重写（Rewrite）模块扩展

7）支持HTTP DAV模块，从而为HttpWebDAV提供PUT、DELETE、MKCOL、COPY以及MOVE方法

8）支持FLV流和MP4流传输

9）支持网络监控，基于客户端IP地址和Http基本认证机制的访问控制、速度限制、来自同一地址的同时连接数或请求数限制等

10）支持嵌入Perl语言

 

## 3、邮件代理服务

这个服务包含以下功能特性：

1）支持使用外部HTTP认证服务器重定向到IMAP/POP3后端，并支持IMAP认证方式（LOGIN、AUTH LOGIN/PLAIN/CRAM-MD5）和POP3认证方式（USER/PASS、APOP、AUTH LOGIN/PLAIN/CRAM-MD5）

2）支持使用外部HTTP服务器认证用户后重定向连接到内部SMTP后端，并支持SMTP认证方式（AUTH LOGIN/PLAIN/CRAM-MD5）

3）支持邮件代理下的安全套接层协议SSL

4）支持纯文本协议的扩展协议STARTTLS

 

## 4、常用功能

常用：HTTP代理、反向代理、负载均衡、Web缓存。

 

**注意点：**

Web缓存服务主要是有Proxy_Cache相关指令集和FastCGI_Cache指令集构成。Proxy_Cache主要用于在Nginx服务器提供反向代理服务时，对后端源服务器的返回内容进行URL缓存；FastCGI_Cache主要用于对FastCGI的动态程序记性缓存。另外还有一款常用的第三方模块ngx_cache_purge也很常用，用来清除Nginx服务器上指定URL缓存。