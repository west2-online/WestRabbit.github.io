---
layout: post
title:  "iOS-初解SceneDelegate"
date: 2020-11-28
author: "Capa"
catalog: true
tags:
     - iOS
     - 19级
---



### 前言

iOS13之后，AppDelegate的职能被拆解为：

- AppDelegate

- SceneDegate

  

引入场景（Scene）概念，应用程序现在能够拥有不止一个场景。

如果开发者不开发多场景的iPadOS/MacOS程序，仅使用AppDelegate也可以满足需求，但个人认为这是有意思的概念，所以在此介绍。

 在本文中，为便于（自己）理解，笔者将采取口语化表述，如果有误欢迎指正。

### 相关名词解释

- UIScene(场景)

  用户应用程序界面的对象实例

- UISceneSession（场景会话）

  包含一个场景配置信息的对象

- UIWindowScene

  特定类型的场景，管理一个或多个窗口(Window)

- UIWindow（窗口）

  应用程序的界面容器，并将事件调度给视图（View）

### AppDelegate相关方法

负责应用整体生命周期和设置，处理场景会话

#### App生命周期

`application(_:didFinishLaunchingWithOptions:) -> Bool`

#### 处理场景会话

`application(_:configurationForConnecting:options:) -> UISceneConfiguration`

当应用需要新场景或窗口进行显示，调用此方法

返回场景的配置数据，包括场景名称和它的委托类（SceneDelegate）名称

（提供的所有场景需要事先在Info.plist里声明）

`application(_:didDiscardSceneSessions:)`

当应用丢弃一个或多个旧场景时，调用此方法

### SceneDelegate的各种方法

控制和管理应用界面和内容（用户看到的内容），并管理应用的显示方式

#### 连接场景

`scene(_:willConnectTo:options:)`

#### 调用场景

`sceneWillEnterForeground(_:)`

场景即将被唤起，调用该方法

`sceneDidBecomeActive(_:)`

场景已经进入活跃状态并可见，开始响应用户事件，调用该方法

#### 挂起场景

`sceneWillResignActive(_:)`

场景即将退出活跃状态，停止响应用户事件，调用该事件

`sceneDidEnterBackground(_:)`

场景已经在后台运行且不可见，调用该方法

#### 断开场景

`sceneDidDisconnect(_:)`

应用程序当前不再调用这个场景，切断和它的联系，等再次使用时重新连接

### 方法运用实例

* 在`scene(_:willConnectTo:options:)`设置应用程序窗口，分配根视图控制器(iOS13以前该操作在`application(**_** application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: **Any**]?) -> Bool`中进行)
* 在`sceneWillEnterForeground(_:)`从硬盘或网络获取数据资源
* 在`sceneDidBecomeActive(_:)`更新数据模型并更新UI，处理用户事件，如：切换视图控制器，启动计时器等等
* 在`sceneWillResignActive(_:)`保存数据，挂起操作队列，销毁计时器
* 在`sceneDidEnterBackground(_:)`隐藏私密信息，释放资源，如：图像，视频，相机权限等
* 在`sceneDidDisconnect(_:)`执行清理操作，释放对文件或共享资源的引用并保存数据







