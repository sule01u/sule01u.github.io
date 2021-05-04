---
layout:     post                    
title:      安全建设之WAF(Modsecurity)防御        
date:       2020-04-17           
author:     suleo       
header-img: img/IMG_7745.JPG   
catalog: true                     
tags:                       
    - 渗透测试
---
# Modsecurity介绍

> WAF (Web Application Firewall)，Web 应用防火墙，通过解析 HTTP/HTTPS 请求内容，并执行一系列的安全检测策略，对目标 Web 应用提供安全防护，同时记录相关安全防御日志。文章将介绍 ModSecurity 相关部署配置概念、防御规则及简单地定制化实践。

## WAF 概念简述

WAF 根据产品形态不同，通常可以分为硬件设备类、软件产品类及云 WAF 类别，相关产品代表如下

- 硬件设备类，如绿盟、安恒、启明星辰等厂商生产的 WAF；
- 软件产品类，如 `ModSecurity` 、网站安全狗等；
- 云 WAF，如阿里云盾、知道创宇加速乐等；

根据工作原理不同可分为反向代理模式、云 AGENT 模式及容器安全模块形式

| 工作模式      | 工作原理                                                     | 优劣分析                                                     |
| :------------ | :----------------------------------------------------------- | :----------------------------------------------------------- |
| 反向代理模式  | 修改DNS，让域名解析到反向代理服务器。所有流量经过反向代理进行检测，检测无问题之后再转发给后端的Web服务器 | 集中式流量出入口，可针对大数据分析，动态添加一层，增加网络开销，站点后端 Web 较多的情况，转发规则比较复杂，流量都被捕捉，涉及敏感数据需要保护 |
| 云 AGENT 模式 | 所有请求通过 Web 服务器模块转发到云端进行安全检测，NGINX 根据云端检测结果进行转发，若检测为攻击则执行配置的动作 | 安全规则统一管理，规则更新只需要更新后端的决策系统，不涉及 Web 服务器端规则更新，根据业务配置规则，需要较高成本，Web 服务器变更时不便于更新 |
| 容器安全模块  | 所有请求流量均先经过检测引擎的检测，若未发现攻击，则进行正常业务响应，否则按照配置的动作进行响应 | 网络结构简单，仅部署容器安全模块，但维护困难，大规模服务器集群，任何更新涉及多台服务器，需部署操作，成本高，无集中化数据中心，安全事件汇总不方便 |

## ModSecurity 部署

> `Nginx` 加载 `ModSecurity` 模块安装有两种方式：一种是编译为 `Nginx` 静态模块，另一种是通过 `ModSecurity-Nginx Connector` 加载动态模块，下面将实践 `ModSecurity-Nginx Connector` 动态模块的安装和配置

安装编译 `libmodsecurity` 依赖库

```
yum install -y pt-utils autoconf automake build-essential git libcurl4-openssl-dev libgeoip-dev liblmdb-dev libpcre++-dev libtool libxml2-dev libyajl-dev pkgconf wget zlib1g-dev doxygen libxml2-devel pcre-devel
```

### 编译 libmodsecurity 模块

下载 `libmodsecurity` 源码并编译模块

```
git clone --depth 1 -b v3/master --single-branch https://github.com/SpiderLabs/ModSecurity
cd ModSecurity
git submodule init
git submodule update
./build.sh
./configure
make
make install
```

> 若编译安装过程出现如下错误提示，可直接忽略
>
> ```
> fatal: No names found, cannot describe anything.
> ```

### 编译 nginx connector 模块

下载 `nginx connector` 模块，并编译成动态模块

```
git clone --depth 1 https://github.com/SpiderLabs/ModSecurity-nginx.git
```

> 目标机器已安装 `Nginx` ，需确认 `Nginx` 版本，并下载指定版本的 `Nginx` 源码进行编译，否则可下载任意版本 `Nginx` 源码进行编译

下载 `Nginx` 源码进行编译，编译过程加入 `ModSecurity` 依赖库模块

```
wget http://nginx.org/download/nginx-1.14.1.tar.gz
tar zxvf nginx-1.14.1.tar.gz
cd nginx-1.14.1
./configure --add-dynamic-module=../ModSecurity-nginx
make modules
```

> 网上部分文档编译过程 `configure` 添加编译参数 `--with-compat` 启用模块兼容性，导致编译后的 `so` 模块无法使用(错误提示如下)，去除该选项即可
>
> ```
> nginx: [emerg] module "/usr/local/nginx/conf/modules/ngx_http_modsecurity_module.so" is not binary compatible in /usr/local/nginx/conf/nginx.conf:11
> ```

将编译成功的 ModSecurity 模块复制到 Nginx 安装目录中，在 Nginx 配置文件中添加如下配置，加载 ModSecurity 模块

```
load_module modules/ngx_http_modsecurity_module.so;
```

### 配置 ModSecurity 规则

下载 `ModSecurity` 配置规则

```
mkdir modseucirty
wget -P xxx/modsecurity/ https://raw.githubusercontent.com/SpiderLabs/ModSecurity/v3/master/modsecurity.conf-recommended
mv xxx/modsecurity/modsecurity.conf-recommended xxx/modsecurity/modsecurity.conf
```

> 在对 `ModSecurity` 进行配置时，需要将原 `ModSecurity` 文件夹中的 `unicode.mapping` 复制到与上述 `modsecurity.conf` 文件相同的文件夹中，否则启动 `Nginx` 过程会提示文件缺失，如下所示
>
> ```
> nginx: [emerg] "modsecurity_rules_file" directive Rules error. File: /usr/local/nginx/conf/modsec/modsecurity.conf. Line: 236. Column: 17. Failed to locate the unicode map file from: unicode.mapping Looking at: 'unicode.mapping', 'unicode.mapping', '/usr/local/nginx/conf/modsec/unicode.mapping', '/usr/local/nginx/conf/modsec/unicode.mapping'.  in /usr/local/nginx/conf/nginx.conf:41
> ```

修改 `ModSecurity` 配置文件 `modsecurity.conf` 中检测模式 `SecRuleEngine DetectionOnly` 为 `SecRuleEngine On`，如下图所示

```
# -- Rule engine initialization ----------------------------------------------

# Enable ModSecurity, attaching it to every transaction. Use detection
# only to start with, because that minimises the chances of post-installation
# disruption.
#
#SecRuleEngine DetectionOnly
SecRuleEngine On
```

在 `Nginx` `Server` 配置中开启 `ModSecurity` 配置，如下所示

```
nginx.conf 配置文件

server {
    listen       80;
    server_name  localhost;

    modsecurity on;
    modsecurity_rules_file /usr/local/nginx/conf/modsec/main.conf;

    location / {
        root   html;
        index  index.html index.htm;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   html;
    }
}

----------------------------------------------------------------------------------------------

main.conf 配置文件

Include "/usr/local/nginx/conf/modsec/modsecurity.conf"
# 添加测试规则，当请求中包含参数 testparam 同时其值为 test 时，拦截请求并返回 403
SecRule ARGS:testparam "@contains test" "id:1234,deny,status:403"
```

### ModSecurity 规则测试

`ModSecurity` 规则测试如下图所示，当携带参数 `?testparam=test` 发起请求时，请求被拒绝

![img](https://l0gs.xf0rk.space/2018/12/21/step-into-modsecurity/modsecurity_testparam_demo.png)

## ModSecurity 配置简介

### ModSecurity 日志配置

`ModSecurity` 应用中的日志包括 `Debug`（调试日志）、`Audit`（审计日志）两大类，调试日志记录 `ModSecurity` 规则匹配、检测过程，并依据调试日志相关参数的不同，其所记录的日志字段均不同，调试日志配置如下所示

| 调试日志配置     | 参数说明         |
| :--------------- | :--------------- |
| SecDebugLog      | 调试日志记录路径 |
| SecDebugLogLevel | 调试日志记录等级 |

调试日志记录等级分为 7 个等级，如下所示

- 0 不输出日志
- 1 输出错误日志，例如致命的处理错误、阻塞的会话
- 2 记录警告日志，例如非阻塞的规则匹配
- 3 输出通知日志，例如非致命的处理错误
- 4 信息
- 5 详情
- 9 输出所有调试日志

审计日志，可用于输出所有 HTTP 会话日志，其日志数据如下所示

| 日志属性                  | 参数说明                                                     |
| :------------------------ | :----------------------------------------------------------- |
| SecAuditEngine            | 控制审计日志输出，其可选值为 On、Off、RelevantOnly           |
| SecAuditLogRelevantStatus | 待审计的 HTTP 状态码，例如 ^(?:5\|4(?!04)) 用于过滤 5xx 或 4xx，排除 404 |
| SecAuditLogParts          | ABCDEFGHIJKZ 所需记录的 HTTP 请求字段                        |
| SecAuditLogType           | 审计日志类型，可选值为 Serial、Concurrent                    |
| SecAuditLog               | 记录审计日志路径                                             |
| SecAuditLogStorageDir     | Concurrent 审计日志时所输出的目录                            |

`SecAuditLogParts` 取值为 `ABCDEFGHIJKZ` 说明如下所示

| 日志属性 | 参数说明                       |
| :------: | :----------------------------- |
|    A     | 审计日志头                     |
|    B     | 请求头                         |
|    C     | 请求 Body                      |
|    D     | 保留                           |
|    E     | 响应 Body                      |
|    F     | 响应头                         |
|    G     | 保留                           |
|    H     | 审计标记，包含额外审计会话数据 |
|    I     | 保留                           |
|    J     | 保留                           |
|    K     | 所匹配的规则集合               |
|    Z     | 终结标记                       |

如下为一段审计日志示例

```
---Xd59KAAO---A--
[26/Nov/2018:09:46:59 +0800] 154319681942.596956 10.211.55.2 58760 10.211.55.2 80
---Xd59KAAO---B--
GET /?testparam=test HTTP/1.1
Host: 10.211.55.14
Connection: keep-alive
Cache-Control: max-age=0
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_1) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/70.0.3538.102 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,en-US;q=0.8,en;q=0.7
If-None-Match: "5bf974af-264"
If-Modified-Since: Sat, 24 Nov 2018 15:56:31 GMT

---Xd59KAAO---C--

---Xd59KAAO---D--

---Xd59KAAO---E--
<html>\x0d\x0a<head><title>403 Forbidden</title></head>\x0d\x0a<body bgcolor="white">\x0d\x0a<center><h1>403 Forbidden</h1></center>\x0d\x0a<hr><center>nginx/1.14.1</center>\x0d\x0a</body>\x0d\x0a</html>\x0d\x0a<!-- a padding to disable MSIE and Chrome friendly error page -->\x0d\x0a<!-- a padding to disable MSIE and Chrome friendly error page -->\x0d\x0a<!-- a padding to disable MSIE and Chrome friendly error page -->\x0d\x0a<!-- a padding to disable MSIE and Chrome friendly error page -->\x0d\x0a<!-- a padding to disable MSIE and Chrome friendly error page -->\x0d\x0a<!-- a padding to disable MSIE and Chrome friendly error page -->\x0d\x0a

---Xd59KAAO---F--
HTTP/1.1 403
Server: nginx/1.14.1
Date: Mon, 26 Nov 2018 01:46:59 GMT
Content-Length: 571
Content-Type: text/html
Connection: keep-alive

---Xd59KAAO---H--

---Xd59KAAO---I--

---Xd59KAAO---J--

---Xd59KAAO---Z--
```

> 在测试 ModSecurity 功能时，想通过 WAF 输出的审计日志来学习和测试相应防护规则，结果发现将 SecRuleEngine 设置为 DetectionOnly 后，ModSecurity 并没有在审计日志中输出匹配过程和规则；正确的做法应该是通过记录调试日志，来跟踪规则匹配、执行。

## ModSecurity CRS 规则

ModSecurity [CRS](https://github.com/SpiderLabs/owasp-modsecurity-crs) 是由 OWASP 社区所维护的攻击检测规则，可在 ModSecurity 中直接使用，其检测规则覆盖 OWASP Top 10 中常见漏洞和一些框架层面的漏洞。

### CRS 部署使用

下载 CRS 规则到本地，可通过如下命令进行操作

```
git clone git@github.com:SpiderLabs/owasp-modsecurity-crs.git
```

在 CRS 的 `crs-set.conf.example` 中说明 CRS 的配置需要在 WebServer 中包含如下三种配置文件

```
# The order of file inclusion in your webserver configuration should always be:
# 1. modsecurity.conf
# 2. crs-setup.conf (this file)
# 3. rules/*.conf (the CRS rule files)
#
# Please refer to the INSTALL file for detailed installation instructions.
```

将下载目录下的 `rules` 移动到 `/usr/local/nginx/conf/modsec` 目录下，通知将 `crs-set.conf.example` 重命名为 `crs-set.conf` 并移动到 `/usr/local/nginx/conf/modsec`，部署玩 CRS 后的目录如下所示

```
[root@Centos7 modsec]# pwd
/usr/local/nginx/conf/modsec
[root@Centos7 modsec]# ls
crs-setup.conf  main.conf  modsecurity.conf  rules  unicode.mapping
[root@Centos7 modsec]#
```

在上述 `/usr/local/nginx/conf/modsec/main.conf` 配置文件中添加 CRS 配置，添加后的文件内容如下所示

```
[root@Centos7 modsec]# cat main.conf
Include "/usr/local/nginx/conf/modsec/modsecurity.conf"
Include "/usr/local/nginx/conf/modsec/crs-setup.conf"
Include "/usr/local/nginx/conf/modsec/rules/*.conf"
```

测试 CRS 规则，访问连接 `http://10.211.55.14/?id=1'or'1=1`，如下所示，触发了 CRS 中的 SQL 注入防御规则

> 需注意 ModSecurity CRS 规则默认将本机加入白名单，所以直接通过 `curl http://localhost?id=1'or'1=1` 测试是无法触发 CRS 防御规则。

![img](https://l0gs.xf0rk.space/2018/12/21/step-into-modsecurity/modsecurity_crs.png)

触发 `REQUEST-920-PROTOCOL-ENFORCEMENT`、`REQUEST-942-APPLICATION-ATTACK-SQLI.conf` 等规则检测，并且被默认拦截，其匹配记录如下所示

```
ModSecurity: Warning. Matched "Operator `Rx' with parameter `^[\d.:]+$' against variable `REQUEST_HEADERS:Host' (Value: `10.211.55.14' ) [file "/usr/local/nginx/conf/modsec/rules/REQUEST-920-PROTOCOL-ENFORCEMENT.conf"] [line "777"] [id "920350"] [rev "2"] [msg "Host header is a numeric IP address"] [data "10.211.55.14"] [severity "4"] [ver "OWASP_CRS/3.0.0"] [maturity "9"] [accuracy "9"] [tag "application-multi"] [tag "language-multi"] [tag "platform-multi"] [tag "attack-protocol"] [tag "OWASP_CRS/PROTOCOL_VIOLATION/IP_HOST"] [tag "WASCTC/WASC-21"] [tag "OWASP_TOP_10/A7"] [tag "PCI/6.5.10"] [hostname "10.211.55.2"] [uri "/"] [unique_id "154530934235.756485"] [ref "o0,12v37,12"]
ModSecurity: Warning. detected SQLi using libinjection. [file "/usr/local/nginx/conf/modsec/rules/REQUEST-942-APPLICATION-ATTACK-SQLI.conf"] [line "43"] [id "942100"] [rev "1"] [msg "SQL Injection Attack Detected via libinjection"] [data "Matched Data: s&s found within ARGS:id: 1'or'1=1"] [severity "2"] [ver "OWASP_CRS/3.0.0"] [maturity "1"] [accuracy "8"] [hostname "10.211.55.2"] [uri "/"] [unique_id "154530934235.756485"] [ref "v9,8"]
ModSecurity: Access denied with code 403 (phase 2). Matched "Operator `Ge' with parameter `5' against variable `TX:ANOMALY_SCORE' (Value: `8' ) [file "/usr/local/nginx/conf/modsec/rules/REQUEST-949-BLOCKING-EVALUATION.conf"] [line "44"] [id "949110"] [rev ""] [msg "Inbound Anomaly Score Exceeded (Total Score: 8)"] [data ""] [severity "2"] [ver ""] [maturity "0"] [accuracy "0"] [tag "application-multi"] [tag "language-multi"] [tag "platform-multi"] [tag "attack-generic"] [hostname "10.211.55.2"] [uri "/"] [unique_id "154530934235.756485"] [ref ""]
ModSecurity: Warning. Matched "Operator `Ge' with parameter `5' against variable `TX:INBOUND_ANOMALY_SCORE' (Value: `8' ) [file "/usr/local/nginx/conf/modsec/rules/RESPONSE-980-CORRELATION.conf"] [line "65"] [id "980130"] [rev ""] [msg "Inbound Anomaly Score Exceeded (Total Inbound Score: 8 - SQLI=5,XSS=0,RFI=0,LFI=0,RCE=0,PHPI=0,HTTP=0,SESS=0): SQL Injection Attack Detected via libinjection"] [data ""] [severity "0"] [ver ""] [maturity "0"] [accuracy "0"] [tag "event-correlation"] [hostname "10.211.55.2"] [uri "/"] [unique_id "154530934235.756485"] [ref ""]
```

### CRS 规则说明

`/usr/local/nginx/conf/modsec/rules/*.conf` 目录下的 CRS 防御规则文件及相关说明如下所示

| 规则文件                                        | 规则说明                                     |
| :---------------------------------------------- | :------------------------------------------- |
| REQUEST-900-EXCLUSION-RULES-BEFORE-CRS          | 定制化个人防护规则，调整规则防护场景         |
| REQUEST-901-INITIALIZATION                      | CRS 配置初始化                               |
| REQUEST-910-IP-REPUTATION                       | IP 信誉库                                    |
| REQUEST-911-METHOD-ENFORCEMENT                  | HTTP 请求方法检测                            |
| REQUEST-912-DOS-PROTECTION                      | 拒绝服务规则                                 |
| REQUEST-913-SCANNER-DETECTION                   | 扫描器检测                                   |
| REQUEST-920-PROTOCOL-ENFORCEMENT                | URL Scheme 协议检测                          |
| REQUEST-921-PROTOCOL-ATTACK                     | HTTP 协议攻击检测                            |
| REQUEST-930-APPLICATION-ATTACK-LFI              | LFI 本地文件包含漏洞检测                     |
| REQUEST-931-APPLICATION-ATTACK-RFI              | RFI 远程文件包含检测                         |
| REQUEST-932-APPLICATION-ATTACK-RCE              | RCE 远程命令执行检测                         |
| REQUEST-933-APPLICATION-ATTACK-PHP              | PHP 攻击检测                                 |
| REQUEST-941-APPLICATION-ATTACK-XSS              | XSS 攻击检测                                 |
| REQUEST-942-APPLICATION-ATTACK-SQLI             | SQL 注入攻击检测                             |
| REQUEST-943-APPLICATION-ATTACK-SESSION-FIXATION | 会话固定攻击检测                             |
| REQUEST-949-BLOCKING-EVALUATION                 | 基于风险值检测入站请求                       |
| RESPONSE-950-DATA-LEAKAGES                      | 数据泄漏检测                                 |
| RESPONSE-951-DATA-LEAKAGES-SQL                  | SQL 数据泄漏检测                             |
| RESPONSE-952-DATA-LEAKAGES-JAVA                 | JAVA 数据泄漏                                |
| RESPONSE-953-DATA-LEAKAGES-PHP                  | PHP 数据泄漏                                 |
| RESPONSE-954-DATA-LEAKAGES-IIS                  | IIS 数据泄漏                                 |
| RESPONSE-959-BLOCKING-EVALUATION                | 基于风险值检测出站请求                       |
| RESPONSE-980-CORRELATION                        | 入站、出站风险值关联                         |
| RESPONSE-999-EXCLUSION-RULES-AFTER-CRS          | 定制化个人防护规则，重载、更新、移除检测规则 |

如下所示，CRS 支持两种运行模式，CRS 默认采用的为异常评分模式

- 异常评分模式，每条防御规则都有相应的风险评分，若匹配成功，则直接累加对应的风险值，当所有规则都匹配完成后比较入站、出站的风险值，若比设定的风险阈值还高，则默认返回 403；
- 检测拦截模式，若请求匹配到了防御规则，则依据所配置的行为进行响应，同时将忽略其后所有的防御规则，该模式节约资源、性能；

若 CRS 要切换成检测拦截模式，需将 `crs-setup.conf` 中异常评分模式配置注释，同时开启检测拦截模式配置，如下所示

```
#SecDefaultAction "phase:1,log,auditlog,pass"
#SecDefaultAction "phase:2,log,auditlog,pass"

SecDefaultAction "phase:1,log,auditlog,deny,status:403"
SecDefaultAction "phase:2,log,auditlog,deny,status:403"
```

在异常评分模式中，CRS 包含 规则等级（Paranoia）、异常阈值（Anomaly） 两个量化值，随着规则等级越来越高，其所启用的安全防御规则就越来越多，同时误报也会越来越多，规则等级分为以下几类，规则等级高于 PL2 在审计日志中会输出规则等级标签

- 1（PL1），默认风险等级，启用了大部分防御规则，误报较少
- 2（PL2），比 PL1 启用更多防御规则，例如基于正则的 SQL 注入和 XSS，比 PL1 误报多
- 3（PL3），比 PL2 启用更多防御规则，面向经验丰富用户，满足较高安全性场景
- 4（PL4），最严格的风险等级，会产生一定数量的误报

基于异常告警模式，每条检测规则都包含一定的风险值，不同危害风险值不同，如下所示，可根据业务场景自主调整

- CRITICAL，致命，风险值为 5
- ERROR，错误，风险值为 4
- WARNING，警告，风险值为 3
- NOTICE，通知，风险值为 2

根据对 CRS 防御规则的熟知程度慢慢深入，相关异常阈值和规则等级设置可参考下表所示，即异常告警阈值慢慢降低，规则等级慢慢严格

| 阶段编号 | 阶段名称     | 异常阈值 | 规则等级 |
| :------: | :----------- | :------: | :------: |
|    1     | 初始阶段     |    高    |    低    |
|    2     | 实验阶段     |    高    |    高    |
|    3     | 标准阶段     |    低    |    低    |
|    4     | 高安全性阶段 |    低    |    高    |

## 参考链接

1. 如何搭建一个靠谱的WAF [查看原文](http://www.91ri.org/12766.html)
2. 【门神】WAF应用层实现的架构漫谈 [查看原文](https://security.tencent.com/index.php/blog/msg/63)
3. 主流WAF架构分析与探索 [查看原文](https://security.tencent.com/index.php/blog/msg/56)
