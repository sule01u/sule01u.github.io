---
layout:     post             
title:      构建webug测试环境              
date:       2020-04-17             
author:     suleo                  
header-img: img/post-bg-keybord.jpg  
catalog: true                      
tags:                              
    - 渗透测试 
---

# 前言

>  WeBug名称定义为”我们的漏洞”靶场环境 ，基础环境是基于PHP/mysql制作搭建而成，中级环境与高级环境分别都是由互联网漏洞事件而收集的漏洞存在的操作环境。在webug3.0发布后的四百多天226安全团队终于在大年初二发布了webug的4.0版本。
>
> Webug4.0官网地址：`https://www.webug.org`
> Webug4.0官方源码：`https://github.com/wangai3176/webug4.0`
> Webug4.0安装介绍：`https://www.freebuf.com/column/195521.html`
> 基于项目的源码搭建了一个Docker版本的Webug4.0，已经push到了Docker hub，欢迎大家下载来玩~
> 听很多小伙伴说有时候自己搭建的时候，由于php版本问题导致有些注入题做不了，这点我做Docker镜像的时候想到了，然后搭建完本地测试了一下，注入题没问题。

# 构建过程

首先是下载了Webug4.0版本的源码，然后编写Dockerfile
Dockerfile内容如下：

```
FROM ubuntu:trusty
MAINTAINER Area39@163.com
RUN echo "deb http://mirrors.aliyun.com/ubuntu/ trusty main restricted universe multiverse"> /etc/apt/sources.list
RUN apt-get update \
        && apt-get install -y mysql-server apache2 php5 php5-mysql
COPY sql /root/
RUN /etc/init.d/mysql start &&\
    mysql -e "grant all privileges on *.* to 'root'@'%' identified by 'toor';"&&\
    mysql -e "grant all privileges on *.* to 'root'@'localhost' identified by 'toor';"&&\
    mysql -u root -ptoor -e "show databases;" &&\
    mysql -u root -ptoor --default-character-set=utf8 </root/webug.sql &&\
    mysql -u root -ptoor -e "show databases;" &&\
    mysql -u root -ptoor </root/webug_sys.sql &&\
    mysql -u root -ptoor -e "show databases;" &&\
    mysql -u root -ptoor </root/webug_width_byte.sql &&\
    mysql -u root -ptoor -e "show databases;"
RUN sed -Ei 's/^(bind-address|log)/#&/' /etc/mysql/my.cnf \
	&& echo 'skip-host-cache\nskip-name-resolve' | awk '{ print } $1 == "[mysqld]" && c == 0 { c = 1; system("cat") }' /etc/mysql/my.cnf > /tmp/my.cnf \
	&& mv /tmp/my.cnf /etc/mysql/my.cnf
COPY webug /var/www/html
RUN rm /var/www/html/index.html &&\
 chown www-data:www-data /var/www/html -R &&\
 rm -rf /root/*
COPY httpd-foreground /usr/bin
EXPOSE 80
CMD ["httpd-foreground"]
```

在这里踩了个坑，Dockerfile第11行`mysql -u root -ptoor ，由于导入数据库文件时没指定字符集，所以直接乱码了。解决方法：`mysql -u root -ptoor --default-character-set=utf8 指定为utf8就可以了。

# 搭建方式

- 通过Dockerfile
- 通过Docker hub

## 通过Dockerfile搭建

项目已经传到了Github，[传送门](https://github.com/Area39/Webug4.0-Docker)
目录结构
[![img](https://tva4.sinaimg.cn/large/007DFXDhgy1g5nckxhmv5j30uk04uq38.jpg)](https://tva4.sinaimg.cn/large/007DFXDhgy1g5nckxhmv5j30uk04uq38.jpg)
然后在本目录下输入`docker build -t webug:4.0 .`
[![img](https://tva4.sinaimg.cn/large/007DFXDhgy1g5ncmy9y7fj316d0er401.jpg)](https://tva4.sinaimg.cn/large/007DFXDhgy1g5ncmy9y7fj316d0er401.jpg)
稍等片刻，你的Webug就搭建完成了。
启动Webug:4.0容器
`docker run -d -P webug:4.0`
[![img](https://tva3.sinaimg.cn/large/007DFXDhly1g5ncr0avthj314s06274n.jpg)](https://tva3.sinaimg.cn/large/007DFXDhly1g5ncr0avthj314s06274n.jpg)
然后访问ip+映射的端口，可以看到后台登录界面
[![img](https://tva3.sinaimg.cn/large/007DFXDhgy1g5ncp6v58fj31650kqgmd.jpg)](https://tva3.sinaimg.cn/large/007DFXDhgy1g5ncp6v58fj31650kqgmd.jpg)
默认账号：

- 后台管理员：admin/admin
- 数据库密码：root/toor

[![img](https://tva3.sinaimg.cn/large/007DFXDhgy1g5nctcdvatj31hc0sc77j.jpg)](https://tva3.sinaimg.cn/large/007DFXDhgy1g5nctcdvatj31hc0sc77j.jpg)

## 通过Docker hub搭建

为了大家方便使用，已经push到了docker hub上面。
从Docker hub上拉取镜像
`docker pull area39/webug`
[![img](https://tva2.sinaimg.cn/large/007DFXDhgy1g5nd0z95hyj30yf0bzdgu.jpg)](https://tva2.sinaimg.cn/large/007DFXDhgy1g5nd0z95hyj30yf0bzdgu.jpg)
启动过程
`docker run -d -P area39/webug`
[![img](https://tva3.sinaimg.cn/large/007DFXDhly1g5oprxekjnj314w052dg7.jpg)](https://tva3.sinaimg.cn/large/007DFXDhly1g5oprxekjnj314w052dg7.jpg)
此时你的Webug就能使用啦。
[![img](https://tva4.sinaimg.cn/large/007DFXDhgy1g5nd8tcujij31460ne0ti.jpg)](https://tva4.sinaimg.cn/large/007DFXDhgy1g5nd8tcujij31460ne0ti.jpg)
