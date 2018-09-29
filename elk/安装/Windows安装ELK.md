# Windows安装ELK

## **1.**背景

日志主要包括系统日志、应用程序日志和安全日志。系统运维和开发人员可以通过日志了解服务器软硬件信息、检查配置过程中的错误及错误发生的原因。经常分析日志可以了解服务器的负荷，性能安全性，从而及时采取措施纠正错误。

通常，日志被分散的储存不同的设备上。如果需要管理数十上百台服务器，必须依次登录每台机器的传统方法查阅日志，这样很繁琐和效率低下。当务之急是使用集中化的日志管理，开源实时日志分析ELK平台能够完美的解决上述所提到的问题。



## **2.**工具

ELK由ElasticSearch（ES）、Logstash和Kiabana三个开源工具组成。

ES是个开源分布式搜索引擎，它的特点有：分布式，零配置，自动发现，索引自动分片，索引副本机制，restful风格接口，多数据源，自动搜索负载等。

Logstash是一个完全开源的工具，可以对日志进行收集、分析、并将其存储供以后使用。

kibana也是一个开源和免费的工具，他Kibana可以为Logstash和ES提供的日志分析友好的Web界面，可以帮助您汇总、分析和搜索重要数据日志。



## 3.安装过程

3.1.首先要保证 Windows 已经装好JDK ，并且配置好环境变量，这个就不多说了，应该大多都会配置

3.2.下载 elasticsearch logstash ,kibana  下载地址：<https://www.elastic.co/downloads> 

![](https://raw.githubusercontent.com/yang-zhijiang/learn-data/master/elk/%E5%AE%89%E8%A3%85/%E4%B8%8B%E8%BD%BD%E7%BD%91%E5%9D%80.png)

3.3.下载完成后分别解压 （windows 一般下载 ZIP 包）

![](https://raw.githubusercontent.com/yang-zhijiang/learn-data/master/elk/%E5%AE%89%E8%A3%85/%E8%A7%A3%E5%8E%8B.png)

## 4.启动

elasticsearch  kibana logstash 方式 很简单   

### 方式一

分别进入各自的bin 目录 双击 elasticsearch.bat  kibana.bat 即可运行

logstash 稍微复杂些，需要编写 logstash.conf  然后执行命令：

cmd 进入 bin 目录 执行命令

```java
  logstash.bat -f  logstash.conf 
```

### 方式二

将三者注册成windows 服务 用windows服务的方式来启动

#### 先配置elasticsearch 服务

cd到elasticsearch文件夹的bin目录下  

cmd 运行 elasticsearch-service install，会提示安装成功

cmd 运行 elasticsearch-service manager 会弹出服务管理界面，可以设置自动启动，并启动之。

浏览器访问 127.0.0.1:9200 ，出现成功的json

 ![](https://raw.githubusercontent.com/yang-zhijiang/learn-data/master/elk/%E5%AE%89%E8%A3%85/%E5%90%AF%E5%8A%A8%E6%88%90%E5%8A%9F.png)

配置logstash ,cd 到logstash文件夹的下bin目录

创建配置文件 logstash.conf  ,内容如下：

```sql
input{
   stdin {
   }
}
output{
    elasticsearch {
      hosts => ["127.0.0.1:9200"]
      index => "logstash-%{+YYYY.MM.dd}"
	  document_type => "form"
	  document_id => "%{id}"
    }
	stdout {
	   codec => json_lines
    }
}
```

这里有坑

1）编辑文件最好选择 notepad 打开 必须是 UTF-8 withou BOM 

否则 启动 logstash 会报错： 最好的方式不要去粘贴别人的，自己手动写logstash.conf  这个最保险

**"Expected one of #, input,filter, output at line 1, column 1 (byte 1) after "}>reasons**

网上解决办法如下： 但是不行

2）iconv 转换，iconv的命令格式如下：

iconv -f encoding -t encodinginputfile

比如将一个UTF-8 编码的文件转换成GBK编码

iconv -f GBK -t UTF-8 file1 -o file2

./bin/logstash  -e "input { stdin{}}  filter {}  output{ elasticsearch {  host =>"127.0.0.1:9200"    index=>"cpcn-%{+YYYY.MM.dd}" }stdout { codec => rubydebug }}"

**正确解决如下：**

 ![](https://raw.githubusercontent.com/yang-zhijiang/learn-data/master/elk/%E5%AE%89%E8%A3%85/%E6%AD%A3%E7%A1%AE%E8%A7%A3%E5%86%B3%E6%96%B9%E6%B3%95.png)

安装步骤：

cd到logstash文件夹下bin目录

创建一个run.bat 内容如下：

```sql
logstash.bat -f test1.conf 
```

#### 下载nssm

nssm 可以将其注册为windows 服务

```sql
https://nssm.cc/release/nssm-2.24.zip
```

解压拷贝nssm-2.24\win64目录下nssm.exe到logstash bin目录

cmd 运行 nssm.exe  install logstash

![](https://raw.githubusercontent.com/yang-zhijiang/learn-data/master/elk/%E5%AE%89%E8%A3%85/nssm.png)

 

会弹框：

![](https://raw.githubusercontent.com/yang-zhijiang/learn-data/master/elk/%E5%AE%89%E8%A3%85/%E5%BC%B9%E6%A1%86.png)

在弹出的界面设置 Path为run.bat,Details选项卡设置显示名，Dependencies选项卡设置依赖服务 elasticsearch-service-x64

最后点击install service 安装成功



#### 配置 kibana 服务

 config目录kibana.yml 修改配置：

```sql
elasticsearch.url: "http://127.0.0.1:9200"
server.port: 5601
```

和之前一样拷贝nssm文件，安装服务的Path为kibana.bat，依赖项可以设置logstash,elasticsearch-service-x64



服务 ：点击我的电脑-》右键 管理-》 服务  可以看到刚才装的三个服务，可以按照 es -logstash -<kibana 顺序启动起来

![](https://raw.githubusercontent.com/yang-zhijiang/learn-data/master/elk/%E5%AE%89%E8%A3%85/%E6%9C%8D%E5%8A%A1%E6%98%BE%E7%A4%BA.png)

 

装完之后，可以考虑用一下：

可能出现如下问题：

kibana 头顶上有一串红色的

[Unableto fetch mapping. Do you have indices matching the pattern? Windows](https://stackoverflow.com/questions/30862274/unable-to-fetch-mapping-do-you-have-indices-matching-the-pattern-windows)

 这个还是因为 logstash 未将数据传输到 ES ,并且没有默认索引 logstash-*

![](https://raw.githubusercontent.com/yang-zhijiang/learn-data/master/elk/%E5%AE%89%E8%A3%85/%E6%9C%AA%E4%BC%A0%E6%95%B0%E6%8D%AE.png)

只有当数据传输到logsestash 了 并且索引匹配了才可看到如下： 才有create

![](https://raw.githubusercontent.com/yang-zhijiang/learn-data/master/elk/%E5%AE%89%E8%A3%85/%E5%B7%B2%E4%BC%A0%E6%95%B0%E6%8D%AE.png)

 

logstash.conf 

这里创建了一个新索引 cpcn-*

```sql
input{
   stdin{}
}
output{
    elasticsearch {
      hosts => ["127.0.0.1:9200"]
      index => "cpcn-%{+YYYY.MM.dd}"
	  document_type => "form"
	  document_id => "%{id}"
    }
	stdout {
	   codec => json_lines
    }
}
```

这样才可以创建

![](https://raw.githubusercontent.com/yang-zhijiang/learn-data/master/elk/%E5%AE%89%E8%A3%85/%E5%88%9B%E5%BB%BA.png)当然 也可通过命令创建 索引

```sql
 put 索引名{
   setting{
   }
}
```

 

