# Nginx之九Upstream后端服务器组

后端服务器组的配置

upstream指令是设置后端服务器组的主要指令,如下所示，都是设置在upstream花括号内

| 指令                         | 说明                                                         |
| ---------------------------- | ------------------------------------------------------------ |
| upstream name {…}            | upstream指令是设置后端服务器组的主要指令,请求按照轮叫调度(Round-Robin, RR)策略顺序选择服务器处理,<br/>**name**是给后端服务器组起的组名
**花括号** 是包含其他的服务器 |
| server address [parameters]; | server指令用于设置组内的服务器:<br/>**address**： 服务器地址，可以包括端口号或者以”unix:”为前缀的进程间通信的Unix Domain Socket<br/>
**params**： 为当前服务器配置更多属性，如下：
1）weight=number，组内服务器权重，权重高的优先处理请求（采用加权轮询策略），组内所有的权重默认设置为1.
2）max_fails=number，设置一个请求失败的次数，在一定时间范围内，当对组内服务器请求失败次数超过该变量时，认为该服务器无效（404除外）,请求失败的各种情况和proxy_next_upstream指令的配置相匹配，默认值是1，如果设置为0，则不使用上面的办法检查服务器是否有效
3）fail_timeout=time，作用两个，一是设置max_fails指令尝试请求组内服务器的时间（上面的一定时间）另一个作用是检查服务器是否有效，如果被认为是无效，这个这个时间是被认为无效的时间，而不是永久无效，默认值是10s 
4）backup，将服务器标记为备用服务器,当正常服务器处于无效状态或者繁忙状态时才能被用来处理请求。 
5）down，标记服务器永久失效 |
| ip_hash;                     | ip_hash指令用于实现会话保持功能，将某个客户端的多次请求定向到组内同一台服务器上，保证客户端与服务器之间建立稳定的会话。<br/>这个ip_hash技术主要对客户端的IP地址做hash分配，这样每个访客固定访问一个后端服务器，可以解决session的问题,每次访问由于自己本地的ip不变，所以访问的服务器也不容易变。 
注：ip_hash指令不能与weight变量一起使用，在整个系统中，Nginx服务器必须处于最前端的服务器，而且客户端地址必须为C类地址 |
| keepalive connections;       | keepalive指令用于控制网络连接保持功能 设置服务器的每一个工作进程允许该服务器组保持的空闲网络连接数的上限值 |
| least_conn;                  | least_conn指令用于配置Nginx服务器使用负载均衡策略为网络连接分配服务器组内的服务器，将请求分配给当前网络连接最少的服务器 |

```
以下表示三种方式访问后端服务器
例一（轮询）：
upstream backend {   #这里没有使用其他参数，表示通过轮询的方式访问下面的服务器
	server www.viss.com;
	server www.vison.com;
}
例二（权重）：
upstream backend1{
	server www.vison.com weight=5;         #权重最大优先访问，其他默认都是1
	server 127.0.0.1:8080 max_fails=3 fail_timout=30s;   #如果在30s内连续三次产生3次失败，就认为这个服务器在之后的30s是无效(down)状态
	server www.ss.com      backup;   #表示这个是一个备用服务器
}
例三（ip_hash）:
upstream backend2 {
	ip_hash;          #表示通过客户端的ip做hash访问下面这两个服务器
	server www.vison.com;
	server www.ws.com;
}
```

