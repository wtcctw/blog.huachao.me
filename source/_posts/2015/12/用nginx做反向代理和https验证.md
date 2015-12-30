title:  "用nginx做反向代理和https验证"
date: 2015-12-29
updated: 2015-12-29
categories:
- network
tags:
- network
- nginx
---

> 没有nginx，90%的网站的所谓load balance和反向代理，都是玩不转的

## nginx是什么
nginx是一个高性能的Web和反向代理服务器。
这样说比较抽象，考虑一下nginx一般使用的场景就比较实际了。在大众点评，nginx部署在tomcat之前，负责tomcat集群的load balance，比如一个web应用被部署到3台tomcat上，那么会在这3台tomcat之前加一台nginx来做load balance以及同时做反向代理服务器。这样做的好处是，tomcat是Servlet容器，本身擅长的是业务逻辑的处理，高并发不行，在tomcat之前同nginx来抗住高并发的流量。同时，nginx在转发HTTP请求之前，会维持TCP长连接，并且知道HTTP请求完整到达nginx并且nginx将其log下来后，才会转发到tomcat。
![nginx的load balance功能和反向代理功能](/images/2015/12/nginx简图.png)

## nginx负载均衡配置和反向代理
upstream模块用来配置负载均衡的所有server信息，在server模块中server_name是HTTP请求头中的Server信息，也就是说HTTP请求头中如果是www.domain.com的，那么就用这个server模块处理。对于所有uri，这个server模块会反向代理到myserver上，myserver对应的就是负载均衡的配置，里面有3台机器。这样就实现了一个域名www.domain.com对应3台服务器（192.168.12.181/182/183）的负载均衡。
所谓反向代理，可以理解为配置多个location块，不同的location对应不同的uri，proxy_pass中将其路由到不同的内部服务器即可。
```bash
upstream  myserver {
    server   192.168.12.181:80 weight=3 max_fails=3 fail_timeout=20s;
    server   192.168.12.182:80 weight=1 max_fails=3 fail_timeout=20s;
    server   192.168.12.183:80 weight=4 max_fails=3 fail_timeout=20s;
}

server {
    listen       80;
    server_name  www.domain.com 192.168.12.189;
    #index index.htm index.html;
    #root  /var/web/www;

    location / {
        proxy_pass http://myserver;
        #proxy_next_upstream http_500 http_502 http_503 error timeout invalid_header;
        #include    /opt/nginx/conf/proxy.conf;
    }
}
```
### 关于反向代理时候的一些参数
最近在`netstat -anp`查看nginx的反向代理连接的时候发现非常多的TIME_WAIT。TIME_WAIT这个TCP状态是在服务端主动关闭的时候会到达的一个状态，且这个时候需要等待一段时间内核才能回收socket。那是什么原因导致的呢。Google了一下，发现nginx 1.1以下版本在反向代理的时候不是用长连接的，也就是说用的HTTP1.0，`Connetion: Close`的HTTP头，所以导致了hexo在相应之后主动去close，结果出现了很多TIME_WAIT。将反向代理改为如下，看到所有相关的参数都注释掉了，因为我用的nginx是1.0，所以无法使用反向代理长连接的feature~~
```bash
upstream hexo {
    server 127.0.0.1:4000;
    #keepalive 10;
}

#proxy & ssl
server {
    listen 443 ssl;
    server_name blog.huachao.me;
    ssl on;
    ssl_certificate /root/ssl.cert;
    ssl_certificate_key /root/ssl.key;
    location / {
        proxy_pass http://hexo;
        #proxy_http_version 1.1;
        #proxy_set_header Connection "";
    }
}
```

## nginx做https验证
- 关于https所需要的证书，可以到[StartSSL](https://www.startssl.com/)上申请，原理部分请移步[将网站打造为https](https://blog.huachao.me/2015/5/将网站打造为https/)。
- nginx的server模块配置443端口的监听，并且将证书，私钥信息也罗列完整
- nginx的server模块配置80端口，强制跳转到https
```bash
#redirect to https
server {
    listen 80;
    server_name blog.huachao.me;
    return 301 https://$server_name$request_uri;
}

#proxy & ssl
server {
    listen 443 ssl;
    server_name blog.huachao.me;
    ssl on;
    ssl_certificate /path/to/cert_file;
    ssl_certificate_key /path/to/private_key;
    location / {
        proxy_pass http://localhost:port;
    }
}
```
## 小结
nginx的配置项非常多，HTTP模块也非常复杂，在配置过程中我们首先抓住主要配置项，将功能搭建起来，然后在后期的学习运营维护的过程中一步步细化即可。否则只是过一遍配置项也意义不大。
