Windows安装ELK

**1.****背景**

日志主要包括系统日志、应用程序日志和安全日志。系统运维和开发人员可以通过日志了解服务器软硬件信息、检查配置过程中的错误及错误发生的原因。经常分析日志可以了解服务器的负荷，性能安全性，从而及时采取措施纠正错误。

通常，日志被分散的储存不同的设备上。如果需要管理数十上百台服务器，必须依次登录每台机器的传统方法查阅日志，这样很繁琐和效率低下。当务之急是使用集中化的日志管理，开源实时日志分析ELK平台能够完美的解决上述所提到的问题。

**2.****工具**

ELK由ElasticSearch（ES）、Logstash和Kiabana三个开源工具组成。

ES是个开源分布式搜索引擎，它的特点有：分布式，零配置，自动发现，索引自动分片，索引副本机制，restful风格接口，多数据源，自动搜索负载等。

Logstash是一个完全开源的工具，可以对日志进行收集、分析、并将其存储供以后使用。

kibana也是一个开源和免费的工具，他Kibana可以为Logstash和ES提供的日志分析友好的Web界面，可以帮助您汇总、分析和搜索重要数据日志。

3.安装过程

3.1首先要保证 Windows 已经装好JDK ，并且配置好环境变量，这个就不多说了，应该大多都会配置

3.2下载 elasticsearch logstash ,kibana  下载地址：<https://www.elastic.co/downloads> 



3.3下载完成后分别解压 （windows 一般下载 ZIP 包）



3.4启动 elasticsearch  kibana logstash 方式 很简单   

方式1：

分别进入各自的bin 目录 双击 elasticsearch.bat  kibana.bat 即可运行

logstash 稍微复杂些，需要编写 logstash.conf  然后执行命令：

cmd 进入 bin 目录 执行命令

```java
  logstash.bat -f  logstash.conf 
```

 方式2 将三者注册成windows 服务 用windows服务的方式来启动

 

 

先配置elasticsearch 服务：

 

cd到elasticsearch文件夹的bin目录下  

cmd 运行 elasticsearch-service install，会提示安装成功

cmd 运行 elasticsearch-service manager 会弹出服务管理界面，可以设置自动启动，并启动之。

浏览器访问 127.0.0.1:9200 ，出现成功的json

 

 

 

 配置logstash ,cd 到logstash文件夹的下bin目录

创建配置文件 logstash.conf  ,内容如下：

