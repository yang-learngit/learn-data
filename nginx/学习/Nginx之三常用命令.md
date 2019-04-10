# Nginx之三常用命令

**以下命令都是在如下nginx安装目录下操作**
 （cd /usr/local/nginx）    

**pid**  指的是nginx工作进程id

nginx工作是一个主进程多个工作进程

**主进程的PID获取** 

```
ps -ef | grep nginx  或者  
cat logs/nginx.pid   这个文件中保存了主进程的pid
```

检查配置是否正确：

```
./nginx -t  -c filename   可以检查配置文件语法是否正确
```

**启动**：

    ./nginx -h   可以查看nginx使用的帮助信息
```
./nginx     直接运行
```

**停止**：

```
./nginx -g INT  或者 ./nginx -g TEMP   快速停止；马上丢弃连接，停止工作
./nginx -g QUIT 平缓停止 ；处理完当前正在处理的请求但是不接受新的请求，然后关闭连接停止工作
kill -g pid   立即停止 
```

​    
**重启**：

    ./nginx -g HUP [-c newConfFile]    平滑重启，newConfFile 是可选项    
    ./nginx -s reload   修改配置后重新加载生效   
升级:

    ./nginx -g USR2    平滑升级       
    ./nginx -g WINCH   发送平滑停止旧服务信号   

另外想主进程发送信号也可使用

```
 kill SIGNAL PID 
```

**Nginx可以接受接受的信号有**： 

- TERM或INT 快速停止Nginx服务   kill  TEMP  pid   上面的写法也可以这样写，下面的类似

- QUIT 平缓停止Nginx服务      kill  QUIT  pid
- HUP 使用新的配置文件启动进程，之后平缓停止原有进程，也就是所谓的”平滑重启”  kill  HUP  pid
- USR1 重新打开日志文件，常用于日志切割，在相关章节中会对此进一步说明   kill  USR1  pid
- USR2 使用新版本的Nginx文件启动服务，之后平缓停止原有Nginx进程，也就是所谓的”平滑升级”   kill  USR2  pid
- WINCH  平缓停止worker process,用于Nginx服务器平滑升级   kill  WINCH  pid

**另外一些常见的命令**：

```
./nginx -h 查看nginx所有的命令参数
Options:
  -?,-h         : this help
  -v            : show version and exit
  -V            : show version and configure options then exit
  -t            : test configuration and exit
  -T            : test configuration, dump it and exit
  -q            : suppress non-error messages during configuration testing
  -s signal     : send signal to a master process: stop, quit, reopen, reload
  -p prefix     : set prefix path (default: /usr/local/nginx/)
  -c filename   : set configuration file (default: conf/nginx.conf)
  -g directives : set global directives out of configuration file
```

```
./nginx -v  显示nginx的版本号
./nginx -V  显示nginx的版本号和编译信息
./nginx -t  检查nginx配置文件的正确性
./nginx -t  检查nginx配置文件的正确定及配置文件的详细配置内容
./nginx -s  向主进程发送信号，如:./nginx -s reload 配置文件变化后重新加载配置文件并重启nginx服务
./nginx -p  设置nginx的安装路径
```



```
./nginx -c  设置nginx配置文件的路径
```

