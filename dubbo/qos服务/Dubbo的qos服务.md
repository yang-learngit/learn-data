# Dubbo的qos服务

QOS=Quality of Service，不同于网络里的QOS，Dubbo里QOS用于动态控制服务，以及查询

新版本Dubbo里，QOS服务可以通过一些命令来返回响应的结果，达到动态控制的目的

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



根据错误日志可以找到报异常的代码位置：dubbo的com.alibaba.dubbo.qos.server的Server类中打印出来,查看Server类源码发现在方法start中调用了netty的初始化方法,并将port作为被外部访问的qos-server端口,代码如下（注：dubbo使用版本是2.6.0）



跟进dubbo源码发现，