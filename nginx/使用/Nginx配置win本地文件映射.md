# Nginx配置win本地文件映射



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
        listen       8088;
        server_name  localhost;

        location / {
            root E://Users/upload;
        }
    }
}
```

