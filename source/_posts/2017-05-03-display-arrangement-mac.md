---
title:      "使用CGBeginDisplayConfiguration切换 Mac 显示器排列"
subtitle:   "什么？没有需求？那就自己创造需求啊！"
date:       2017-5-2 23:30:00
author:     "yicheng"
catalog:     true
tags:
     - 14级
     - Mac
---

> 有遇到这种情况吗？当你有一台外借显示器和一台 Macbook，而且你的显示器是有支架可以移动的！有时候，我把笔记本放在显示器右边写代码，而有时候，我把显示器放在电脑正上方码字，而有时候，我还想把显示器竖起来？？（喂，你怎么那么无聊啊！）感觉每次到设置-显示器排列里面拖太麻烦了？（其实并没有 QAQ）让我们用简单的几行 Swift 代码实现一个 Mac 程序剞劂他吧！

# 获取目前显示器信息

- 获取显示器配置比较简单，我们可以通过一行代码搞定它。

```
    NSScreen.screens()
```

- 获取的结果是一个 NSScreen数组。
- 对于每个 NSScreen 对象，我们可以用screen.deviceDescription 获取显示器信息的 NSDictionary 基本信息字典，从而获取我们需要的基本信息，长这样。
```Swift
    let screen_info = screen.deviceDescription
    print(screen_info)

// ["NSDeviceBitsPerSample": 8, "NSDeviceColorSpaceName": NSCalibratedRGBColorSpace, "NSScreenNumber": 69680128, "NSDeviceSize": NSSize: {1440, 900}, "NSDeviceResolution": NSSize: {72, 72}, "NSDeviceIsScreen": YES]


```

``` Swift
        screen_id = screen_info["NSScreenNumber"] as! CGDirectDisplayID // screen_id 用来指定要配置的显示器

        let screen_size = screen_info["NSDeviceSize"] as! NSSize
        screen_width = Int32(screen_size.width)
        screen_height = Int32(screen_size.height)
        
        let position = screen.frame
        position_x = Int32(position.origin.x)
        position_y = Int32(position.origin.y)
```
- 之后，自己处理这些信息的保存，用于后续一键配置显示器排列中使用~

# 配置显示器信息

- 首先我们要创建一个 config 指针用于后续各种配置

    ```Swift
    import AppKit

    let config = UnsafeMutablePointer<CGDisplayConfigRef?>.allocate(capacity:1)
    ```

- 调用 CGBeginDisplayConfiguration开始配置显示器
    ```Swift
    CGBeginDisplayConfiguration(config)
    ```

    现在开始配置。

    ```Swift
     for display in displays{ //假设这里有个 display s 数组存储了各个显示器的相关信息（display 为自定义结构体）
                
                let error:CGError
                
                if display.position_x == 0 && display.position_y == 0{
                    
                    //坐标为（0，0）的显示器可以直接设置无需计算坐标矫正
                    error = CGConfigureDisplayOrigin(config.pointee, display.screen_id, 0, 0)
                   
                }else{
                    //其他显示器需要计算坐标偏移，将在下面讲解。
                    error = CGConfigureDisplayOrigin(config.pointee, display.screen_id, display.position_x, -display.position_y - display.screen_height + main_height)
                }
                
                if error != .success{
                    //若配置失败，取消配置，防止内存泄漏等现象
                    CGCancelDisplayConfiguration(config.pointee)
                    return
                }
                
    }

    ```

    为什么需要计算坐标偏移呢？
    如下图，在我们之前通过 NSScreen.screens 获取的 Screen 坐标中，坐标系的原点是主显示器的左下角，也就是说所有显示器都是以左下角为基准计算偏移量的。如下图。

    ![img](downleft.png)

    而我们通过 api 进行排列设置的时候，很坑的是此时系统坐标系是以显示器的左上角为原点计算偏移量的！如下图。注意此时主显示器设置的坐标依然是(0,0),而其他显示器的坐标的 y 轴就需要进行小小的转换。对了,系统坐标系中，x 坐标轴是从左往右，而` y 轴坐标是从上往下`。不要弄错了 ORZ。

    ![img](upleft.png)

    于是，目前来看，公式应该是这样的（简单小学数学？）主显示器 y 轴分辨率 - 所设置显示器竖向辨率大小 - 所设置显示器之前所获得y轴坐标

    - 完成显示器配置

    ``` Swift
        let error = CGCompleteDisplayConfiguration(config.pointee, .permanently) //设置配置为永久
            if error != .success{
                CGCancelDisplayConfiguration(config.pointee)
            }

    ```

    ## Finished！