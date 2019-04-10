# Nginx之二安装Nginx及文件简单介绍

### **1. 安装Nginx** 

**通过源码包编译安装**

这种方式可以自定安装指定的模块以及最新的版本。方式更灵活。

官方下载页面：<http://nginx.org/en/download.html>

configure配置文件详解：<http://nginx.org/en/docs/configure.html>

 

> 1）为了编译nginx源代码，需要标准的GCC（GNU Compiler Collection）编译器（用来处理c语言，现在可以处理包含Java，C++等语言）。
>
> 2）需要Automake工具，完成自动创建Makefile的工作，不需要可以不下载
>
> 3）由于Nginx需要依赖其他三方库，所以需要pcre库（支持rewrite模块）、zlib（支持gzip模块）、openssl库（支持ssl模块）

   

**安装gcc g++的依赖库**

```
sudo apt-get install build-essential

sudo apt-get install libtool
```



**安装pcre依赖库（http://www.pcre.org/）**

```
sudo apt-get update

sudo apt-get install libpcre3 libpcre3-dev
```



**安装zlib依赖库（http://www.zlib.net）**

```
sudo apt-get install zlib1g-dev
```



**安装SSL依赖库（16.04默认已经安装了）**

```
sudo apt-get install openssl
```



**安装Nginx**

```
#下载最新版本：
wget http://nginx.org/download/nginx-1.9.3.tar.gz
#解压：
tar -zxvf nginx-1.9.3.tar.gz
#进入解压目录：
cd nginx-1.9.3
#配置：
./configure --prefix=/usr/local/nginx 
#编译：
make
#安装：
sudo make install
#进入/nginx/sbin启动：
./nginx

#查看进程：
ps -ef | grep nginx
```



### 2.Nginx解压文件讲解

如下所示这是Nginx解压后的文件

```
auto    CHANGES.ru    configure    html    Makefile    objs    src    CHANGES    conf  contrib    LICENSE    man    README
```

**src**：存放了Nginx软甲的所有源代码

**man**：存放了Nginx软件的帮助文档

**html**：包含两个.html的静态文件，这个和Nginx服务运行有关

**config**：存放了Nginx服务器配置文件，包含Nginx服务器的基本配置文件和对部分特性的配置文件

**auto**：存放了大量脚本文件和configure脚本程序有关

**configure**：这个文件是Nginx软件的自动脚本。运行configure自动脚本一般会完成两个工作，一是检查环境，根据环境检查结果生成C代码，二是生成编译代码需要Makefile文件。

CHANGES,CHANGE.ru ,LICENSE,README:这些是Nginx服务器相关的文档资料

 

**Nginx的源代码的编译需要先使用configure脚本自动生成Makefile文件，configure脚本有很多选项支持，上面我们编译的时候使用了**  ./configure --prefix=/usr/local/nginx  这里指定了Ngin软件的安装路径/usr/local/nginx， 未指定也是这个默认地址的。使用了这个命令，也会使用到auto文件中的脚本命令

 

### 3.Nginx 安装目录介绍

在上面使用如下命令执行后：就会在/usr/local/nginx 生成一个新的安装目录文件

```
#配置：
./configure --prefix=/usr/local/nginx 
\#编译：
make
\#安装：
sudo make install
```



这些文件主要包含四个：

**conf**：存放了Nginx的配置文件，其中nginx.conf是主配置文件，其他的都是配置Nginx相关的功能的，比如配置fastcgi使用fastcgi.conf和fastcgi_params两个文件。 .default结尾的都是默认的配置文件。方便将我们配置的.conf恢复到初始状态。

**html**：存放了Nginx服务器在运行过程中调用的一些html网页文件

**logs**：用来存放Nginx服务器的日志的，

**sbin**：只有一个nginx文件，就是Nginx服务器的主程序

 