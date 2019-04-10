# Nginx之六：服务器的优化配置

## 一、sysctl.conf针对IPv4内核的7个参数的配置优化

**这里涉及到的参数都是和IPv4网络相关的内核参数，我们可以将这些内核参数的值追加到Linux系统的/etc/sysctl.conf文件中。**

```
然后使用这个命令让它生效
#/sbin/sysctl -p
```

1、net.core.netdev_max_backlog #每个网络接口的处理速率比内核处理包的速度快的时候，允许发送队列的最大数目。

```
[root@Server1 nginx]# sysctl -a | grep max_backlog
net.core.netdev_max_backlog = 1000 这里默认是1000，可以设置的大一些，比如：
net.core.netdev_max_backlog = 102400
```

2、net.core.somaxconn: #用于调节系统同时发起的TCP连接数，默认值一般为128，在客户端存在高并发请求的时候，128就变得比较小了，可能会导致链接超时或者重传问题。

```
net.core.somaxconn = 128 #默认为128，高并发的情况时候要设置大一些，比如：
net.core.somaxconn = 102400
```

3、net.ipv4.tcp_max_orphans:设置系统中做多允许多少TCP套接字不被关联到任何一个用户文件句柄上，如果超出这个值，没有与用户文件句柄关联的TCP套接字将立即被复位，同时给出警告信息，这个值是简单防止DOS（Denial of service）的攻击，在内存比较充足的时候可以设置大一些：

```
net.ipv4.tcp_max_orphans = 32768 #默认为32768，可以改大一些：
net.ipv4.tcp_max_orphans = 102400
```

4、net.ipv4.tcp_max_syn_backlog #用于记录尚未收到客户度确认消息的连接请求的最大值，一般要设置大一些：

```
net.ipv4.tcp_max_syn_backlog = 256 #默认为256，设置大一些如下：
net.ipv4.tcp_max_syn_backlog = 102400
```

5、net.ipv4.tcp_timestamps #用于设置时间戳，可以避免序列号的卷绕，有时候会出现数据包用之前的序列号的情况，此值默认为1表示不允许序列号的数据包，对于Nginx服务器来说，要改为0禁用对于TCP时间戳的支持，这样TCP协议会让内核接受这种数据包，从而避免网络异常，如下：

```
net.ipv4.tcp_timestamps = 1 #默认为1，改为0，如下：
net.ipv4.tcp_timestamps = 0
```

6、net.ipv4.tcp_synack_retries #用于设置内核放弃TCP连接之前向客户端发生SYN+ACK包的数量，网络连接建立需要三次握手，客户端首先向服务器发生一个连接请求，服务器收到后由内核回复一个SYN+ACK的报文，这个值不能设置过多，会影响服务器的性能，还会引起syn攻击：

```
net.ipv4.tcp_synack_retries = 5 #默认为5，可以改为1避免syn攻击
net.ipv4.tcp_synack_retries = 1
```

7、net.ipv4.tcp_syn_retries #与上一个功能类似，设置为1即可：

```
net.ipv4.tcp_syn_retries = 5 #默认为5，可以改为1
net.ipv4.tcp_syn_retries = 1
```



## 二、nginx.conf配置文件针对CPU的2个优化参数

这个配置在nginx.conf的全局块中

```
1、woker_process #设置Nginx 启动多少个工作进程的数量 和CPU数量相同即可或者两倍
2、woker_cpu_affinit #设置 Nginx 工作进程所运行的CPU工作内核

如果机器CPU内核是四核，并且woker_process设置的是4 则设置如下：
          woker_cpu_affinit 0001 0100 1000 0010;
如果机器CPU内核是四核，并且woker_process设置的是8 则设置如下：
         woker_cpu_affinit 0001 0100 1000 0010 0001 0100 1000 0010;            
如果机器CPU内核是八核，并且woker_process设置的是8 则设置如下：
         woker_cpu_affinit 00000001 00000010 00000100 00001000 00010000       00100000 01000000 10000000;
```



## 三、nginx.conf配置文件中与网络相关的4个指令

```
1、keepalived_timeout 60 50； #设置Nginx服务器与客户端保持连接的时间是60秒，到60秒后服务器与客户端断开连接，50s是使用Keep-Alive消息头与部分浏览器如 chrome等的连接事件，到50秒后浏览器主动与服务器断开连接。 　
　 keepalived_timeout 60 50;
2、sendtime_out 10s #Http核心模块指令，指定了发送给客户端应答后的超时时间，Timeout是指没有进入完整established状态，只完成了两次握手，如果超过这个时间客户端没有任何响应，nginx将关闭与客户端的连接。
　　sendtime_out 10s;
3、client_header_timeout #用于指定来自客户端请求头的headerbuffer大小，对于大多数请求，1kb的缓冲区大小已经足够，如果自定义了消息头部或有更大的cookie，可以增加缓冲区大小。
　　client_header_timeout 4k;
4、multi_accept #设置是否允许，Nginx在已经得到一个新连接的通知时，接收尽可能更多的连接。
   multi_accept on;
```



## 四、nginx.conf配置文件中与驱动模型相关的8个指令

```
1、use； #用于指定Nginx 使用的事件驱动模型

2、woker_process； #指定Nginx启动的工作进程的数量

3、woker_connections 65535； #指定Nginx 每个工作进程的最大连接数，woker_connections * woker_process即为Nginx的最大连接数量。

4、woker_rlimit_sigpending 65535 #Nginx每个进程的事件信号队列的上限长度，如果超出长度，Nginx则使用poll模型处理客户的请求。

5、devpoll_changes 和 devpoll_events #用于设置Nginx 在/dev/poll 模型下Nginx服务器可以与内核之间传递事件的数量，前一个设置传递给内核的事件数量，后一个设置从内核读取的事件数量，默认为512。

6、kqueue_changes 和 kqueue_events #设置在kqueue模型下Nginx服务器可以与内核之间传递事件的数量，前一个设置传递给内核的事件数量，后一个设置从内核读取的事件数量，默认为512。

7、epoll_events #设置在epoll驱动模式下Nginx 服务器可以与内核之间传递事件的数量，默认为512。

8、rtsig_signo #设置Nginx在rtsig 模式使用的两个信号中的第一个，第二个信号是在第一个信号的编号上加1。

9、rtsig_overflow #这些参数指定如何处理rtsig队列溢出。当溢出发生在nginx清空rtsig队列时，它们将连续调用poll()和 rtsig.poll()来处理未完成的事件，直到rtsig被排空以防止新的溢出，当溢出处理完毕，nginx再次启用rtsig模式，rtsig_overflow_events
specifies指定经过poll()的事件数，默认为16，rtsig_overflow_test指定poll()处理多少事件后nginx将排空rtsig队列，默认值为32，rtsig_overflow_threshold只能运行在Linux
2.4.x内核下，在排空rtsig队列前nginx检查内核以确定队列是怎样被填满的。默认值为1/10，“rtsig_overflow_threshold
3”意为1/3。
```





