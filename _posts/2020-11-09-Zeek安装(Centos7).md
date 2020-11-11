---
layout:     post             
title:      Zeek安装    
subtitle:   Zeek最方便快捷的安装方式      
date:       2020-11-05             
author:     suleo                  
header-img: img/post-bg-keybord.jpg    
catalog: true                      
tags:                              
    - 渗透测试   
---  

# Zeek安装指南
> 采用源安装，最方便管理的方式
### 依赖安装
1. 安装依赖
   - `sudo yum install cmake make gcc gcc-c++ flex bison libpcap-devel openssl-devel python-devel swig zlib-devel`
2. 安装gcc升级工具
  - 安装cmake3
  `yum -y install epel-release`
  `echo '[group_kdesig-cmake3_EPEL]
  name=Copr repo for cmake3_EPEL owned by @kdesig
  baseurl=https://copr-be.cloud.fedoraproject.org/results/@kdesig/cmake3_EPEL/epel-7-$basearch/
  type=rpm-md
  skip_if_unavailable=True
  gpgcheck=1
  gpgkey=https://copr-be.cloud.fedoraproject.org/results/@kdesig/cmake3_EPEL/pubkey.gpg
  repo_gpgcheck=0
  enabled=1
  enabled_metadata=1' >> /etc/yum.repos.d/cmake3.repo`
  `yum install -y cmake3`
  - 安装devtoolset-7
  `yum install centos-release-scl`
  `yum install devtoolset-7`
  `scl enable devtoolset-7 bash`

3. 添加zeek软件源

   `cd /etc/yum.repo.d/`

   `wget https://download.opensuse.org/repositories/security:zeek/CentOS_7/security:zeek.repo`

   `yum install zeek-lts`

   默认安装到/opt下

4. 设置系统环境变量

   `vim ~/.bashrc`

   `export PATH=/opt/zeek/bin:$PATH`

