---
layout:     post             
title:      nessus破解版安装    
subtitle:   nessus破解版安装       
date:       2020-05-01             
author:     suleo                  
header-img: img/post-bg-keybord.jpg    
catalog: true                      
tags:                              
    - 渗透测试   
---  

# Nessus破解版安装

> Nessus号称是世界上最流行的漏洞扫描程序，全世界有超过75000个组织在使用它。该工具提供完整的主机漏洞扫描服务，并随时更新其漏洞数据库。

## 一、流程图

1. 安装破解流程

![img](https://img2020.cnblogs.com/blog/2019081/202004/2019081-20200429102712121-798023580.png) 2. 更新规则库流程图

![img](https://img2020.cnblogs.com/blog/2019081/202004/2019081-20200429102813535-1392735593.png)

安装下载的软件都是来自官网的，破解无需借助第三方工具，很干净。

规则库的链接可以重复使用，每次都是下载最新的版本。

linux系统和windows系统的破解都一样，只是把两个文件里的home改成ProfessionalFeed而已。



## 二、软件包下载

Nessus官网下载地址：https://www.tenable.com/downloads/nessus

根据系统下载不同的nessus版本

![img](https://img2020.cnblogs.com/blog/2019081/202004/2019081-20200428150946202-813764955.png)

## 三、安装

1. Kali中安装，包下载在哪个目录下就在哪个目录执行：

   `dpkg -i Nessus-8.10.0-debian6_amd64.deb`

   ![img](https://img2020.cnblogs.com/blog/2019081/202004/2019081-20200428151622356-297469886.png)  

2. 启动nessus

   `service nessusd start`

3. 访问nessus

   在浏览器中访问https://localhost:8834，初始化扫描器

   选择Managed Scanner→Managed by Tenable.sc，点击 Continue。
   ![img](https://img2020.cnblogs.com/blog/2019081/202004/2019081-20200428152036702-1335117956.png)
   ![img](https://img2020.cnblogs.com/blog/2019081/202004/2019081-20200428152109869-529652535.png)

4. 现在会要求新建账号，自己记住账号密码
   ![img](https://img2020.cnblogs.com/blog/2019081/202004/2019081-20200428152343627-690325526.png)

5. 等待初始化完成。完成后登陆进去是没有扫描界面的。
   ![img](https://img2020.cnblogs.com/blog/2019081/202004/2019081-20200428152407645-543018994.png)

**接下来获取插件包**

1. kali中默认把nessus安装在/OPT目录下, 进入目录, 执行以下操作, 复制并记录challenge code

   `/opt/nessus/sbin/nessuscli fetch --challenge`
   ![img](https://img2020.cnblogs.com/blog/2019081/202004/2019081-20200428152826175-1503482077.png)

2. 访问上面输出的网址https://plugins.nessus.org/v2/offline.php，把challenge code填入第一个框：
   ![img](https://img2020.cnblogs.com/blog/2019081/202004/2019081-20200428153039783-1305279160.png)

3. 接下来获取第二个框的激活码，访问网站https://zh-cn.tenable.com/products/nessus/nessus-essentials，姓名随便写，邮箱写真实邮箱，用来接受激活码：
   ![img](https://img2020.cnblogs.com/blog/2019081/202004/2019081-20200428153420451-1737188047.png)

4. 点击注册后，过大约一分钟左右，邮箱收到邮件，找到激活码，复制：
   ![img](https://img2020.cnblogs.com/blog/2019081/202004/2019081-20200428153622027-435821379.png)

5. 把激活码贴到第二个框，点submit：
   ![img](https://img2020.cnblogs.com/blog/2019081/202004/2019081-20200428153736230-608105450.png)

6. 注册成功后网页返回更新包的下载链接，在浏览器输入上述链接就可以下载最新插件包：
   ![img](https://img2020.cnblogs.com/blog/2019081/202004/2019081-20200428153825235-1424528360.png)

7. 注册包下载完成后，执行更新操作：

   `/opt/nessus/sbin/nessuscli update all-2.0.tar.gz`
   ![img](https://img2020.cnblogs.com/blog/2019081/202004/2019081-20200428154113962-1808120566.png)



## 四、破解

> 到现在为止，nessus安装完成，但只支持16个IP

1. 接下来进行破解，修改下面两个文件，没有的话创建一下，再改成下面的内容。	

```shell
# 文件路径
/opt/nessus/lib/nessus/plugins/plugin_feed_info.inc
/opt/nessus/var/nessus/plugin_feed_info.inc
```

```shell
# 文件内容
PLUGIN_SET = "202006272346";
PLUGIN_FEED = "ProfessionalFeed (Direct)";

PLUGIN_FEED_TRANSPORT = "Tenable Network Security Lightning";
```

![img](https://img2020.cnblogs.com/blog/2019081/202004/2019081-20200428155646049-1657641494.png)

2. 重启nessus
   `service nessusd restart`

> ⚠️注意：每次更新完后，上述两个文件都会变回家庭版的配置（因为我们是通过下载家庭版的插件包来进行离线更新的），所以原本是破解的，一更新就又变限制版了，需要重新改成上面的内容。
>
> 修改上面两个文件是用来把16个IP的家庭版转化成无限制的专业版的，windows平台破解方法同。

3. 进行定期自动更新脚本

   自己在安装目录下新建一个文件夹nessus-update，以后下载的插件包都放在这里，自动更新脚本也放这里。
   ![img](https://img2020.cnblogs.com/blog/2019081/202004/2019081-20200428160229779-594775637.png)

   设置crontab计划任务，每个月跑一次更新



## 五、卸载

1. mac系统卸载

```shell
# macOS系统卸载流程
系统设置 -> 选择nessus -> 解锁 -> 停止nessus -> 退出界面
sudo rm -rf /Library/Nessus
sudo rm -rf /Library/LaunchDaemons/com.tenablesecurity.nessusd.plist
sudo rm -rf /Library/PreferencePanes/Nessus Preferences.prefPane
sudo rm -rf /Applications/Nessus
sudo launchctl remove com.tenablesecurity.nessusd
```

2. linux卸载

   ```bash
   1. 停止进程
   # Red Hat, CentOS and Oracle Linux
   /sbin/service nessusd stop
   
   # Debian/Kali and Ubuntu
   /etc/init.d/nessusd stop
   
   2. 确定包名
   #Red Hat, CentOS, Oracle Linux, Fedora, SUSE, FreeBSD
   rpm -qa | grep Nessus
   
   #Debian/Kali and Ubuntu
   dpkg -l | grep Nessus
   
   3. 卸载
   #Red Hat, CentOS, Oracle Linux, Fedora, SUSE,
   rpm -e <Package Name>
   
   #Debian/Kali and Ubuntu
   dpkg -r <package name>
   
   4. 删除历史文件
   rm -rf /opt/nessus
   ```

   