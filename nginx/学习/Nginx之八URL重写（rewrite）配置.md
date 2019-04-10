# Nginx之八 URL重写（rewrite）配置

## Nginx URL重写（rewrite）配置及信息详解

**1、if判断指令**

```
语法为if(condition){…}     #对给定的条件condition进行判断。
如果为真，大括号内的rewrite指令将被执行，if条件(conditon)可以是如下任何内容：

  a:当表达式只是一个变量时，如果值为空或任何以0开头的字符串都会当做false，其他情况为true。 
  b: 直接比较变量和内容时，使用 = 或!= 
  c: 正则表达式匹配，*不区分大小写的匹配，!和！*反之。 

注意：使用正则表达式字符串一般不需要加引号，但是如果含有右花括号“}”或者分号“；”字符时，必须要给整个正则表达式加引号

其他指令：
-f和!-f用来判断请求文件是否存在
-d和!-d用来判断请求目录是否存在
-e和!-e用来判断是请求的文件或者目录否存在
-x和!-x用来判断请求的文件是否可执行 

例子：if (-f $request_filename){
        … #判断请求的文件是否存在，存在就执行这里面的代码块
    }
```

**2、break指令**

```
用于中断当前相同作用域中的Nginx配置，和Java中的break语法类似，可以在server块和location以及if块中使用。
语法：break;
```

**3、if 可用的全局变量**

还有$host_host变量，和$host区别如下：
$host不带端口，$http_host带端口

![å¨è¿éæå¥å¾çæè¿°](https://img-blog.csdn.net/20181023233221230?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MDc5Mjg3OA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)



**4、return指令**

```
该指令用于完成对请求的处理，直接向客户端响应状态的代码。和Java中的return语法类似。可以再server块和location以及if块中使用。 

语法：return code URL;  #code表示状态码，URL表示返回给客户单的URL地址
或者：return URL: #当状态码是302或者307的时候，可以使用，返回的URL必须包含“http://”、“https://”或者直接使用“$scheme”变量(RequestScheme代表传输协议，
Nginx内置变量)
或者 return [text]; #为返回给客户端的响应体内容，支持变量的使用
```



**5、rewrite指令**

```
该指令通过正则表达式的使用来改变URI.可以同时存在一个或者多个指令，按照顺序一次对URL进行匹配和处理。该指令可以在server块后者location块中配置 
语法： 　
指令语法：rewrite regex replacement [flag];
rewrite：是实现URL重定向的重要指令， 　
regex：用来匹配URI的正则表达式；
replacement：匹配成功后用来替换URI中被截取内容的字符串，默认情况如果该字符串包含“http://”、"https://"开头，则不会继续向下对URI进行其他处理。直接返回重写的URI给客户端
flag：用来设置rewrite对URI的处理行为,包含如下数据：
```

| 标记符号 | 说明                                                         |
| -------- | ------------------------------------------------------------ |
| last     | 终止在本location块中处理接收到的URI，并将此处重写的URI作为新的URI使用其他location进行处理。（只是终止当前location的处理） |
| break    | 将此处重写的URI作为一个新的URI在当前location中继续执行，并不会将新的URI转向其他location。 |
| redirect | 将重写后的URI返回个客户端，状态码是302，表明临时重定向，主要用在replacement字符串不以“http://”，“ https://”或“ $scheme” 开头； |
| redirect | 将重写的URI返回客户端，状态码为301,指明是永久重定向；        |



**6、rewrite_log指令**

```

```



**7、set指令**

```

```



**8、uninitialized_variable_warn指令**

```

```



**9、防盗链的例子**

```

```

这里表示请求头部Referer域是否匹配上面值，如果匹配了$invalid_referer 的值为0，没有相匹配就是1；

| 字符        | 说明                                                         |
| ----------- | ------------------------------------------------------------ |
| none        | 表示Referer头域不存在的情况                                  |
| blocked     | 检测Referer头域的值被防火墙或者代理服务器删除或者伪装的情况，这种情况，该头域的值不以“http://”或者“https://”开头 |
| server_name | 设置一个或者多个URL，检测Referer头域的值是否是这些URL中的某个 |



**10、例子**

```

```

