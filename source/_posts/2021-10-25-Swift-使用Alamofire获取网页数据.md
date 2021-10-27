---
title: Swift-使用Alamofire获取网页数据
date: 2021-10-25
tags:
    - 19级
    - iOS
author: HeKai
---

前段时间完成了iOS福大助手校招日历的更新，这里对如何使用网络请求获取福大就业信息网上的内容做一个简单的记录。

# 一、分析策略

## 1、寻找网络请求URL

我们需要从福州大学就业信息网上获取数据，打开网页[http://jycy.fzu.edu.cn/cms](http://jycy.fzu.edu.cn/cms)，进入Safari开发模式，查看页面资源，发现XHR中的"getDateZPHKeynoteList_month"为所需的json，复制其链接得到网络请求URL:"http://jycy.fzu.edu.cn/CmsInterface/getDateZPHKeynoteList_month"
![在这里插入图片描述](https://img-blog.csdnimg.cn/e9f5197aa9f84a35953495a991543c5e.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBASEhLS3Nkag==,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)

## 2、寻找网络请求参数

为了获取该月的校招日历，需要在网络请求时附上参数，否则返回数据为空。由于对网页开发不是很熟悉，寻找这个参数花费了许多时间。最后在calendar.js文件中找到了切换月份的相关调用，得到了网络请求所需的参数dateday，其格式为YYYY/MM。

![在这里插入图片描述](https://img-blog.csdnimg.cn/b94f12bc47f34ca3b79fb9470fe6a4a3.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBASEhLS3Nkag==,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)

# 二、具体实现

这部分就比较简单了，只需要用Alamofire的request进行网络请求后对返回的json进行解析就可以了。

```
func getCalendar(_ completion: @escaping (Error?, JSON?) -> ()) {
    let url = "http://jycy.fzu.edu.cn/CmsInterface/getDateZPHKeynoteList_month"
    let nowTime = NSDate()
    let format = DateFormatter()
    format.dateFormat = "YYYY/MM"
    let dateday = format.string(from: nowTime as Date) as String
    let parameters = ["dateday":dateday]
        
    AF.request(url,method: .get,parameters: parameters)
        .responseJSON { responds in
            switch responds.result {
            case .success(let value):
                print("success")
                let json = JSON(value)
                print(json)
                completion(nil, json)
            case .failure(let error):
                print("error")
                completion(error, nil)
        }
    }
}

```

