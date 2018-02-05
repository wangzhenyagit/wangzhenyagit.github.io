---
layout: post
title: Nginx与HAproxy对比测试
category: 性能测试
tags: Nginx HAproxy
keywords: Nginx HAproxy
---

## 测试目的
此次测试只在验证ngnix反向代理服务器的性能，客户并发最多在5000左右，客户请求为http协议，post方式数据大概在1k左右，吐吞量大概要求2w左右。
经验证，并发上对于nginx不算大，官方给出为最大5w并发，一般实际中能有2w并发，主要是吐吞量上。

## 配置
### ngnix配置
#### window下的限制
- 只能使用一个cpu，无法发挥多核心服务器性能
- 一个work只能最多处理1024个连接，并发有影响
- 使用select而不是I/O completion ports
实际测试也发现，在windows上，只能利用一个cpu核心，而且但cpu非常容易
参考：http://nginx.org/en/docs/windows.html

#### linux下优化配置
配置可以分为两部分，一是在模块之上，对整个程序的配置，一个是对各个模块的配置。
有几个比较影响性能的配置需要修改下，有几个非常重要的参数已经默认是最优配置，比如已经使用了默认的epoll（在centos下），http模块中的缓存配置、keepalive_timeout时间、sendfile
开启、tcp_nopush开启等;

#### 其他的需要修改的有如下几个地方：
##### worker_processes 
配置使用的cpu数目和cpu绑定，无论作为代理还是web服务器，都需要根据机器的cpu信息进行配置。
```
worker_processes  8;  
worker_cpu_affinity 10000000 01000000 00100000 00010000 00001000 00000100 00000010 00000001;
```

##### worker_connections
events模块，配置表示每个工作进程的并发连接数，默认设置为1024，在events模块下。
```
events {
    worker_connections  65535;
}
```

##### keepalive
http_upstream_module 模块，Activates the cache for connections to upstream servers.
与upstream保持长连接的数目。
```
upstream myServer {
   server 10.45.157.90:80;
   server 10.45.157.81:80;
   server 10.45.157.31:80;
   server 10.45.157.31:8081;
   server 10.45.157.82:80;
   server 10.45.157.225:80;
   server 10.45.157.225:8081;
   server 10.45.157.225:8083;
   server 10.45.157.225:8084;
   server 10.45.157.225:8085;
   server 10.45.157.225:8086;
   server 10.45.157.225:8087;
   server 10.45.157.225:8088;
   keepalive 32;
}
```

##### access_log
记录请求的log，默认是打开的，可以在相应模块中给指定关闭，例如http模块中关闭，但实测对性能影响并不大，关闭前最后经过压力测试验证。
```
http {
	...
    sendfile       on;
    tcp_nopush     on;
    access_log off;	
	...
}
```

其他配置如静态的配置缓存由于此次测试没有涉及到可以缓存的内容如各种格式图片、css和js代码等没有进行配置，此外Gzip压缩参数也可以配置，但这次测试主要是post大约1k的数据，数据量较少，对带宽影响不大，压缩意义不是很大。

###  负载均衡端的配置
```
#user  nobody;
worker_processes  8;  
worker_cpu_affinity 10000000 01000000 00100000 00010000 00001000 00000100 00000010 00000001;

#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

#pid        logs/nginx.pid;


events {
    worker_connections  102400;
}
stream {
access_log off;
upstream http_backend {
server 10.45.157.81:80 max_fails=2 fail_timeout=10s weight=1;
server 10.45.157.90:80 max_fails=2 fail_timeout=10s weight=1;
least_conn;
}
server {
listen 90;
proxy_next_upstream on;
proxy_next_upstream_timeout 0;
proxy_next_upstream_tries 0;
proxy_connect_timeout 1s;
proxy_timeout 1m;
proxy_upload_rate 0;
proxy_download_rate 0;
proxy_pass http_backend;
}
}
http {
    include       mime.types;
    default_type  application/octet-stream;

	upstream myServer {
	   server 10.45.157.90:80;
	   server 10.45.157.81:80;
	   server 10.45.157.31:80;    #window主机
	   server 10.45.157.31:8081;  #window主机
	   server 10.45.157.82:80;    #window主机
	   server 10.45.157.225:80;   #window主机
	   server 10.45.157.225:8081; #window主机
	   server 10.45.157.225:8083; #window主机
	   server 10.45.157.225:8084; #window主机
	   server 10.45.157.225:8085; #window主机
	   server 10.45.157.225:8086; #window主机
	   server 10.45.157.225:8087; #window主机
	   server 10.45.157.225:8088; #window主机
	   keepalive 256;
	}
    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log  logs/access.log  main;

    sendfile        on;
    tcp_nopush     on;
    access_log off;	

    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;

    server {
        listen       80;
        server_name  localhost;

        #charset koi8-r;

        #access_log  logs/host.access.log  main;

        location / {
            root   html;
            index  index.html index.htm;
		proxy_pass http://myServer;
        }

        #error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }


    }

}
```

### web服务端的配置
```
#user  nobody;
worker_processes  8;  
worker_cpu_affinity 10000000 01000000 00100000 00010000 00001000 00000100 00000010 00000001;

#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

#pid        logs/nginx.pid;

worker_rlimit_nofile 100000;
events {
    use epoll;
    worker_connections  102400;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  off;

    sendfile        on;
    tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;

    server {
        listen       80;
        server_name  localhost;

        #charset koi8-r;

        #access_log  logs/host.access.log  main;

        location / {
            root   html;
            index  index.html index.htm;
        }

        #error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }

}
```

### 测试记录

#### 测试环境
测试环境都是虚拟机，cpu型号，Intel(R) Xeon(R) CPU E5-2630 v3 @ 2.40GHz，因为没有测试高并发情况，此次测试主要关心吞吐量，内存影响不大。虚拟机所在主机均为千兆网卡，网络IO也影响不大。

10.45.157.64 centos64位，8核心cpu
10.45.157.81 centos64位，8核心cpu
10.45.157.90 centos64位，8核心cpu
10.45.157.31 windows2008 64位 双核
10.45.157.81 windows2008 64位 双核
10.45.157.225 windows2008 64位 八核

组网如下：
10.45.157.64 负载均衡服务器
10.45.157.81 ngnix web server（centos）
10.45.157.90 ngnix web server（centos）
10.45.157.225 ngnix web server（windows 起了8个）
10.45.157.31 ngnix web server（windows 起了2个）
10.45.157.81 ngnix web server（windows 起了2个）

由于测试目前对同时在线连接数要求不高，5000左右，对吞吐量要求较高，测试使用简单ab测试，测试过程中发现，短链接情况，短链接使用500个并发，ngnix的处理速度最快，而长连接情况并发数目影响不大，从1000并发到5000并发（用5个ab，分布在3台设备上模拟）ngnix吞吐量影响不大。

#### 测试结果
注意，web Server端的服务页面为ngnix的默认主页，大小173个字节。

##### ngnix单机windows性能测试
10.45.157.225 ngnix web server 短链接情况，ab测试，吐吞量6000左右，单核cpu已经用满，cpu瓶颈。
```
ab -c 100 -n 100000 -T 'application/x-www-form-urlencoded' -p postdata1.txt http://10.45.157.225:80/index.html

Concurrency Level:      100
Time taken for tests:   15.782 seconds
Complete requests:      100000
Failed requests:        0
Write errors:           0
Non-2xx responses:      100000
Total transferred:      32500000 bytes
Total body sent:        16700000
HTML transferred:       17300000 bytes
Requests per second:    6336.19 [#/sec] (mean)
Time per request:       15.782 [ms] (mean)
Time per request:       0.158 [ms] (mean, across all concurrent requests)
Transfer rate:          2011.00 [Kbytes/sec] received
                        1033.34 kb/s sent
                        3044.34 kb/s total
```						
10.45.157.225 ngnix web server 长链接情况，ab测试，吐吞量8000左右，单核cpu已经用满，cpu瓶颈。
```
ab -c 100 -n 100000 -T 'application/x-www-form-urlencoded' -p postdata1.txt http://10.45.157.225:80/index.html

Concurrency Level:      100
Time taken for tests:   11.618 seconds
Complete requests:      100000
Failed requests:        0
Write errors:           0
Non-2xx responses:      100001
Keep-Alive requests:    99050
Total transferred:      32995580 bytes
Total POSTed:           19619796
HTML transferred:       17300173 bytes
Requests per second:    8607.47 [#/sec] (mean)
Time per request:       11.618 [ms] (mean)
Time per request:       0.116 [ms] (mean, across all concurrent requests)
Transfer rate:          2773.52 [Kbytes/sec] received
                        1649.19 kb/s sent
                        4422.71 kb/s total
```						
##### ngnix单机linux性能测试
10.45.157.81 ngnix web server（centos）短链接情况，吐吞量有3.5w左右，8核心大概每个用了50%
```
ab -c 500 -n 1000000 -T 'application/x-www-form-urlencoded' -p postdata1.txt http://10.45.157.81:80/index.html

Concurrency Level:      500
Time taken for tests:   27.519 seconds
Complete requests:      1000000
Failed requests:        0
Write errors:           0
Non-2xx responses:      1000243
Total transferred:      325078975 bytes
Total POSTed:           171041553
HTML transferred:       173042039 bytes
Requests per second:    36338.33 [#/sec] (mean)
Time per request:       13.760 [ms] (mean)
Time per request:       0.028 [ms] (mean, across all concurrent requests)
Transfer rate:          11535.96 [Kbytes/sec] received
                        6069.69 kb/s sent
                        17605.66 kb/s total
```						
10.45.157.81 ngnix web server（centos）长链接情况，吐吞量有9w左右，8核心大概每个用了50%
```
ab -c 500 -k -n 1000000 -T 'application/x-www-form-urlencoded' -p postdata1.txt http://10.45.157.81:80/index.html 

Concurrency Level:      500
Time taken for tests:   10.758 seconds
Complete requests:      1000000
Failed requests:        0
Write errors:           0
Non-2xx responses:      1000115
Keep-Alive requests:    990238
Total transferred:      329989115 bytes
Total POSTed:           195118560
HTML transferred:       173019895 bytes
Requests per second:    92955.66 [#/sec] (mean)
Time per request:       5.379 [ms] (mean)
Time per request:       0.011 [ms] (mean, across all concurrent requests)
Transfer rate:          29955.43 [Kbytes/sec] received
                        17712.28 kb/s sent
                        47667.71 kb/s total
```						
##### ngnix七层负载均衡服务器性能测试
10.45.157.64 负载均衡服务器，1.3w左右
短链接情况
```
ab -c 500 -n 1000000 -T 'application/x-www-form-urlencoded' -p postdata1.data http://10.45.157.64:80/index.html
Concurrency Level:      500
Time taken for tests:   74.557 seconds
Complete requests:      1000000
Failed requests:        0
Write errors:           0
Non-2xx responses:      1000000
Total transferred:      325000000 bytes
Total body sent:        166000000
HTML transferred:       173000000 bytes
Requests per second:    13412.60 [#/sec] (mean)
Time per request:       37.278 [ms] (mean)
Time per request:       0.075 [ms] (mean, across all concurrent requests)
Transfer rate:          4256.93 [Kbytes/sec] received
                        2174.31 kb/s sent
                        6431.24 kb/s total
```
10.45.157.64 负载均衡服务器，3w左右				
长链接情况
```
ab -c 500 -k -n 1000000 -T
'application/x-www-form-urlencoded' -p postdata1.txt http://10.45.157.64:80/index.html						
Concurrency Level:      500
Time taken for tests:   31.261 seconds
Complete requests:      1000000
Failed requests:        0
Write errors:           0
Keep-Alive requests:    990234
Total transferred:      850024255 bytes
HTML transferred:       612052632 bytes
Requests per second:    31989.22 [#/sec] (mean)
Time per request:       15.630 [ms] (mean)
Time per request:       0.031 [ms] (mean, across all concurrent requests)
Transfer rate:          26554.31 [Kbytes/sec] received
```			
##### ngnix四层负载均衡服务器性能测试

10.45.157.64 负载均衡服务器	短链接1.8w左右
```
Concurrency Level:      500
Time taken for tests:   54.117 seconds
Complete requests:      1000000
Failed requests:        0
Write errors:           0
Total transferred:      845256880 bytes
HTML transferred:       612186048 bytes
Requests per second:    18478.61 [#/sec] (mean)
Time per request:       27.058 [ms] (mean)
Time per request:       0.054 [ms] (mean, across all concurrent requests)
Transfer rate:          15253.10 [Kbytes/sec] received
```

10.45.157.64 负载均衡服务器	长链接6w左右，大概为七层的两倍
```
ab -c 500 -k -n 1000000 -T 'application/x-www-form-urlencoded' -p postdata1.txt http://10.45.157.64:90/index.html

Concurrency Level:      1000
Time taken for tests:   15.794 seconds
Complete requests:      1000000
Failed requests:        0
Write errors:           0
Non-2xx responses:      1000164
Keep-Alive requests:    990398
Total transferred:      330006065 bytes
Total POSTed:           198226512
HTML transferred:       173028372 bytes
Requests per second:    63314.78 [#/sec] (mean)
Time per request:       15.794 [ms] (mean)
Time per request:       0.016 [ms] (mean, across all concurrent requests)
Transfer rate:          20404.55 [Kbytes/sec] received
                        12256.51 kb/s sent
                        32661.06 kb/s total
```						
##### HAproxy负载均衡测试七层负载均衡测试
配置：
```
global
    daemon
    maxconn 10000
    ulimit-n 100000
    user haproxy
    group haproxy
    chroot /var/empty
    pidfile /var/run/haproxy.pid
    log localhost local0 notice
    nbproc 8

defaults
    mode http
    timeout connect 5000ms
    timeout client 50000ms
    timeout server 50000ms
    option http-keep-alive

frontend http-front
    bind *:80
    option httpclose
    maxconn 65536
    default_backend default-servers

backend default-servers
        balance roundrobin
    server default-servers80 10.45.157.81:80 maxconn 10240
    server default-servers90 10.45.157.90:80 maxconn 10240
    server default-servers31-1 10.45.157.31:80 maxconn 10240
    server default-servers31-2 10.45.157.31:8081 maxconn 10240
    server default-servers82-1 10.45.157.82:80 maxconn 10240
    server default-servers82-2 10.45.157.82:8080 maxconn 10240
    server default-servers225-1 10.45.157.225:80 maxconn 10240
    server default-servers225-2 10.45.157.225:8081 maxconn 10240
    server default-servers225-3 10.45.157.225:8083 maxconn 10240
    server default-servers225-4 10.45.157.225:8084 maxconn 10240
    server default-servers225-5 10.45.157.225:8085 maxconn 10240
    server default-servers225-6 10.45.157.225:8086 maxconn 10240
    server default-servers225-7 10.45.157.225:8087 maxconn 10240
    server default-servers225-8 10.45.157.225:8088 maxconn 10240
listen status
     bind 0.0.0.0:1080
     mode http
     log global
     stats refresh 10s
     stats uri /admin?stats
     stats realm Private lands
     stats auth admin:admin
```

执行和结果，不管带不带k，服务端怎么配置，结果大概都是2.4w
```
ab -c 500 -n 1000000 -T 'application/x-www-form-urlencoded' -p postdata1.txt http://10.45.157.64:80/index.html

Concurrency Level:      500
Time taken for tests:   41.553 seconds
Complete requests:      1000000
Failed requests:        0
Write errors:           0
Total transferred:      845003380 bytes
HTML transferred:       612002448 bytes
Requests per second:    24065.66 [#/sec] (mean)
Time per request:       20.776 [ms] (mean)
Time per request:       0.042 [ms] (mean, across all concurrent requests)
Transfer rate:          19858.95 [Kbytes/sec] received
```
##### HAproxy四层负载均衡测试
配置
```
frontend tcp_front
bind *:90
mode tcp
log global
option tcplog
timeout client 300s
backlog 4096
maxconn 65000
default_backend tcp_behind

backend tcp_behind
mode  tcp
option tcplog
option log-health-checks
balance roundrobin
server s1 10.45.157.81:80
server s2 10.45.157.90:80

长连接 ab -c 500 -k -n 1000000 -T 'application/x-www-form-urlencoded' -p postdata1.txt http://10.45.157.64:90/index.html
Concurrency Level:      500
Time taken for tests:   15.025 seconds
Complete requests:      1000000
Failed requests:        0
Write errors:           0
Keep-Alive requests:    990236
Total transferred:      849957120 bytes
HTML transferred:       612004284 bytes
Requests per second:    66555.58 [#/sec] (mean)
Time per request:       7.513 [ms] (mean)
Time per request:       0.015 [ms] (mean, across all concurrent requests)
Transfer rate:          55243.54 [Kbytes/sec] received

短链接  ab -c 500 -n 1000000 -T 'application/x-www-form-urlencoded' -p postdata1.txt http://10.45.157.64:90/index.html
Concurrency Level:      500
Time taken for tests:   44.290 seconds
Complete requests:      1000000
Failed requests:        0
Write errors:           0
Total transferred:      845245050 bytes
HTML transferred:       612177480 bytes
Requests per second:    22578.32 [#/sec] (mean)
Time per request:       22.145 [ms] (mean)
Time per request:       0.044 [ms] (mean, across all concurrent requests)
Transfer rate:          18636.93 [Kbytes/sec] received
```
## 监控功能对比
这个功能上Haproxy比Nginx还是好很多。
Nginx监控页面

<img src="https://raw.githubusercontent.com/wangzhenyagit/wangzhenyagit.github.io/master/images/tech/2017-09-05-nginx-mointor-page.png">

Haproxy监控页面
<img src="https://raw.githubusercontent.com/wangzhenyagit/wangzhenyagit.github.io/master/images/tech/2017-09-05-haproxy-mointor-page.png" height="100%" width="100%">

## 测试结论
- 不论是ngnix还是HAproxy在并发数目要求上都能满足5000左右的并发。
- 经测试性能上ngnix与HAproxy相差不大，但网上一般结论是HAproxy的性能比nginx高。
- 请求客户端与复杂均衡服务器之间是否保持长连接对吐吞量有比较大的影响，ngnix在七层吞吐量有1.5w左右，长连接为3w左右。
- 吞吐量上如果使用七层长连接（或四层长连接）方式基本能满足要求（5000设备每秒5次上报2.5w的吞吐量），但测试过程中有单热点问题，而且请求处理速度很快，单机处理性能未知，可能与实际有一定偏差。如果需要更进一步准确的数据，可在单机性能确定，部署实验环境后进行再次测试。
- 如果请求发送使用短链接的方式，由于可以配置商汤服务器的上报主机地址，可以使用两台服务器作为负载均衡服务器。
- HAproxy相比ngnix，监控功能比较完善，见附件截图。
- Ngnix二次开发相比HAproxy较容易

总体上看，在要求吐吞量为2.5w的时候，如果是商汤服务器长连接方式方式发送请求，那么单台负载均衡（ngnix或HAproxy）能够满足要求，如果是短链接情况下，在机器性能较差可能需要两台设备作为负载均衡使用。





