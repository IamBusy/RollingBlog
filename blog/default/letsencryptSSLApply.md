title: letsencrypt HTTPS SSL 证书在不影响生产的情况下申请
date: 2016-04-06 16:22:14
categories: 技术
tags: [letsencrypt,docker,nginx]
---

## VERSION: 1

## TODO:

 自动续期

![](https://letsencrypt.org/images/letsencrypt-logo-horizontal.svg)

## 背景

在线上运行的nginx web service 需要持续不间断的做生产服务，这时候证书需要更新，`LE`又需要独立占用一个端口这里使用webroot的方式实现证书申请。
同时为了不占用线上服务器的端口，并且发挥docker的优势不需要装太多的软件。因此做了这个法案。

## 部署结构

```
           反向代理             文件共享
线上Nginx ----------> LE nginx ---------> LE container
```

## 准备nginx配置文件

``` shell
docker run -it --rm -v /root/:/data/ nginx cp -rf /etc/nginx /data/le
mkdir /root/le/static
echo a >> /root/le/statuc/index.html
```

## LE nginx

### 配置文件

```
server {
    listen       80;
    server_name  localhost;

    location / {
        root   /etc/nginx/static/;
        index  index.html index.htm;
    }
}
```

### 运行方法
```
docker run -it -d --name lenginx -p 8989:80 -v /root/le/:/etc/nginx/ --net=aaa nginx
```

### 测试

```
# curl http://127.0.0.1:8989/
a
```

如果测试结果有返回自己输入的hml内容即为成功。

## 域名配置

开始配置目标域名A 记录到服务器

## 线上服务Nginx 配置调整：

在 server 配置中加入如下location配置

```
location /.well-known {
    proxy_pass http://lenginx.aaa;
}
```

## 以容易的方式运行LetsEncrypt

```
docker run -it -d --name le -v /root/le/static/:/webroot index.tenxcloud.com/philo/le:0 /bin/bash
```

### 在LE容器中申请证书

```
letsencrypt-auto certonly -a webroot --webroot-path=/webroot -d example.com -d www.example.com
```

## 总结

当使用LE申请证书的时候，LE的服务器（我抓到的IP地址 `66.133.109.36`）会请求你的域名到类似这种地址位置：`/.well-known/acme-challenge/j3r4u4GEXbYIqjbGotSqbBRNp_3sohuzZw_G5Aw1lcI` 所以这里在线上服务器使用反向代理到LE共享磁盘位置的nginx中，是的LE服务器访问我们的服务器授权正常。就ok了。
