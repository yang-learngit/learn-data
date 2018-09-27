



# Windows安装Redis

## **一、下载windows版本的Redis**

去官网找了很久，发现原来在官网上可以下载的windows版本的，现在官网以及没有下载地址，只能在github上下载，官网只提供linux版本的下载

官网下载地址：<http://redis.io/download>

github下载地址：<https://github.com/MSOpenTech/redis/tags>

## **二、安装Redis**

1.这里下载的是Redis-x64-3.2.100版本，我的电脑是win7 64位，所以下载64位版本的，在运行中输入cmd，然后把目录指向解压的Redis目录。

![](https://raw.githubusercontent.com/yang-zhijiang/learn-data/master/redis/%E5%AE%89%E8%A3%85/reids%E8%A7%A3%E5%8E%8B%E7%9B%AE%E5%BD%95.png)

2、启动命令

redis-server redis.windows.conf，出现下图显示表示启动成功了。

![](https://raw.githubusercontent.com/yang-zhijiang/learn-data/master/redis/安装/redis启动.png)

## **三、设置Redis服务**

1、由于上面虽然启动了redis，但是只要一关闭cmd窗口，redis就会消失。所以要把redis设置成windows下的服务。

也就是设置到这里，首先发现是没用这个Redis服务的。

![](https://raw.githubusercontent.com/yang-zhijiang/learn-data/master/redis/安装/查看redis服务.png)

2、设置服务命令

redis-server --service-install redis.windows-service.conf --loglevel verbose

![](https://raw.githubusercontent.com/yang-zhijiang/learn-data/master/redis/安装/设置redis服务命名.png)

输入命令之后没有报错，表示成功了，刷新服务，会看到多了一个redis服务。

![](https://raw.githubusercontent.com/yang-zhijiang/learn-data/master/redis/安装/查看redis服务存在.png)

3、常用的redis服务命令。

卸载服务：redis-server --service-uninstall

开启服务：redis-server --service-start

停止服务：redis-server --service-stop

4、启动服务

![](https://raw.githubusercontent.com/yang-zhijiang/learn-data/master/redis/安装/启动redis服务.png)

5、测试Redis

![](https://raw.githubusercontent.com/yang-zhijiang/learn-data/master/redis/安装/测试redis服务.png)

安装测试成功。