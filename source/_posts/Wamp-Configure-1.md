---
title: PHP集成环境WAMP简单配置指南
date: 2016-09-20 20:59:34
tags:
---
# Wamp配置篇（一）
为了将Wamp应用于实际站点搭设，需要对默认安装好的Wamp进行一些参数上的配置，以达到我们想要的效果

### 一、默认无密码状态
Wamp的MySQL默认是没有密码的，这对于实际应用而言有诸多的不便，因此，我们先更改个密码。
首先，打开MySQL的控制台，这时候会提示你：
```
Password:
```
默认是空的，所以回车即可，进入后，输入：
```
use mysql;
```
然后执行：
```
UPDATE user SET authentication_string=PASSWORD('密码') WHERE `user`='root';
```
这是对于MySQL5.7以上版本而言的，若为以下版本，则authentication_string用password替换，完成后，还应该执行以下语句让它生效：
```
flush privileges
```

----

### 二、忘记MySQL密码，怎么办？
编辑 **my.ini** 文件，打开后，找到
```
skip-grant-tables
```
去掉前面的“#“，然后这时候密码就为空了，再进入Mysql Console重复一的步骤即可

----

### 三、绑定域名
1、编辑 **/conf/httpd.conf** 文件，搜索：
```
Options +Indexs  +FollowSymLinks
```
将其改为：
```
Options -Indexs  +FollowSymLinks
```
这一步是为了禁止列出目录，避免安全问题
2、搜索：
```
AllowOverride all
```
在其下面加入：
```
Require all granted
```
3、在Apache的conf/extra/httpd-vhost.conf文件里，加入：
```
<VirtualHost *:80>
    DocumentRoot "文件路径"
    ServerName 域名
</VirtualHost>
```