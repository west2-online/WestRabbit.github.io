---
title: 福大助手iOS客户端更新日志&&随笔
date: 2016-02-10 22:35:31
author: 称一称
catalog: true
tags:
     - 福大助手
     - iOS
---

> 本文为福大助手更新日志记录及作者碎碎念系列，非正经博文，大家可直接略过。

### 3.7.3

- 新版课程详情存在错误的AutoLayout，暂时使用旧版详情界面替代。
- 修复考场时间缓存读取错误。
- 刷新成绩考场后立即保存数据。
- 修复iPad设备上启动画面的问题。

### 2/10 下发jspatch提示更新
```javascript
var data = require('NSUserDefaults').standardUserDefaults();
var alert_flag = data.objectForKey("show_alert_flag")
console.log(alert_flag)
if(alert_flag < 2){
    data.setObject_forKey(alert_flag+1,"show_alert_flag");
    data.synchronize();
    var alertView = require('UIAlertView').alloc().init();
    alertView.setTitle('升级提醒~');
    alertView.setMessage('福大助手升级啦，请尽快前往AppStore下载最新的3.7.2版本，我们尽可能修复了许多崩溃问题，有问题请尽快加群联系我们呦~ qq群234230317\n本消息将提示2次');
    alertView.addButtonWithTitle('知道啦~');
    alertView.show();
}

```

### 3.7.2

- 我们发现新版更新后崩溃率奇高无比。我们正在想方设法修复orz。
- 因为我们无法在自己的设备上重现出大家出现的错误，恳请出现崩溃的同学加群234230317反馈。。
- 我们将在开学前更新多个版本，希望能够稳定下来。
- 向Apple烧香中……感谢您的不离不弃。（请脑补程序员向服务器磕头的图）

这个版本理论上：

- 修复补考时间无法显示问题
- 修复学期菜单崩溃
- 修复考场为空时崩溃
- 重构考场、课表获取模块，支持显示备注
- 部分逻辑加固
- 修复成绩更新推送的问题（相信有些人在我们修复的那天收到了莫名的成绩更新推送orz，对不起！）
- 福大助手程序连续快速退出2次后会尝试清空助手数据以修复，麻烦按提示再重开一下啦谢谢~

有问题除了加群外……还可以发邮件到 yicheng.fzu@gmail.com ......
关于AppStore 的下载问题……这不是我们能控制的[笑哭]。对啦，如果您心情好，可以赐个好评吗?[卖萌]

快开学啦，祝……开学快乐？

### 3.7.1 && 3.7.0
- 尝试修复各种bug
- 重构了课表界面
- 更改绩点趋势图的样式
- 初步的研究生支持,目前暂时只支持课表功能
- 使用更安全的方式存储密码信息
- Wifi登录功能优化，主屏幕快捷菜单优化
- 只允许在ipad下使用横屏界面
- 支持在Spotlight快捷查看考试地点时间，只要输入考试名称或exam/考试 就好啦
- 再次尝试修复iOS8闪退
- 支持url-shame快速进入助手程序，可以使用 fzuhelper://goto/栏目名 来快速启动相应菜单
- 默认开启教务处通知推送，不需要的话可在侧滑菜单-设置-推送设置中关闭教务处通知。我们将使用推送功能给您继续带来实用的考场、成绩更新提醒~





