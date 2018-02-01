---
title: 搭建Hexo博客
date: 2018-02-01 15:44:17
tags:
---

### 安装Hexo
假设电脑上已安装了Nodejs以及git, 没有安装请自行Google

``` bash
$ npm install hexo-cli -g
```

初始化一个名为fpgayu的项目（blog的名字可以根据个人喜好自行定义） 

``` bash
$ hexo init fpgayu
```

进入项目根目录, 安装相关依赖

```
$ npm install
```

### 更换主题
用户可以根据个人喜好更换主题, 首先进入fpgayu/themes目录, 使用git获取一个自己喜欢的主题

``` bash
$ git clone https://github.com/CodeDaraW/Hacker
```
同时修改根目录下_config.yml文件, 将theme一项设置为自己的主题，这里是Hacker

### 在本地查看效果

在根目录下, 执行下面指令，启动一个本地服务

``` bash
$ hexo generate
$ hexo server
```

### 配置Nginx服务器

nginx 安装过程这里旧不再赘述，可以自行搜索, 在nginx中配置一个server

```
server {
        listen 80;
        listen [::]:80;

        root /var/www/fpgayu/html;
        index index.html index.htm index.nginx-debian.html;
        server_name example.com www.example.com;
        location / {
                try_files $uri $uri/ =404;
        }
}
```

在这里我们可以看到nginx将会读取/var/www/fpgayu/html目录下的html文件返回给浏览器进行渲染, 所以我们只需要将我们的代码部署到对应的目录中既可

为了实现自动化部署，需要在服务器端配置一个git仓库

```
$ cd ~
$ mkdir fpgayu.git
$ cd fpgayu.gi
$ git init --bare
```

在/root/fpgayu.git/hooks中添加一个post-receive的脚本

``` bash
$ vi /root/fpgayu.git/hooks/post-receive
```

内容如下(该脚本来自: [链接](https://eliyar.biz/how_to_build_hexo_blog/))：

``` bash
#!/bin/bash
GIT_REPO=/root/fpgayu.git
TMP_GIT_CLONE=/tmp/fpgayu
PUBLIC_WWW=/var/www/fpgayu/html #网站目录
rm -rf ${TMP_GIT_CLONE}
git clone $GIT_REPO $TMP_GIT_CLONE
rm -rf ${PUBLIC_WWW}/*
cp -rf ${TMP_GIT_CLONE}/* ${PUBLIC_WWW}
```

### 部署上线

在部署上线之前，需要修改_config.yml配置文件

```
deploy:
  type: git
  message: update
  repo:
      s1: root@服务器ip地址:/root/fpgayu.git/,master
```

在根目录安装git部署工具

```
$  npm install hexo-deployer-git --save
```

一键部署
``` bash
$ hexo deploy
```

