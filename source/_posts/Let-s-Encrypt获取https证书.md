---
title: 免费获取Let's Encrypt的SSL证书
date: 2018-02-02 23:56:13
tags: https 
---

### 环境介绍
本文所有操作都在centos7环境下操作.

首先给出两个关键的网站：Let's Encrypt的[官网](https://letsencrypt.org/), 以及本次安装过程的工具[cerbot](https://certbot.eff.org/).

### 安装cerbot
我们在[cerbot](https://certbot.eff.org/)首页，选择我们的环境（这里是nignx）和操作系统(centos7),如下图所示：
![](Jietu20180203-001634@2x.jpg)
在下方则会显示安装cerbot工具的操作过程：
``` bash
$ yum -y install yum-utils
$ yum-config-manager --enable rhui-REGION-rhel-server-extras rhui-REGION-rhel-server-optional
$ sudo yum install certbot-nginx
```
如果上述操作没有问题，那么cerbot工具就安装好了.

### 通过cerbot获取ssl/tls证书
这个过程十分简单，首先需要关闭nginx服务，否则执行不会成功：
``` bash
$ certbot certonly --standalone --email 你的邮箱 -d 你的域名
```
可以有多个-d选项，即可以指定多个域名，但必需是单域名，目前仍不支持泛域名.

如果执行成功，在终端会有成功提示，同时告诉你生成的证书文件存放的目录

### 配置nginx
在你的站点目录中增加配置一个server:

```
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    ssl on;

    ssl_certificate   /etc/letsencrypt/live/yucloud.top/fullchain.pem;
    ssl_certificate_key  /etc/letsencrypt/live/yucloud.top/privkey.pem;
    ssl_session_timeout 5m;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_prefer_server_ciphers on;

    # 其它配置...
}
```
ssl_certificate和ssl_certificate_key配置根据自己证书文件存放的位置自行调整.

### 更新证书
因为Let's Encrypt的证书有效期只有3个月，因此需要一个定时任务来更新证书文件, 每次更新证书文件之前都要关闭nginx服务器.

首先使用下面命令测试是否能够进行更新：
``` bash
$ sudo certbot renew --dry-run
```
没有问题，则可以执行下面命令进行更新：
``` bash
$ certbot renew
```
通过一个shell脚本, 使用crotab来实现定时更新，脚本内容如下:

``` bash
#!/bin/bash
# 关闭nginx
kill `cat /usr/local/nginx/logs/nginx.pid`
# 更新证书文件
sudo certbot renew
# 启动nginx
/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
```
此外，官方推荐每日执行两次该定时任务，以降低出现证书不可用的概率，如果证书未到期，执行更新指令并不会更新证书文件.
