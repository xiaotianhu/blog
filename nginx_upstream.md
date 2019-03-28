Date: 2019-03-27
Title: Nginx做负载均衡流量分配不均衡的一次故障分析
Tags:  nginx upstream 负载均衡 流量
Toc:no
Status: public
Position: 1

公司的一个业务,LNMP架构.前端SLB+2台NGINX做负载均衡,流量分发到后端五台ECS的WEB服务器上,WEB服务主要是PHP(PHP7.1+Laravel5.5)

之前流量不大的时候没有特别关注.最近流量起来了,开发同学手动添加了一个新的ECS节点,并加入负载均衡的Upstream池里面.结果这台机器始终是没有流量,其他机器的负载很高,也不给这台机器分配流量.很是诡异, 我觉得很有意思,花了两天时间终于研究明白问题了, 记录一下

先说表现,深入观察后发现,后端原来的五台机器,负载分配也不是很均衡.总是有1-2台机器很高,其他机器却很低的情况.查看Nginx主要配置如下

```
upstream web {
        server 192.168.1.66 weight=10;
        server 192.168.1.67 weight=10;
        server 192.168.1.63 weight=10;
        server 192.168.1.77 weight=10;
        server 192.168.1.78 weight=10;
        server 192.168.1.84 weight=10;#新添加

        check interval=100 fall=1 rise=3 timeout=5000 type=http default_down=false;
        check_http_send "HEAD /ok HTTP/1.0\r\n\r\n";
        check_http_expect_alive http_2xx;
}
server
{
    listen 80 ;
    server_name foo.bar.com;
    index index.html index.htm index.php;
    location / {
                add_header  X-Upstream  $upstream_addr;
                proxy_pass http://$web;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_next_upstream http_500 http_502 http_503 error timeout invalid_header;
                proxy_buffering on;
                proxy_redirect off;
                proxy_connect_timeout 300s;
                proxy_send_timeout 300s;
                proxy_read_timeout 300s;
                proxy_buffer_size 64k;
                proxy_buffers 4 64k;
                proxy_busy_buffers_size 64k;
                proxy_temp_file_write_size 64k;
                proxy_max_temp_file_size 1024m;
                proxy_ignore_client_abort on;
        }
    access_log  logs/access.log main;
}
```

做了很多猜测和验证,都没有结果.仔细分析负载均衡的配置,对着文档研究,发现这行配置很诡异:

```
proxy_next_upstream http_500 http_502 http_503 error timeout invalid_header;
```
查看官网的说明:
```
Syntax:	proxy_next_upstream error | timeout | invalid_header | http_500 | http_502 | http_503 | http_504 | http_403 | http_404 | http_429 | non_idempotent | off ...;
Default:	
proxy_next_upstream error timeout;
Context:	http, server, location
Specifies in which cases a request should be passed to the next server:
```

就是指定,在什么情况下,会把请求发给下一台机器重试.
cat一下accesslog,发现因为业务问题,有不少500的请求错误.
那就很好理解了,Nginx把请求先给A机器,发现有500,转给B,B出错继续给C,就导致了负载不均衡,总是有机器负载比较高.
至于为啥新加入的机器没有负载,可能还得看源码才能明白具体的算法了.

去掉这一行配置,果然立马好了.整个集群内不停重试的请求也少了,集群的负载瞬间下来了.看图感受一下

![](images/WX20190328-174541.png)

proxy_next_upstream 的使用还是很容易入坑的.想想集群平白浪费了很多性能就很蛋疼.运维同学在配置这个的时候估计也没有想到这个坑.
我们使用openresty,用upstream的check模块其实就能解决很大一部分问题了.




