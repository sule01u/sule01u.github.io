---
layout:     post             
title:      构建DVWA测试环境     
subtitle:   使用docker构建DVWA渗透测试环境         
date:       2020-04-17             
author:     suleo                  
header-img: img/post-bg-keybord.jpg    
catalog: true                      
tags:                              
    - 渗透测试   
---  

## docker搭建dvwa渗透测试平台（centos7）

1. **dvwa简介**
   DVWA（Damn Vulnerable Web Application）是一个用来进行安全脆弱性鉴定的PHP/MySQL Web应用，旨在为安全专业人员测试自己的专业技能和工具提供合法的环境，帮助web开发者更好的理解web应用安全防范的过程。

2. **安装**

   卸载旧版本较旧的Docker版本称为docker或docker-engine。如果已安装这些程序，请卸载它们以及相关的依赖项。

   ```
   sudo yum remove docker 
                  docker-client
                  docker-client-latest
                  docker-common
                  docker-latest
                  docker-latest-logrotate 
                  docker-logrotate
                  docker-engine
   ```

3. **设置存储库**

- 安装所需的软件包,yum-utils提供了yum-config-manager 效用，并device-mapper-persistent-data和lvm2由需要 devicemapper存储驱动程序

  ```
  sudo yum install -y yum-utils device-mapper-persistent-data lvm2
  ```

- 使用以下命令来设置稳定的存储库

  ```
  sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
  ```

4. **安装DOCKER ENGINE-社区**

- 安装最新版本的Docker Engine-Community和containerd，或者转到下一步安装特定版本:

  ```
  sudo yum install docker-ce docker-ce-cli containerd.io
  ```

- 如果提示您接受GPG密钥，请验证指纹是否匹配

  ```
  060A 61C5 1B55 8A7F 742B 77AA C52F EB6B 621E 9F35
  ```

​       如果是，则接受它。

5. **启动Docker**

   `sudo systemctl start docker`

- 通过运行hello-world 映像来验证是否正确安装了Docker Engine-Community 。

  ```
  sudo docker run hello-world
  ```

6. **使用**

1. 拉取dvwa镜像以及启动一个容器,容器命名为dvwa,容器内80端口映射到宿主机8080端口,web端访问的是8080端口

   ```
   docker run -p 8080:80 -v /opt/containerd/logs/dvwa_logs:/var/log/apache2 -it -d --name dvwa vulnerables/web-dvwa
   ```

2. 其他常用操作

   ```
   docker start dvwa
   docker restart dvwa
   docker stop dvwa
   ```

3. 使用

   1. 访问http://dvwa_ip:8080/(修改为自己的ip)
      用户名:admin
      密码:Password
      
   2. 点击创建数据库
      
      ![image-20200417220548851](/Users/shulei/Library/Application Support/typora-user-images/image-20200417220548851.png)
      
   3. 首页

      ![image-20200417220629497](/Users/shulei/Library/Application Support/typora-user-images/image-20200417220629497.png)