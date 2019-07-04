# NGINX Configuration

### Verify nginx configuration

```
sudo nginx -t

nginx: the configuration file /usr/local/etc/nginx/nginx.conf syntax is ok
nginx: configuration file /usr/local/etc/nginx/nginx.conf test is successful
```

### Sample nginx configuration
There are two server configurations here:

A reverse proxy that routes requests arrving on port 80 to a docker container listening on port 8080 of the host machine. Note that port 8080 of the host machine is mapped to port 80 of the docker container

A load balanced configuration that routes traffic arriving on port 81 to 4 docker containers listening on port range 8081-8084. Notice that we are using a log_format directive "fishball" in the access_log directive to customise the output of the nginx access log. 

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
                proxy_pass http://127.0.0.1:8080;
        }
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }

    log_format  fishball '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for" "$upstream_addr" "$upstream_connect_time" "$request_time"';
    
    upstream fishball_servers {
        server 127.0.0.1:8081;      # fishball-no-nginx-test listens to port 8081
        server 127.0.0.1:8082;      # fishball-no-nginx-test-2 listens to port 8082
        server 127.0.0.1:8083;      # fishball-no-nginx-test-3 listens to port 8083
        server 127.0.0.1:8084;      # fishball-no-nginx-test-4 listens to port 8084
    }

    server {
        access_log /usr/local/var/log/nginx/fishball_servers.access.log fishball;
        error_log /usr/local/var/log/nginx/fishball_servers.error.log;
        
        listen 81;
        server_name localhost;
        location / {
            proxy_set_header   X-Real-IP $remote_addr;
            proxy_set_header   Host      $http_host;
            proxy_pass         http://fishball_servers;
        }   
    }

    include servers/*;
}

```

Note that the $upstream_addr specified in the log_format directive allows us to see exactly which docker container each request is being routed to. See [Module ngx_http_upstream_module](http://nginx.org/en/docs/http/ngx_http_upstream_module.html)

```
127.0.0.1 - - [03/Jul/2019:13:25:58 +0800] "GET / HTTP/1.1" 200 1455 "-" "curl/7.63.0" "-" "127.0.0.1:8081"
127.0.0.1 - - [03/Jul/2019:13:25:59 +0800] "GET / HTTP/1.1" 200 1455 "-" "curl/7.63.0" "-" "127.0.0.1:8082"
127.0.0.1 - - [03/Jul/2019:13:26:15 +0800] "GET / HTTP/1.1" 200 1455 "-" "curl/7.63.0" "-" "127.0.0.1:8083"
127.0.0.1 - - [03/Jul/2019:13:26:16 +0800] "GET / HTTP/1.1" 200 1455 "-" "curl/7.63.0" "-" "127.0.0.1:8084"
127.0.0.1 - - [03/Jul/2019:13:26:17 +0800] "GET / HTTP/1.1" 200 1455 "-" "curl/7.63.0" "-" "127.0.0.1:8081"
127.0.0.1 - - [03/Jul/2019:13:26:18 +0800] "GET / HTTP/1.1" 200 1455 "-" "curl/7.63.0" "-" "127.0.0.1:8082"
127.0.0.1 - - [03/Jul/2019:13:26:19 +0800] "GET / HTTP/1.1" 200 1455 "-" "curl/7.63.0" "-" "127.0.0.1:8083"
127.0.0.1 - - [03/Jul/2019:13:26:20 +0800] "GET / HTTP/1.1" 200 1455 "-" "curl/7.63.0" "-" "127.0.0.1:8084"

```

It is also important to note that when we access http://localhost:81, four individual GET requests are observed in the logs and each of these GET requests are routed to different docker containers:

```
127.0.0.1 - - [04/Jul/2019:09:12:39 +0800] "GET / HTTP/1.1" 200 1455 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10.12; rv:67.0) Gecko/20100101 Firefox/67.0" "-" "127.0.0.1:8083" "0.000" "0.005"
127.0.0.1 - - [04/Jul/2019:09:12:39 +0800] "GET /js/app.c7d10227.js HTTP/1.1" 200 43917 "http://localhost:81/" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10.12; rv:67.0) Gecko/20100101 Firefox/67.0" "-" "127.0.0.1:8081" "0.000" "0.006"
127.0.0.1 - - [04/Jul/2019:09:12:39 +0800] "GET /js/chunk-vendors.009c6101.js HTTP/1.1" 200 97039 "http://localhost:81/" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10.12; rv:67.0) Gecko/20100101 Firefox/67.0" "-" "127.0.0.1:8084" "0.000" "0.012"
127.0.0.1 - - [04/Jul/2019:09:12:39 +0800] "GET /favicon.ico HTTP/1.1" 200 1150 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10.12; rv:67.0) Gecko/20100101 Firefox/67.0" "-" "127.0.0.1:8082" "0.000" "0.006"
```

To fix this we probably have to rely on NGINX Plus' [Session Persistence](https://www.nginx.com/products/nginx/load-balancing/) feature or simply pick a more appropriate tool for load balancing like HA Proxy.