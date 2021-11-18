---
title: Debian系统安装Proxmox VE(PVE)虚拟化管理软件
date: 2021.11.17
tags: 
    - Proxmox VE
    - 19级
author: 3927
---



## 前言

由于公司最近服务器到期了，如果要直接续费的话，在没有优惠的情况下负担不起四千多一年的续费金额，恰逢腾讯云双十一做活动，同样配置服务器三年只需要七百多，故打算进行服务器迁移。

公司之前的服务是跑在虚拟机上的，对于每一个服务，开创一个虚拟化空间，在上面运行。虚拟空间的开创与配置等等是通过Proxmox VE实现的。

`Proxmox VE`（以下简称为PVE），是一个款开源虚拟化管理软件，和ESXI类似，简单的说就是用来开设和管理虚拟机的。

PVE的安装方式有2种，第一种是直接下载PVE的ISO直接安装，第二种是先装Debian再添加proxmox的安装源来安装，本教程将详细介绍第二种安装PVE的过程。

## 安装步骤
### 更换apt源
新建`/etc/apt/sources.list.d/pve-no-subscription.list`,以`Debian 10`系统为例，内容为：
```
deb https://mirrors.tuna.tsinghua.edu.cn/proxmox/debian buster pve-no-subscription
```
Debian系统其他版本的文件具体内容，请参考[该链接](https://mirrors.tuna.tsinghua.edu.cn/help/proxmox/)

### 更改hosts文件
对`/etc/hosts`文件进行如下修改，添加如下内容
```
<ip> <hostname>.proxmox.com <hostname>
```
其中`<ip>`为服务器公网ip或私网ip都可，届时需要通过该ip服务pve后台。`hostname`为任意名称即可。

这里以`ip`为**公网ip**，`hostname`为`pve`为例，需要在`/etc/hosts`文件中添加如下内容
```
135.138.161.12 pve.promxmox.com pve
```
当然，我上面写的肯定不是真的服务器公网ip。

修改完hosts文件后，设置服务器hostname为我们刚刚填加的hostname
```
hostnamectl set-hostname <hostname>
```

### 添加PVE安装源
依次执行下面的命令
```bash
wget http://download.proxmox.com/debian/proxmox-ve-release-6.x.gpg -O /etc/apt/trusted.gpg.d/proxmox-ve-release-6.x.gpg
chmod +r /etc/apt/trusted.gpg.d/proxmox-ve-release-6.x.gpg
apt update && apt full-upgrade
```

### 安装PVE
```
#安装
apt -y install proxmox-ve postfix open-iscsi
```
安装过程中根据提示进行设置即可，直接采用默认设置也问题不大。

### 收尾工作
1. 若不是采用双系统，可以删除`os-prober`软件包
```
apt remove os-prober
```

2. 使用`reboot`重启服务器，使系统加载PVE内核。

3. 重启成功后，访问`https://IP:8006`进入PVE后台管理页面，其账号密码即为服务器ssh登录的账号密码。注意采用`https`协议，否则进不去。

## 结语
到此为止PVE系统已经安装完成了，至于要怎么进行虚拟机的创建等等其他一系列操作，那就不是我的事了。

## 参考文章
[Debian 10安装Proxmox VE（PVE）虚拟化管理软件](https://cloud.tencent.com/developer/article/1834051)