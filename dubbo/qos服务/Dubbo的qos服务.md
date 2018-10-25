# Dubbo的qos服务

QOS=Quality of Service，不同于网络里的QOS，Dubbo里QOS用于动态控制服务，以及查询

新版本Dubbo里，QOS服务可以通过一些命令来返回响应的结果，达到动态控制的目的

dubbo版本2.5.9

### 端口

新版本的 telnet 端口 与 dubbo 协议的端口是不同的端口，默认为 `22222`，可通过配置文件`dubbo.properties` 修改:

```
dubbo.application.qos.port=33333
```

或者通过设置 JVM 参数:

```
-Ddubbo.application.qos.port=33333
```

### 安全

默认情况下，dubbo 接收任何主机发起的命令，可通过配置文件`dubbo.properties` 修改:

```
dubbo.application.qos.accept.foreign.ip=false
```

或者通过设置 JVM 参数:

```
-Ddubbo.application.qos.accept.foreign.ip=false
```

拒绝远端主机发出的命令，只允许服务本机执行

### 开启

默认情况下，dubbo的qos服务是开启的，可通过配置文件`dubbo.properties` 修改:

```
dubbo.application.qos.enable=false
```

或者通过设置 JVM 参数:

```
-Ddubbo.application.qos.enable=false
```

禁用qos服务



### 问题

在本地同一台机器上开了两个运行实例，一个是dubbo provider实例，一个是dubbo consumer实例。首先启动provider实例没有任务问题，当启动consumer以后，控制台却抛出如下问题

![](https://raw.githubusercontent.com/yang-zhijiang/learn-data/master/dubbo/qos%E6%9C%8D%E5%8A%A1/%E7%AB%AF%E5%8F%A3%E5%86%B2%E7%AA%81.png)

根据错误日志可以找到报异常的代码位置：dubbo的com.alibaba.dubbo.qos.server的Server类中打印出来,查看Server类源码发现在方法start中调用了netty的初始化方法,并将port作为被外部访问的qos-server端口,代码如下（注：dubbo使用版本是2.6.0）

![](https://raw.githubusercontent.com/yang-zhijiang/learn-data/master/dubbo/qos%E6%9C%8D%E5%8A%A1/%E6%8A%9B%E5%BC%82%E5%B8%B8%E4%BB%A3%E7%A0%81.png)

跟进dubbo源码发现，端口的获取方式是System.getProperty(key)，获取系统的配置文件内容，代码如下：

![](https://raw.githubusercontent.com/yang-zhijiang/learn-data/master/dubbo/qos%E6%9C%8D%E5%8A%A1/%E8%8E%B7%E5%8F%96%E9%85%8D%E7%BD%AE%E4%BB%A3%E7%A0%81.png)

因此，Dubbo2.6.0版本不能在配置文件配置qos的参数，需要设置JVM的参数配置，且2.6.0版本固定开启qos服务，不能通过配置文件关闭qos服务

```
System.getProperty(key)
```

相关内容可以参考：

https://blog.csdn.net/kongqz/article/details/3987198

https://www.cnblogs.com/acm-bingzi/p/6673823.html

比较其他版本Dubbo的端口的定义方式，代码如下：

![](https://raw.githubusercontent.com/yang-zhijiang/learn-data/master/dubbo/qos%E6%9C%8D%E5%8A%A1/2.6.1.png)

dubbo的配置参数不同版本不同：

2.5.9版本：

```
dubbo.application.qos.enable=true
dubbo.application.qos.port=33333
dubbo.application.qos.accept.foreign.ip=true
```

2.6.0版本

```
-Ddubbo.qos.port=33333
-Ddubbo.qos.accept.foreign.ip=true
```

2.6.1以上版本：

```
dubbo.application.qosEnable=true
dubbo.application.qosPort=33333
dubbo.application.qosAcceptForeignIp=true
```

**注意：**

2.6.0版本默认启用qos服务且不能通过配置文件关闭qos服务

![](https://raw.githubusercontent.com/yang-zhijiang/learn-data/master/dubbo/qos%E6%9C%8D%E5%8A%A1/2.6.0%E4%B8%8D%E8%AF%BB%E5%90%AF%E7%94%A8%E9%85%8D%E7%BD%AE.jpg)

2.6.0版本默认启用qos服务可以通过配置文件关闭qos服务

![](https://raw.githubusercontent.com/yang-zhijiang/learn-data/master/dubbo/qos%E6%9C%8D%E5%8A%A1/2.6.1%E8%AF%BB%E5%90%AF%E7%94%A8%E9%85%8D%E7%BD%AE.png)





参考文档：

https://blog.csdn.net/u013202238/article/details/81432784

http://lihuia.com/2018/10/17/dubbo-qos%E6%9C%8D%E5%8A%A1/

