---
layout:      post
title:      "Gocv--Go语言调用Opencv4"
date:       2020-11-28
author:     "mirrorlied"
catalog:     true
tags:
     - 19级
     - Golang
     - 图像处理
---

# Gocv的安装

其实[官方文档](https://gocv.io/getting-started/windows/)讲的很清楚了，但是在安装过程还是有很多坑~~。写着凑字数。~~

##### 1.安装gocv

```
go get -u -d gocv.io/xgocv
```

##### 2.安装MingGW-W64和CMake

 1.[下载MingGW](https://sourceforge.net/projects/mingw-w64/files/mingw-w64/) (要带posix、seh参数的，而且版本大于7.0)。

 2.解压到任意位置。

 3.将mingw64\bin加到环境变量。

 1.[下载CMake](https://cmake.org/download/)

 2.防呆安装，照着点就行了。

##### 3.编译Opencv

###### 	这里的安装方法与官方给出的有些区别~~，官方的装不好。~~

1.下载opencv编译相关文件（下面两个）。

[opencv](https://opencv.org/opencv-4-5-0/)	[opencv_contrib](https://github.com/opencv/opencv_contrib/releases/tag/4.5.0)

2.在C:\下新建目录opencv,在opencv下新建目录build。

3.解压刚刚下的两个压缩包到C:\opencv。

4.**把Python相关的环境变量改掉**（比如在后面加个1）。

​		(如果Python安装过Opencv这里就装不上)

5.开始编译

```
cd C:\opencv\build
mingw32-make
ming-make install
```

如果没报错，编译完后把C:\opencv\build\install\x64\mingw\bin丢进环境变量。

如果上面报错

```
fatal error: boostdesc_bgm.i: No such file or directoryc
```

下载缺失文件丢进C:\opencv\opencv_contrib\modules\xfeatures2d\src

​		链接：https://pan.baidu.com/s/1_nipZrmFPGkYma1NpgCq4A
​		提取码：t2r5

然后重复操作5。

6.测试

```
cd %GOPATH%\src\gocv.io\x\gocv
go run cmd\version\main.go
```

若安装成功会返回

```
gocv version: 0.25.0
opencv lib version: 4.5.0
```

7.把Python环境变量调回去~~(其实可以不要，我不学Python啦)~~

# Gocv实战

下面将以背景差分法实现运动目标检测为例展示Gocv的基本用法。

一、连接摄像头，并创建窗口

```go
// 设备号（0为默认摄像头）
deviceID := 0

// webcam：摄像头，下面会从这里读取源图。
webcam, err := gocv.OpenVideoCapture(deviceID)
if err != nil {
   fmt.Printf("无法打开摄像头: %v\n", deviceID)
   return
}
defer webcam.Close()

// window：窗口，显示图片。
window := gocv.NewWindow("Main")
defer window.Close()
```

二、新建图形矩阵

```go
// gocv.NewMat(): 新建矩阵。

// img：摄像头截获到的源图。
img := gocv.NewMat()
defer img.Close()
// imgDelta：前景图。
imgDelta := gocv.NewMat()
defer imgDelta.Close()
// imgThresh：二值化后的图像。
imgThresh := gocv.NewMat()
defer imgThresh.Close()
```

三、构建混合高斯背景建模的背景减除法

```go
// gocv.NewBackgroundSubtractorMOG2WithParams(history, varThreshold, detectShadows)
// history：用于训练背景的帧数，默认为500帧。
// varThreshold：方差阈值，用于判断当前像素是前景还是背景，默认为16。值越大越灵敏，用于光照变化大的情况；值小相反。
// detectShadows：是否检测影子，检测复杂度加倍（大概），一般设为false。

mog2 := gocv.NewBackgroundSubtractorMOG2WithParams(500, 25, false)
// mog2 := gocv.NewBackgroundSubtractorMOG2() //默认参数的。
defer mog2.Close()
```

四、构建循环捕捉、处理并展示

```go
for {
   if img.Empty() {
       // 源图为空就跳过，免得影响后面的计算。
      continue
   }

   // 清理背景图像，获取运动的物体（也就是前景）。
   mog2.Apply(img, &imgDelta)
   // 设置阈值进行二值化（黑/白）。
   gocv.Threshold(imgDelta, &imgThresh, 25, 255, gocv.ThresholdBinary)

   // 膨胀操作（填补凹洞，没有也行，效果可能会差一点）。
   gocv.Dilate(imgThresh, &imgThresh, gocv.NewMat())

   // 找到运动物体的轮廓(们)。
   contours := gocv.FindContours(imgThresh, gocv.RetrievalExternal, gocv.ChainApproxSimple)
   for i, c := range contours {
      area := gocv.ContourArea(c)
      // 过滤太小的轮廓。
      if area < 3000 {
         continue
      }
		// 将轮廓画在源图上
      gocv.DrawContours(&img, contours, i, color.RGBA{0, 0, 255, 0}, 2)
		// 加个矩形边框
      rect := gocv.BoundingRect(c)
      gocv.Rectangle(&img, rect, color.RGBA{255, 0, 0, 0}, 2)
   }
	// 窗口展示源图。
   window.IMShow(img)
    //检测退出键，退出程序。
   if window.WaitKey(1) == 27 {
      break
   }
}
```

五、完整的代码实现（额外加了一点点东西：提示词和其他预览窗口）

```go
package main

import (
   "fmt"
   "gocv.io/x/gocv"
   "image"
   "image/color"
)

const MinimumArea = 3000

func main() {
   deviceID := 0

   webcam, err := gocv.OpenVideoCapture(deviceID)
   if err != nil {
      fmt.Printf("无法打开摄像头: %v\n", deviceID)
      return
   }
   defer webcam.Close()

   window := gocv.NewWindow("Main")
   defer window.Close()

    // 多开两窗口展示图片变化过程
   Delta := gocv.NewWindow("Delta")
   defer Delta.Close()
   Thresh := gocv.NewWindow("Thresh")
   defer Thresh.Close()

   img := gocv.NewMat()
   defer img.Close()
    
   imgDelta := gocv.NewMat()
   defer imgDelta.Close()
    
   imgThresh := gocv.NewMat()
   defer imgThresh.Close()

   mog2 := gocv.NewBackgroundSubtractorMOG2WithParams(500, 25, false)
   defer mog2.Close()

   status := "ready!!"    //初始提示词
   statusColor := color.RGBA{0, 255, 0, 0}  //提示词颜色
   fmt.Printf("开始读取设备: %v\n", deviceID)
   for {
      if ok := webcam.Read(&img); !ok {
         fmt.Printf("设备已关闭: %v\n", deviceID)
         return
      }
      if img.Empty() {
         continue
      }

      status = "ready!!" // 默认为ready
      statusColor = color.RGBA{0, 255, 0, 0} // 默认绿色

      // 获取前景
      mog2.Apply(img, &imgDelta)
      // 展示前景
      Delta.IMShow(imgDelta)

      // 二值化
      gocv.Threshold(imgDelta, &imgThresh, 25, 255, gocv.ThresholdBinary)

      // 膨胀
      gocv.Dilate(imgThresh, &imgThresh, gocv.NewMat())

      // 轮廓
      contours := gocv.FindContours(imgThresh, gocv.RetrievalExternal, gocv.ChainApproxSimple)
      for i, c := range contours {
         area := gocv.ContourArea(c)
         // 过滤
         if area < MinimumArea {
            continue
         }

         status = "move!!"
         statusColor = color.RGBA{0, 0, 255, 0}
          // 画轮廓
         gocv.DrawContours(&img, contours, i, statusColor, 2)
		  // 画边框
         rect := gocv.BoundingRect(c)
         gocv.Rectangle(&img, rect, color.RGBA{255, 0, 0, 0}, 2)
      }
       
		// 画提示词
      gocv.PutText(&img, status, image.Pt(10, 50), gocv.FontHersheyPlain, 2.4, statusColor, 2)
		// 展示膨胀后图像
      Thresh.IMShow(imgThresh)
       	// 展示源图
      window.IMShow(img)
      if window.WaitKey(1) == 27 {
         break
      }
   }
}
```

![预览图](result.png)

六、写一个WebSocket把图片投上去就可以直播了！！（bushi）

![websocket实现直播](websocket.png)

<span style="color:#aaa;font-size:13px;"><del>所以为啥不用python呢</del>...</span>