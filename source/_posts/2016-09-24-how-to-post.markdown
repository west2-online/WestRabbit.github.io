---
layout:      post
title:      "How to post"
subtitle:   "如何将md传到仓库里"
date:       2016-09-24
author:     "XushengLee"
catalog:    true
tags:
     - Tutorial
---

> A post a day, keep bugs away.

# 环境

我是用的Mac，windows下路径的写法不同，请注意。我以本教程post为例。

# git clone

在terminal下进入自己喜欢的目录下，使用

	git clone https://github.com/WestRabbit/WestRabbit.github.io.git

那一长串是在下图拿到的
![img](git-clone.png)

# git add

当你在本地完成了编写之后。对每个你新建or修改过的文件or文件夹进行add，例如：
	
	git add ./source/_post/2014-09-24-how-to-post.markdown
	git add ./source/_post/2014-09-24-how-to-post

# git commit

在WestRabbit.github.io下

	git commit

# git push origin source

我们所用的分支叫做source。所以push到这里，同理在WestRabbit.github.io文件夹下

	git push origin source 

and job done !
