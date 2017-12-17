---
title:      "Git WebHook简易自动部署笔记 "
subtitle:   "一次git+webhook自动部署的尝试？"
date:       2017-12-17 18:30:00
author:     "yicheng"
catalog:     true
tags:
     - 14级
     - Linux
---

> CYC有三台服务器分别位于腾讯云、阿里云、美国，最近在研究自动化部署。主要使用git+webhook方案。实现git push后服务器自动拉取并运行部署脚本。
> 
> CYC在写iOS的过程中，经常划水写一写Python的Flask Web项目。感觉本地修改调试然后部署到服务器有时有点麻烦=、= 趁前阵子工作不饱和的时候瞎折腾了一番。

## 部署git服务器
为了搭建私有git 原先是简单使用 git init --bare 的方式建立仓库然后编辑hook/post-recieve 添加脚本，造成扩展复杂并且脚本执行的权限便是ssh推送用户的权限，略有不便，同时在两台服务器+多个项目的情况下，部署复杂。详见 https://zhuanlan.zhihu.com/airbnb/19757507

于是这次搭建了一个gogs 一个由go语言编写的git服务器（后来发现有个衍生版本gotea，不过因为已经弄好了就没再折腾了）。
![](https://ws1.sinaimg.cn/large/006tNc79gy1fmk0al5citj31kw0ldadh.jpg)

gogs中可以很方便的添加webhook记下密钥文本，这是webhook服务端用来验证推送者合法性的密钥

对了，截止现在最新版本的左侧选项中的git钩子是无效的在官方issue中确认了这个问题，坑。
![](https://ws3.sinaimg.cn/large/006tNc79gy1fmk0d0tit3j31kw0lk7ao.jpg)

添加完后就长这样

![](https://ws3.sinaimg.cn/large/006tNc79gy1fmk0c4rv7mj31kw0kx42d.jpg)


## 准备部署脚本
诶诶诶，webhook怎么生效的呢？ 便是仓库有动作后遍像指定的url POST一个提交的相关信息。服务器收到这条POST校验合法后执行部署脚本进行上线操作。下面是我一个python-web服务器的部署脚本，其实就是很简单的git pull然后reload服务器。

``` shell
#!/usr/bin/env bash
set -xe

echo "Running Post Receive Hook"

export GIT_WORK_TREE=/home/pyweb/src

cd ${GIT_WORK_TREE}
git fetch origin
git reset --hard origin/master
# reload my apps
echo "reload"
supervisorctl restart pyweb
echo "finish!"

```

## 运行webhook服务器
webhook服务器有很多开源的项目，自己写一个也非常的简单。因为我的需求也非常简单，只需要收到指定项目post url后运行特定项目的部署脚本即可。于是用flask简单实现了一个。代码如下

![](https://ws1.sinaimg.cn/large/006tNc79gy1fmk0oto249j313217in4o.jpg)

同时支持通过配置文件来进行不同项目的配置

```json
{
  "progect1": {
    "key": "xxxxxxx",
    "shell_path": "progect1.sh",
    "branch": "master"
  },
  "progect2":{
      .......
  }
}
```

然后就可以设置webhook url到  https://xxxxx/webhook/update/project1 进行更新操作了。


## 让supervisorctl 能在非root权限下运行
刚刚有人肯定注意到我在脚本中执行了 supervisorctl 并没有添加sudo，而我们的webhook服务器也肯定是已非root的低权限用户运行的，那要怎么做到呢？

编辑 /etc/supervisor/supervisord.conf

```
修改此字段
[unix_http_server]
file=/var/run/supervisor.sock   ; (the path to the socket file)
chmod=0770                       ; sockef file mode (default 0700)
chown=nobody:git ;这里我允许git用户组进行操作
```

然后重启supervisor即可。
 
## 用git管理服务器公钥、https证书？
由于CYC的免费https wildcard将于2018年过期，同时听说let's encrypt 也即将允许签署wildcard。打算将多个服务器的https证书同一部署webhook管理。部署脚本做 git pull; nginx reload。想想应该很靠谱（大flag（（误

想想ssh公钥也可以这样管理吧？git pull && 替换 ~/.ssh/authorized_keys && chmod 600 && service ssh restart

嗯 多人服务器管理中 要添加密钥还可以又用户发起pull request，admin同意merge后即可自动授权（对了 gogs也支持网页上直接编辑文件不需要clone，perfect！） 

但是 这会遇到一个权限问题：
 
## 指定命令可以在特定用户下sudo免密码运行
执行 sudo visudo 编辑sudoer文件

添加

```
www-data ALL=(ALL:ALL) NOPASSWD:/usr/sbin/service nginx reload 
// 这里也可以替换成一个指定的shell脚本路径
```

运行后如图

![](https://ws1.sinaimg.cn/large/006tNc79gy1fmjzwsoczzj30lq06kwh7.jpg)

可见只有这条命令能够sudo提权，比起添加整个用户到sudo group安全的多啦~

有时在这里sudo会报错“unable to resolve host xxxx（hostname）” 这时我们只要编辑hosts添加hostname到127.0.0.1的映射即可。

阿里云给的hostname是一堆接近随机数字的编码，看着很不爽 好在ubuntu下可以很方便的永久修改hostname。只需

```
sudo vi /etc/hostname
```

编辑完文件保存 重启即可。

# Emmmmmmm
感觉好像差不多了诶。。。不过感觉很粗暴emmmmm（瑟瑟发抖，溜了溜了），还是怎么简单怎么来吧。接下去有空去了解了解docker。
