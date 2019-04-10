## Nginx反向代理-如何在web应用中获取用户ip

**问题背景：**

在实际应用中，我们可能需要获取用户的ip地址，比如做异地登陆的判断，或者统计ip访问次数等，通常情况下我们使用request.getRemoteAddr()就可以获取到客户端ip，但是当我们使用了nginx作为反向代理后，使用request.getRemoteAddr()获取到的就一直是nginx服务器的ip的地址，那这时应该怎么办？

## **part1：解决方案**

我在查阅资料时，有一本名叫《实战nginx》的书，作者张晏，这本书上有这么一段话“经过反向代理后，由于在客户端和web服务器之间增加了中间层，因此web服务器无法直接拿到客户端的ip，通过$remote_addr变量拿到的将是反向代理服务器的ip地址”。这句话的意思是说，当你使用了nginx反向服务器后，在web端使用request.getRemoteAddr()（本质上就是获取$remote_addr），取得的是nginx的地址，即$remote_addr变量中封装的是nginx的地址，当然是没法获得用户的真实ip的，但是，nginx是可以获得用户的真实ip的，也就是说nginx使用$remote_addr变量时获得的是用户的真实ip，如果我们想要在web端获得用户的真实ip，就必须在nginx这里作一个赋值操作，如下：

proxy_set_header            X-real-ip $remote_addr;

其中这个X-real-ip是一个自定义的变量名，名字可以随意取，这样做完之后，用户的真实ip就被放在X-real-ip这个变量里了，然后，在web端可以这样获取：

request.getAttribute("X-real-ip")

这样就明白了吧。



## **part2：原理介绍**

这里我们将nginx里的相关变量解释一下，通常我们会看到有这样一些配置

server {

​        listen       88;

​        server_name  localhost;

​        \#charset koi8-r;

​        \#access_log  logs/host.access.log  main;

​        location /{

​            root   html;

​            index  index.html index.htm;

​                            proxy_pass                  http://backend; 

​           proxy_redirect              off;

​           proxy_set_header            Host $host;

​           proxy_set_header            X-real-ip $remote_addr;

​           proxy_set_header            X-Forwarded-For $proxy_add_x_forwarded_for;

​                     \# proxy_set_header            X-Forwarded-For $http_x_forwarded_for;

​        }

我们来一条条的看

**1. proxy_set_header    X-real-ip $remote_addr;**

这句话之前已经解释过，有了这句就可以在web服务器端获得用户的真实ip

但是，实际上要获得用户的真实ip，不是只有这一个方法，下面我们继续看。

**2.  proxy_set_header            X-Forwarded-For $proxy_add_x_forwarded_for;**

我们先看看这里有个X-Forwarded-For变量，这是一个squid开发的，用于识别通过HTTP代理或负载平衡器原始IP一个连接到Web服务器的客户机地址的非rfc标准，如果有做X-Forwarded-For设置的话,每次经过proxy转发都会有记录,格式就是client1, proxy1, proxy2,以逗号隔开各个地址，由于他是非rfc标准，所以默认是没有的，需要强制添加，在默认情况下经过proxy转发的请求，在后端看来远程地址都是proxy端的ip 。也就是说在默认情况下我们使用request.getAttribute("X-Forwarded-For")获取不到用户的ip，如果我们想要通过这个变量获得用户的ip，我们需要自己在nginx添加如下配置：

**proxy_set_header            X-Forwarded-For $proxy_add_x_forwarded_for;**

意思是增加一个$proxy_add_x_forwarded_for到X-Forwarded-For里去，注意是增加，而不是覆盖，当然由于默认的X-Forwarded-For值是空的，所以我们总感觉X-Forwarded-For的值就等于$proxy_add_x_forwarded_for的值，实际上当你搭建两台nginx在不同的ip上，并且都使用了这段配置，那你会发现在web服务器端通过request.getAttribute("X-Forwarded-For")获得的将会是客户端ip和第一台nginx的ip。

**那么$proxy_add_x_forwarded_for又是什么？**

$proxy_add_x_forwarded_for变量包含客户端请求头中的"X-Forwarded-For"，与$remote_addr两部分，他们之间用逗号分开。

举个例子，有一个web应用，在它之前通过了两个nginx转发，www.linuxidc.com 即用户访问该web通过两台nginx。

在第一台nginx中,使用

proxy_set_header            X-Forwarded-For $proxy_add_x_forwarded_for;

现在的$proxy_add_x_forwarded_for变量的"X-Forwarded-For"部分是空的，所以只有$remote_addr，而$remote_addr的值是用户的ip，于是赋值以后，X-Forwarded-For变量的值就是用户的真实的ip地址了。

到了第二台nginx，使用

proxy_set_header            X-Forwarded-For $proxy_add_x_forwarded_for;

现在的$proxy_add_x_forwarded_for变量，X-Forwarded-For部分包含的是用户的真实ip，$remote_addr部分的值是上一台nginx的ip地址，于是通过这个赋值以后现在的X-Forwarded-For的值就变成了“用户的真实ip，第一台nginx的ip”，这样就清楚了吧。

最后我们看到还有一个$http_x_forwarded_for变量，这个变量就是X-Forwarded-For，由于之前我们说了，默认的这个X-Forwarded-For是为空的，所以当我们直接使用proxy_set_header            X-Forwarded-For $http_x_forwarded_for时会发现，web服务器端使用request.getAttribute("X-Forwarded-For")获得的值是null。如果想要通过request.getAttribute("X-Forwarded-For")获得用户ip，就必须先使用proxy_set_header            X-Forwarded-For $proxy_add_x_forwarded_for;这样就可以获得用户真实ip。

