# Nginx之四nginx.conf配置文件详解

nginx配置文件nginx.conf的讲解

注意点：nginx配置文件的每一条指令都必须用分号结束 ，下面指令配置中 “|”表示或者； “[]”表示可选

## 一、配置文件结构

包含**全局块**，**events块**和**http块**（http块包含http全局块和多个server块-server块又包含server全局块和location块）;如下所示:（另外：外层的指令可以作用于自身的块和此块中的所有低层级块，并且还遵循低级优于高级，类似于Java中的全局变量和局部变量）。

```
worker_processes 4;  
... #全局块
events  #events块
{
  ...
}
http  #http块
{
    ...   #http全局块
    server  #server块 相当于一个虚拟机
    {
        ... #server全局块
        location [Pattern]
        {
           ...
        }
    }
    server 
    {
        ...
    }
    ...
}
```

 大致的模块如下分类：

| 名称       | 说明                                                         |
| ---------- | ------------------------------------------------------------ |
| 全局块     | （从文件开始到events块的内容）用来设置影响Nginx服务器整体运行的配置，作用于全局<br>作用：通常包括服务器的用户组，允许生成的worker process、Nginx进程PID的存放路径、日志的存放路径和类型以及配置文件引入等 |
| events块   | 涉及的指令主要影响Nginx服务器和用户的网络连接。<br/>作用：常用到的设置包括是否开启多worker process下的网络连接进行序列化，是否允许同时接收多个网络连接，选择何种时间驱动模型处理连接请求，每个worker process可以同时支持的最大连接数等 |
| http块     | 这个很重要，包含代理，缓存和日志定义等绝大部分功能和第三方模块的配置都可以放在这个模块中<br/>作用：配置的指令包含文件引入，MIME-Type的定义，日志定义，是否使用sendfile传输文件、连接超时时间，单链接请求数的上限等。 |
| server块   | server块 就相当于虚拟机，一个server就相当于一个虚拟机，所以一个nginx相当于对外可以提供多个虚拟机；server全局块  配置虚拟机的监听配置和虚拟机的名称或者IP<br/>作用：使得Nginx服务器可以在同一台服务器上至运行一组Nginx进程，就可以运行多个网站。 |
| location块 | 一个server可以包含多个location ,'[Pattern]' 这里对去除主机名称或者IP后的字符做匹配（例如server_name/uri_string,那么这里就对‘/uri_string’ 匹配）<br/>作用：基于Nginx服务器接收到的请求字符串，虚拟主机名称（ip，域名）、url匹配，对特定请求进行处理。 |



## 二、配置文件的指令解析

### 1、全局块指令配置

1）配置运行Nginx用户组 指令是user  只能在全局块中配置

```
语法格式： user user [group]
```

user（第二个）:表示可以运行Nginx服务器的用户，group可选项表示运行Nginx服务器的用户组；只有被设置了用户或者用户组的成员才有权限启动Nginx进程。如果非这些用户就无法启动，还会报错；

如果想所有的用户都可以启动这可以把当前行注释掉，或者使用user nobody （默认是注释掉的）

2）配置worker_process 。只能在全局块中配置

这个受软件，操作系统本身，还有硬件本身等制约，一般和cpu数量相等；

    语法格式：worker_process number | auto 
    
    number：指定Nginx进程最多可以产生的worker process数量，默认配置是1
    
    auto：设置此值，Nginx进程将自动检测 

当我们启动nginx,后使用ps -ef | grep nginx 可以看到除了master的nginx 还有相对应设置数量的worker process。

3）配置Nginx进程PID存放路径。指令pid 只能在全局块中配置

      语法： pid     file  

  file是指定存放路径和文件名称，默认是logs/nginx.pid。可以放相对和绝对路径。

4）配置错误日志的存放路径 ，指令error_log ，可以在全局块、http块、server块、以及location块中配置

    语法 error_log  file/stderr [debug | info | notice | warn | error | crit | alert | emerg] ;

  ‘[]’:表示可选 ，‘|’ 表示或者

    file表示文件 ； stderr 表示输出文件名称 ，后面[]中表示数据的日志级别，当设置某一个级别后，比这一级别高的都会被记录下来，比如设置warn后| error | crit | alert | emerg都会被记录下来。默认是error_log logs/error.log error; 指定文件当前用户需要有写权限

5）配置文件的引入，指令include  可以在配置文件的任何地方配置

        当我们需要其他Nginx的配置以及第三方模块的配置引用到当前的主配置文件中时候就可以使用这个指令

   语法 include file 

    file表示要引入的配置文件，支持相对路径，引入的文件对当前用户需要写权限

### 2、events块指令配置

6）设置网络连接序列化  只能在events块设置

        这个是是否开启“惊群”，当某一时刻请求到来是否唤醒多个睡眠的进程

```
语法：accept_mutext on | off 
```

**默认是on状态**

7）设置是否允许同时接收多个网络连接  只能在events块设置

每一个worker process都有能力接受多个新到达的网络连接。但是需要设置如下指令

```
语法：multi_accept on | off
```

默认是off状态，即一个work process一次只能接受一个到达的网络连接

8）事件驱动模型进行消息处理   只能在events块设置

用来处理网络消息，method选择了理性有select,poll,kqueue,epoll,rtsig,/dev/poll，eventport

```
语法： user method;
```

也可以在编译的时候通过--with-select-module,--without-select-modules设置是否强制编译select模块到Nginx内核

9）配置最大连接数    只能在events块设置

用来开启work process 的最大连接数

```
语法：worker_connections number 
```

默认值是512，另外这里的number不仅仅包含所有和前端的连接数，是包含所有的连接数，另外这里的number是不能操作操作系统支持打开的最大文件句柄数量 

### 3、Http块指令配置

10）定义MIME-Type  ,默认如下   可以在http块，server和location块配置

    include       mime.types;     //这个是导入type块
    default_type  application/octet-stream;  //配置处理前端请求的MIME类型

这个是用用识别前端请求资源类型，模式使用的是include mime.type

mime.type中包含了很多资源类型

```
types {
    text/html                             html htm shtml;
    text/css                              css;
    text/xml                              xml;
    image/gif                             gif;
    image/jpeg                            jpeg jpg;
    application/javascript                js;
    application/atom+xml                  atom;
    application/rss+xml                   rss;
 
    text/mathml                           mml;
    text/plain                            txt;
    text/vnd.sun.j2me.app-descriptor      jad;
    text/vnd.wap.wml                      wml;
    text/x-component                      htc;
.....
}
```

11）自定义服务日志   

这个是记录Nginx服务器提供服务过程应答前端请求的日志，我们用服务日志和之前的error_log加以区分。Nginx服务器支持对服务日志的格式、大小、输出等进行配置，需要使用两个指令，分别是access_log和log_format

```
a:    access_log指令语法：access_log path [format []buffer=size]      可以在http块，server块和location块中配置

path:配置服务日志的文件存放的路径和名称

format:可选项，自定义服务日志的格式字符串，也可以通过“格式串的名称，使用log_format指令定义好的格式，格式串的名称在log_format中定义

size:配置临时存放日志的内存缓存区大小；

如果要取消服务日志的功能使用  accesss_log off;
```

文件中被注释的：

```
log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '

'$status $body_bytes_sent "$http_referer" '

'"$http_user_agent" "$http_x_forwarded_for"';

access_log  logs/access.log  main;   //这里的main就是上面log_format中的格式
```

access_log和log_format是联合使用的。

```
b：  log_format的格式  ：log_format name string ...；     只能在http块

name:格式字符串的名字，默认是combined。

string:服务日志的格式字符串，在定义的时候可以通过变量获取相关的内容。
```

如上所示的

“$remote_addr”获取的是用户机的ip地址，

“$time_local”获取的本地时间，

“$request”获取到的是如GET.... 什么请求等

“$status”这个是请求状态

“body_bytes_sent”获取到的是请求体的大小

“http_referer”指http的referer值，可以为1,0,_等

“http_user_agent”：获取到用户使用什么浏览器

例如：日志如下：

```
93.157.175.41 - - [14/Oct/2018:14:34:26 +0000] "GET /axis-cgi/jpg/image.cgi HTTP/1.1" 404 168 "1" "Opera/9.80 (Windows NT 5.1; U; ru) Presto/2.9.168 Version/11.51"
```

12）配置文件允许sendfile方式传输文件  **指令都可以在http块，server块和location块中配置**

用于是否开启或者关闭传输文件，模式是off

```
a:  语法  sendfile on | off

另外可以设置传输的大小

b:  语法：sendfile_max_chunk size;

如果size值大于0，Nginx进程的每一个worker process每次调用sendfile()传输的数据量最大不能超过这个值如果设置为0，则不限制。默认0；
```

13）配置连接超时时间         **指令都可以在http块，server块和location块中配置**

和用户建立会话连接后，Nginx服务可以保持连接打开一段时间，指令keepalive_timeout用来设置整个时间

```
语法：keepalive_timeout timeout [header_timeout]  

timeout:表示服务端对连接保持的时间。默认是75秒

header_timeout:可选项，在应答报文头部的Keep_Alive域设置超时时间，“Keep_Alive:timeout=header_timeout”.报文中的这个指令可以被Mozilla或者Konqueror识别
```

例如：

```
keepalive_timeout 120s 100s;  //含义是服务器端保持连接的时间是120s,发给用户端的应答报文头部中的Keep-Alive域的超时时间设置为100s
```

14）单链接请求数上限 指令都可以在http块，server块和location块中配置

Nginx服务器和用户端建立会话连接后，用户通过此链接发送请求，指令keepalive_requests用于限制用户通过某一个连接想Nginx服务器发送请求的次数。默认值是100

```
语法：keepalive_requests number:
```

### 4、Server块指令配置

15）配置网络监听

配置监听使用指令listen，方式有三种

```
方式一：监听IP地址，

语法：listen address[:port][default_server] [setfib=number][backlog=number] [rcvbuf=size][sndbuf=size] [deferrred][accept_filter=filter] [bind][ssl];

方式二：监听端口，

语法：listen port [default_server][setfib=number] [backlog=number][rcvbuf=size] [sndbuf=size][accept_filter=filter] [deferrred][bind] [ipv6only= on | off][ssl];

方式三：配置UNIX Domain Socket（在原有Socket框架上发展起来的IPC机制）

语法：listen unix:path  [default_server][backlog=number] [rcvbuf=size][sndbuf=size] [accept_filter=filter][deferrred] [bind][ssl];
```

address:IP地址，如果是IPv6的地址，需要使用“[]”括起来，比如[fe22::1]等

port:端口号，如果只定义了IP地址 没有定义端口号，就使用80端口

path:socket文件的路径，例/var/run/nginx.sock等

default_server，标识符，将持虚拟主机设置为address:port的默认主机

sertfib=number:使用这变量监听socket关联的路由表，目前只对FreddBSD起作用，不常用

backlog=nubmer ,设置监听函数listen()最多允许多少网络连接同时处于挂起状态，在FreeBSD中默认是-1,其他平台默认是511

rcvbug=size,设置监听socket接受缓冲区的大小

sndbuf=size,设置监听socket发送缓冲区的大小

deferred,标识符，将accept()设置为Deferred模式

accept_filter=filter，设置监听端口对请求的过滤，被过滤内容是不能被接受和处理。这个指令只在FreeBSD和NetBSD5.0+平台有效，filter可以设置为dataready或者httpready

bind:表示符，使用独立的bind()处理此address:port，一般情况下，对于端口相同而IP地址不相同的多个连接，Nginx服务将只能使用一个监听命令，并使用bind()处理端口相同的所有连接

ssl：标识符，设置会话连接使用SSL模式进行，此标识符和Nginx服务器提供的HTTPS服务相关

```
常用例子：

listen 123.22.3.10:8000; //监听具体的IP和具体端口上的连接

listen 123.22.3.10; //监听具体Ip的所有端口的连接

listen 8000; //监听具体端口上所有IP的连接，等同与 listen *:8000；

listen 123.22.3.10:8000 default_server backlog=1024; //设置123.22.3.10的连接请求默认由虚拟主机处理，并且允许最多1024网络连接出去挂起状态。
```

16）基于名称的虚拟主机配置

这里的主机指用server块对外提供的虚拟主机，设置了主机的名称并配置好了DNS，用户就可以使用这个名称向此虚拟主机发送请求了。配置注解名称的指令：

```
语法： server_name name...; //name 可以多个，中间用空格并列

例如：

server_name vison.com  ws.vison.com;  //这里有两个虚拟机名称，Nginx规定第一个为此虚拟机主要的名称
```

**在name中可以使用通配符“*”**，通常用在三段字符串组成的名称的首段或者尾段，或者两段字符串组成名称的尾段

    例如server_name *.vison.com www.vison.*  vison.*;

**在name中可以使用正则表达式**,并使用“~”作为正则表达式字符串的开始标记

    例如：server_name ~^www\d+\.vison\.com$;  //这个表示用www开头(“^”标记），紧跟一个或者多个0-9的数字，在紧跟.vison.com（"."在正则表达式有特殊含义，需要使用转义），最后com结束（“$”标记）

这个的名称我们可以通过www33.vison.com 或者 www.2.vison.com等访问。

**name使用$1，$2...捕获变量**

注意点：从Nginx-0.7.40开始，name的正则表达式支持字符串捕获功能，就是把正则表达式中的某一段作为变量给后面表达式的使用，拾取的标识是一个完整的小括号“()”且后面不紧跟其他正则表达式字符，括号中的内容就是被拾取的内容；一个正则表达式可以存在多个不嵌套的小括号，这些内容依次从左到有存放在$1,$2,$3...中。下文使用时，直接使用这些变量，但是变量的有效区只能在server块中

```
例如：server_name ~^www\.(.+)\.com$;
```

当前服务器请求www.vison.com时，我们的“vison”就会被记录到$1中。后面我们需要vison的时候就可以通过$1使用了。

name用正则表达式和通配符有可能匹配到相同的配置，那么当出现的是否怎么选择呢：nginx做出如下配置

  排在前面的优先处理请求：

    1、精确匹配server_name
    
    2、通配符在开始时匹配server_name成功
    
    3、通配符在结尾时匹配server_name成功
    
    4、正则表达式匹配server_name成功

17） 基于IP的虚拟主机配置

和基于名称的虚拟配置相同，也是使用server_name配置

例如： server_name 122.12.3.6;

### 5、location块指令配置

18）location指令配置

```
这个location的值是匹配请求连接中的uri

语法格式：  location [= | ~ | ~* | ^~]  uri  { ... }

a）uri表示待匹配的请求字符，可以是正则或者不是正则字符例如：vison.action 表示标准uri  ； \.action$表示以action结尾url

b) “[]”中括号中的是可选项，用来改变请求字符串和uri的匹配方式，

这里有四种：

“=”  用于标准的uri(没有使用正则表达式等)前，要求请求字符串和uri严格匹配，如果匹配成功就停止搜索并立即处理请求

“~” 用于表示uri包含正则表达式，并且区分大小写

“~*”,用于表示uri包含正则表达式，并且不区分大小写 ，

 注意：如果包含uri正则表达式，必须使用“~” 或者 “~*” 标识

“^~” ,用于标准uri(没有使用正则表达式)，要求Nginx服务器找到表示uri和请求字符串匹配度最高的location后，立即使用location处理请求，而不使用location块中的正则uri和请求字符串做匹配。

注意点：浏览器在传送uri的时候会对部分字符url编码，例如空格编码为“%20”，问好编码为“%3f”等，“^~”它可以对这些符号进行编码处理。例如如果URI为“/html/%20/data”,则当Nginx可以收到配置“^~ /html/ /data” 同样也是匹配成功的。
```

19）配置请求的根目录   这个通常是在location块中的配置

这个通常是在location块中的配置，当web服务器接受到网络请求之后，首先需要在服务器端指定目录中寻找请求资源，在Nginx服务器中，指令root就是用来配置这个根目录的。

```
语法：root path;

其中path为Nginx服务器接收到请求以后查找资源的根目录路径，path变量中可以包含Nginx服务器预设的大多数变量，只用$document_root和$realpath_root不可以使用。

例如：

location   /data/  {
      root  /locationtest1
}
```

当location块接收到"/data/index.html" 请求后，将在nginx的“/locationtest1/data/”目录下找到 index.html响应请求。

20）更改location的uri --转发

在location块中，除了使用root指令指明处理根目录，还可以使用alias指令改变location接受到的URI的请求路径

```
语法：alias path;

path:这个就为修改后的根路径，同样这个变量可以包含除了$document_root和$realpath_root的变量。

例如：

location ~ ^/data/(.+\.(htm|html))${

	alias /locationtest1/other/$1;

}  //  当location块接收到“/data/index.html”的请求时，匹配到alias指令的配置,Nginx服务器将到“locationtest1/other”目录下找到index.html并响应请求，可以看到，通过alias指令的配置，根路径已经从/data/ 更改为locationtest1/other了。
```

21 ） 设置网站的默认首页

指令index用来设置网站默认首页、一般有两个作用：

一是：用户在发出请求网站时，请求地址可以不写首页名称

二是：可以对一个请求，根据请求内容而设置不同的首页

```
语法结构：index  file ...  ； file变量可以包含多个文件，中间用空格隔开也可以包含其他变量，默认使用 "index.html";

例子：

location ~ ^ /data/(.+)/web/${

	index index.$1.html  index.m1.html index.html 

}  //当location匹配到“/data/locationtest/web” 时，匹配成功，$1的值就为“locationtest”,那么就会依次寻找这个页面下的 index.locationtest.html   index.m1.html index.html  ，先找到那个就返回那个。
```

22） 设置网站的错误网页      可以在http块，server块和location块中配置

如果用户尝试查看某一个网页出现错误时，我们要返回Http错误网页，用我们自定义的网页显得跟人性化，一般来说HTTP 2XX表示请求正常 ；HTTP 3XX表示网站重定向  ； HTTP 4XX表示客户端出现错误  ； HTTP 5XX 表示服务端出现错误。nginx在设置网站错误页面的指令为error_page。

    语法： error_page code ... [=[response] ] uri;
    
    code:表示要处理的HTTP错误代码，
    
    response:可选项，将code指定的错误代码转换为新的错误代码response
    
    uri，错误页面的路径或者网址地址，如果设置为路径，则是以Nginx服务器安装路径下的html目录为根路径的相对地址，如果设置为网址，nginx服务器会直接访问该网址获取错误页面，并返回给用户端
    
    例如 
    
    error_page 404 /404.html      //通过 Nginx安装路径/html/404.html        响应404错误
    
    error_page 404 http:/somewebsite.com      //出现404错误，直接响应这个连接
    
    error_page 410=310 /empty.gif            //当产生410HTTP消息时，使用Nginx安装路径/html/empty.gif 返回用户端310消息 



如果需要更改error_page 默认的安装路径，怎么修改呢，那么就需要添加location就可以了。

```
例如：

locatin  /404.html{

	root  /myserver/errorpages/

}   //首先捕获到“/404.html页面”请求，然后将请求定向到新的路径下面，
```

 

23）基于IP配置Nginx的访问权限    可以在http块，server块和location块中配置

Nginx配置通过两种途径支持基本访问权限控制，其中一种是由HTTP标准模块ngx_http_access_module支持的，其通过IP来判断客户端是否拥有对Nginx的访问权限，

  a） **allow指令**，用于设置允许访问Nginx的客户端IP,

```
语法： allow address | CIDR | all ;

address：表示允许访问的客户端IP,不支持同时设置多个，如果需要多个IP,那么需要重复使用allow命令

CIDR:允许访问的客户端的CIDR地址，如果202.80.18.22/25，前面的32位IP地址，后面“/25”代表该IP地址中前25位网络部分，其余位表示主机部分。

all:代表允许所有客户端访问。
```

从Nginx 0.8.22 该命令也支持IPv6地址。例如：allow 2301:333:e000::8001;

b） **另一个指令是deny**，作用刚好和allow相反，用来禁止访问Nginx的IP

```
语法: deny address | CIDR | all ;  

上面的字符解释类似
```

 

24） 基于密码配置的Nginx的访问权限

Nginx支持识别用户名和密码的方式认证客户端是否能够访问Nginx，这个功能是由HTTP标准模块ngx_http_auth_basic_module支持

```
a）指令auth_basic ,用来开启和关闭该认证功能

语法：auth_basic string | off

string:开启该认证功能，并配置验证时的指示信息

off:关闭该验证功能

b）auth_basic_user_file 用于设置包含用户名和密码信息的文件路径，

语法：auth_basic_user_file file

file:表示密码文件的绝对路径。文件中的密码可以使用明文或者加密后的密码，
```

明文如下：

name1:password1

name2:password2:comment

name3:password3

加密密码可以使用crypt()函数进行加密。

去掉注释的默认的配置文件
如下：

```
worker_processes  1;

events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;
    server {
        listen       80;
        server_name  localhost;
        location / {
            root   html;
            index  index.html index.htm;
        }
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }

    }

}
```



