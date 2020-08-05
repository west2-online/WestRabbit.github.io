---
title:      "初探iOS Network Extension(新手向) "
subtitle:   "Network Extension听着很高大上。然而用上NEKit,我们可以分分钟钟写出个代理App"
date:       2016-11-21 18:30:00
author:     "yicheng"
catalog:     true
tags:
     - 14级
     - iOS
     - 教程
---


> 经过一番申请，我们西二在线iOS团队成功申请得到了Network Extension的Entitlement。于是，我们开始研究ne的开发。本文使用Swift语言及NEKit开源库制作一个简单的代理软件。



## 一、 安装NEProviderTargetTemplates.pkg

由于未知原因苹果在mac OS 10.12中删除了这个文件，因此我们需要从10.11系统中提取或[下载](https://www.dropbox.com/s/6f1rxfjkk0ha89n/NEProviderTargetTemplates.pkg?dl=0)。

安装完毕后，在新增build target中我们就可以看到多了AppProxy和 Package Tunnrl Provider。我们选择Package Tunnrl Provider，在下一步中照例输入包名作者名等等，完成添加。

![添加Packet Tunnel Provider](https://ww4.sinaimg.cn/large/65e4f1e6gw1f9ztxlxo0qj20ka0eedhh.jpg)



## 二、申请Profile

Network Extension不同于其他权限，无法在Xcode的compalities中一键启用，需要在Apple Developer中心手动申请profile。申请过程中会让选择additional entitlement，钩上我们的ne证书,生成并下载倒入Xcode。

同时编辑entitlememts文件，加入下列内容。

```plist
	<key>com.apple.developer.networking.networkextension</key>
	<array>
		<string>packet-tunnel-provider</string>
		<string>app-proxy-provider</string>
		<string>content-filter-provider</string>
	</array>
```

效果如图

![](https://ww2.sinaimg.cn/large/65e4f1e6gw1f9zu1xof5nj20gd054dg8.jpg)

*PS.你需要同时为主程序和Extension创建profile。*



## 三、第一次建立VPN链接

### 创建VPN profile

1. 首先，我们需要在主程序中像系统生名一个ProviderManager，即设置VPN中的栏目。

   ```swift
   let manager = NETunnelProviderManager()
   let conf = NETunnelProviderProtocol()
   conf.serverAddress = "Rabbit" //任意值,显示在设置-VPN-Detial中
   manager.protocolConfiguration = conf
   manager.localizedDescription = "Rabbit VPN"
   manager.enable = True //使VPN在系统中变为选中的状态
   ```

   ​


2. 将manager保存至系统中。

   ```swift
   manager.saveToPreferencesWithCompletionHandler{
   	error in
       if error != nil{print(error);return;}
       //Todo
   }
   ```

   此时，打开系统-Vpn菜单，即可看见我们新建的Vpn条目

   ![](https://ww1.sinaimg.cn/large/65e4f1e6gw1f9zucaehpfj20a40hyaah.jpg)

   ​

   3. 此时如果save方法调用多次，会出现VPN 1 VPN 2等多个描述文件 ，因此，苹果也要求，在创建前应读取当前的managers。

      ```swift
      NETunnelProviderManager.loadAllFromPreferencesWithCompletionHandler{ 
          (managers, error) in
          guard let managers = managers else{return}
          let manager: NETunnelProviderManager
          if managers.count > 0 {
              manager = managers[0]
          }else{
              manager = self.createProviderManager()
          }
          // Todo
          // manager.saveToPreferences.......
      }
      ```

      ​

### 简单配置extension

1. 打开extension中的模板文件，对应Swift版本与父类修改语法。主要需要以下两个函数控制VPN状态

   ```swift
   func startTunnelWithOptions(options: [String : NSObject]?, completionHandler: (NSError?) -> Void) 
   //启动VPN时调用

   func stopTunnelWithReason(reason: NEProviderStopReason, completionHandler: () -> Void)
   //停止VPN时调用
   ```

2. 此时我们仅仅需要测试VPN连接是否能正常建立。因此我们只需要在startTunnelWithOptions中调用setTunnelNetworkSettings方法即可。当completionHandler()执行后VPN连接就会显示在手机上了。

   ```swift
   override func startTunnelWithOptions(options: [String : NSObject]?, completionHandler: (NSError?) -> Void) {
   	let ipv4Settings = NEIPv4Settings(addresses: ["10.0.0.1"], subnetMasks: ["255.255.255.0"])
   	// 这里RemoteAddress可任意填写。
   	let networkSettings = NEPacketTunnelNetworkSettings(tunnelRemoteAddress: "8.8.8.8")
   	networkSettings.MTU = 1500
   	networkSettings.IPv4Settings = ipv4Settings
   	setTunnelNetworkSettings(networkSettings) {
   	    error in
   	    guard error == nil else {
   	    	nslog(error.debugDescription)
   	        completionHandler(error)
   	        return
   	    }
   	    completionHandler(nil)
   	}
   }
   ```

   ​

### 启动VPN

1. 启动VPN很简单，只需对ProviderManager执行startVPNTunnelWithOptions()方法即可

   ```swift
    try! manager.connection.startVPNTunnelWithOptions([:])
   ```

   - 注意，如果在创建VPN Save后立刻执行startVPN，会遇到报错说Domain=“null”。此时说明系统还未准备好。所以应将启动代码放到 loadFromPreferences 的Block中。

   ```swift
   saveToPreferences{
      error in
      .....
      manager.loadFromPreferencesWithCompletionHandler{
   	    if $0 != nil{print($0)}
   	    manager.startVPN.........
   	}
   }
   ```

2. 此时，VPN应该能成功启动。

   ![](https://ww1.sinaimg.cn/large/65e4f1e6gw1f9zvofsip3j20b1064wep.jpg)

3. 连接成功会manager.connection.status会发生相应改变，因此我们需要在按下连接按钮后监听status，从而知道目前Vpn的连接状态。

   ```swift
   loadProviderManager { [unowned self] (manager) -> Void in
       if let manager = manager {
           NSNotificationCenter.defaultCenter().addObserverForName("VpnStatusChanged", object: manager.connection.status, queue: NSOperationQueue.mainQueue(), usingBlock: { [unowned self] (notification) -> Void in
               self.updateStatus(manager)
               })
       }
   }
   ```

   ​

### Debug

- Extension debug不同于正常的程序，尽量使用NSlog代替print，即可在系统日志中查看到内容。同时，如果需要Debug，可通过Xcode->Debug->Attach To Process 选择你的Tunnel名进行debug

- 若在此时启动VPN时不断提示Connection intrupt、Connection invalid ，一般是因为target配置错误导致启动失败、崩溃。比如Carthage添加了framwork却没有执行copy-framwork等等……

  ​

## 四、导入NEKit

Extension中导入NEKit后进行简单配置，导入规则等等可参照NEKit的项目[Specht](https://github.com/zhuhaow/Specht)配置相应启动停止函数。注意：目前NEKit在Swift3环境下仍然存在相应问题，将导致连接在网络请求下很快崩溃断开，可考虑先使用Swift2.3版本进行操作。



## 五、监听网络状态改变

- 这里是苹果的一个大坑。当网络状态改变 例如发生 WiFi->4G 切换时，若保持VPN连接，则会在状态栏上显示一个暗色的WiFi图标。反之，则会发生让手机在WiFI连接的状态下走流量等奇怪问题。

- NEProvider 中存在属性 
  ```swift
  public var defaultPath: NWPath? { get }
  ```
  表明当前系统的网络请求路径。我们可以通过KVO监听他的改变。当defaultPath与之前状态发生改变，且当

  ```swift
  defaultPath.status == .Satisfied
  ```

  时，我们可以认定系统网络进行了切换。此时我们需要重启VPN及重启代理服务器。

  重连VPN方法非常简单，只需要再次调用StartVPN函数即可。

  ```swift
  NSLog("Network Environment Changed")
  let delayTime = dispatch_time(DISPATCH_TIME_NOW, Int64(1 * Double(NSEC_PER_SEC)))
  dispatch_after(delayTime, dispatch_get_main_queue()) {
      // 延迟1s确保系统就绪
      self.startTunnelWithOptions(nil){_ in}
  }
  ```

  同时我们需要重启proxyServer，否则会发生部分连接在WiFi环境下继续使用4G网络出口等问题。

  ```
  interface.stop()
  proxyServer.stop()
  ```



## 六、传递配置文件

我们需要在主程序中传递类似账号、密码、端口、加密方式等参数给我们的VPN组件。

- 主程序写入

  还记得之前的NETunnelProviderProtocol吗？我们可以把配置放在NETunnelProviderProtocol的providerConfiguration中

  ```swift
  let conf = ["port":1000,"method":"AES-256-CFB","password":"hello"]
  let providerProtocol = manager.protocolConfiguration as! NETunnelProviderProtocol
  providerProtocol.providerConfiguration = conf
  manager.protocolConfiguration = orignConf
  ```

  注意：设置要在save之前，否则会不生效。

- Extension中读取：

  NETunnelProvider存着我们的配置 

  ```swift
  public var protocolConfiguration: NEVPNProtocol { get }
  ```

  因此我们只要直接读取就好啦~

  ```swift
  guard let conf = (protocolConfiguration as! NETunnelProviderProtocol).providerConfiguration else{
      NSLog("[ERROR] No ProtocolConfiguration Found")
      exit(EXIT_FAILURE)
  }
  let address = conf["address"] as! String
  let port = conf["port"] as! Int
  ... 
  ```

  大功告成。

  ​

## 七、结语

此时，我们的一个非常简单的一个利用NEKit进行的简单代理软件就完成了。其实NE开发中最困难和最多坑的地方NEKit已经帮我们解决了。本文只是帮住在像我一样在进行NetworkExtension开发尝试的iOS萌新，和我自己对这两天折腾的一个记录及学习笔记。



