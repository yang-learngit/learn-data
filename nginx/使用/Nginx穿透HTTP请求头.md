# Nginx穿透HTTP请求头

在调用电子签章接口时，通过nginx进行转发http请求，无法访问，查找原因后得知是nginx在转发请求时会默认忽略带有"_"下划线的头信息:

nginx默认request的header的参数包含"_"时，会自动忽略掉。

解决方法是：

在nginx里的nginx.conf配置文件中的HTTP部分或server部分中添加如下配置：

underscores_in_headers on; (默认underscores_in_headers为off)

参考nginx官网说明：<http://nginx.org/en/docs/http/ngx_http_core_module.html#underscores_in_headers>



# 1.问题

之前在一台ECS上搭了个nginx搞跳转服务，昨天突然发现header中自定义数据一直获取不到。因为利用nginx跳转时读取header异常，不用nginx时读取header正常。

先来看看nginx.conf配置文件：

```
worker_processes  1;

#pid        logs/nginx.pid;

events {
    worker_connections  1024;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';

    sendfile        on;

    keepalive_timeout  65;

    server {
        listen       80;
        server_name  http://11.239.162.48;

        location / {
            #root   html;
            #index  index.html index.htm;
            proxy_set_header Host $host;
            proxy_set_header X-Real-Ip $remote_addr;
            proxy_set_header X-Forwarded-For $remote_addr;
            proxy_pass http://11.239.162.48:7000;
            client_max_body_size  100m;
        }
        
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }
}
```

# 2.调查

查看nginx的log日志时，也没有明显的异常记录。查了下网络上的相关信息，有以下两点：

1、默认的情况下nginx引用header变量时不能使用带下划线的变量。要解决这样的问题只能单独配置underscores_in_headers on； 
2、默认的情况下会忽略掉带下划线的变量。要解决这个需要配置ignore_invalid_headers off。

查看自己header中自定义的变量时，有一个project_id包含下划线，接口处理时一直获取不到其值，所以提示异常。

不过，我仅将underscores_in_headers on配置信息加到http块中去了，业务恢复正常。

```
http {
    include       mime.types;
    default_type  application/octet-stream;

    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log  logs/access.log  main;

    sendfile        on;
    #tcp_nopush     on;
    underscores_in_headers on;
    #keepalive_timeout  0;
    keepalive_timeout  65;
```

# 3.分析

需要看看nginx的一段源码，其中有这么一个片段：

```
ngx_http_parse_header_line(ngx_http_request_t *r, ngx_buf_t *b,ngx_uint_t allow_underscores)

if (ch == '_') {
    if (allow_underscores) {
        hash = ngx_hash(0, ch);
        r->lowcase_header[0] = ch;
        i = 1;
    } else {
        r->invalid_header = 1;
    }
    break;
}
```

即包含一个关键变量：allow_underscores，是否允许下划线。

原来nginx对header name的字符做了限制，默认 underscores_in_headers 为off，表示如果header name中包含下划线，则忽略掉。而我的自定义header中恰巧有下划线变量。

所以，后续再碰到类似情况，要么header中自定义变量名不要用下划线，要么在nginx.conf中加上underscores_in_headers on配置。

源码地址为：<https://trac.nginx.org/nginx/browser/nginx/src/http/ngx_http_parse.c>

![è¿éåå¾çæè¿°](https://img-blog.csdn.net/20171011190035988?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbG9vbmdzaGF3bg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

```
ngx_int_t
	ngx_http_parse_header_line(ngx_http_request_t *r, ngx_buf_t *b,
	    ngx_uint_t allow_underscores)
	{
	    u_char      c, ch, *p;
	    ngx_uint_t  hash, i;
	    enum {
	        sw_start = 0,
	        sw_name,
	        sw_space_before_value,
	        sw_value,
	        sw_space_after_value,
	        sw_ignore_line,
	        sw_almost_done,
	        sw_header_almost_done
	    } state;
	
	    /* the last '\0' is not needed because string is zero terminated */
	
	    static u_char  lowcase[] =
	        "\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0"
	        "\0\0\0\0\0\0\0\0\0\0\0\0\0-\0\0" "0123456789\0\0\0\0\0\0"
	        "\0abcdefghijklmnopqrstuvwxyz\0\0\0\0\0"
	        "\0abcdefghijklmnopqrstuvwxyz\0\0\0\0\0"
	        "\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0"
	        "\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0"
	        "\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0"
	        "\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0";
	
	    state = r->state;
	    hash = r->header_hash;
	    i = r->lowcase_index;
	
	    for (p = b->pos; p < b->last; p++) {
	        ch = *p;
	
	        switch (state) {
	
	        /* first char */
	        case sw_start:
	            r->header_name_start = p;
	            r->invalid_header = 0;
	
	            switch (ch) {
	            case CR:
	                r->header_end = p;
	                state = sw_header_almost_done;
	                break;
	            case LF:
	                r->header_end = p;
	                goto header_done;
	            default:
	                state = sw_name;
	
	                c = lowcase[ch];
	
	                if (c) {
	                    hash = ngx_hash(0, c);
	                    r->lowcase_header[0] = c;
	                    i = 1;
	                    break;
	                }
	
	                if (ch == '_') {
	                    if (allow_underscores) {
	                        hash = ngx_hash(0, ch);
	                        r->lowcase_header[0] = ch;
	                        i = 1;
	
	                    } else {
	                        r->invalid_header = 1;
	                    }
	
	                    break;
	                }
	
	                if (ch == '\0') {
	                    return NGX_HTTP_PARSE_INVALID_HEADER;
	                }
	
	                r->invalid_header = 1;
	
	                break;
	
	            }
	            break;
	            
	   /* header name */
	        case sw_name:
	            c = lowcase[ch];
	
	            if (c) {
	                hash = ngx_hash(hash, c);
	                r->lowcase_header[i++] = c;
	                i &= (NGX_HTTP_LC_HEADER_LEN - 1);
	                break;
	            }
	
	            if (ch == '_') {
	                if (allow_underscores) {
	                    hash = ngx_hash(hash, ch);
	                    r->lowcase_header[i++] = ch;
	                    i &= (NGX_HTTP_LC_HEADER_LEN - 1);
	
	                } else {
	                    r->invalid_header = 1;
	                }
	
	                break;
	            }
	
	            if (ch == ':') {
	                r->header_name_end = p;
	                state = sw_space_before_value;
	                break;
	            }
	
	            if (ch == CR) {
	                r->header_name_end = p;
	                r->header_start = p;
	                r->header_end = p;
	                state = sw_almost_done;
	                break;
	            }
	
	            if (ch == LF) {
	                r->header_name_end = p;
	                r->header_start = p;
	                r->header_end = p;
	                goto done;
	            }
	
	            /* IIS may send the duplicate "HTTP/1.1 ..." lines */
	            if (ch == '/'
	                && r->upstream
	                && p - r->header_name_start == 4
	                && ngx_strncmp(r->header_name_start, "HTTP", 4) == 0)
	            {
	                state = sw_ignore_line;
	                break;
	            }
	
	            if (ch == '\0') {
	                return NGX_HTTP_PARSE_INVALID_HEADER;
	            }
	
	            r->invalid_header = 1;
	
	            break;
```



# 4.引用

1、<http://blog.csdn.net/wx_mdq/article/details/10466891>

2、<https://trac.nginx.org/nginx/browser/nginx/src/http/ngx_http_parse.c>