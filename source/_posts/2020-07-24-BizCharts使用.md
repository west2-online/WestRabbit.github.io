---
title: BizCharts使用
date: 2020-7-24 18:24:06
author: ElizzF
tags:
	- 前端
	- 18级
---

## 前言
在做项目的时候要求要柱状图、雷达图展现数据，因为当时是用vue，所以就用了ECharts和v-charts。在使用过程中无意中发现了BizCharts，因为它还是React图表库，所以在学习React过程中也顺带尝试使用了一下。

## 一、引入
[官方文档](https://www.bizcharts.net/index)目前给出两种引入方式：

 1. 在 BizCharts 的 GitHub 上下载最新的 release 版本 [https://github.com/alibaba/BizCharts.git](https://github.com/alibaba/BizCharts.git)
 2. 通过 npm 获取 BizCharts，官网提供了 BizCharts npm 包，通过下面的命令即可完成安装`npm install bizcharts --save`
 
成功安装完成之后，即可使用 import 或 require 进行引用。

## 二、组件初始化

 **1. 创建容器**
 在页面的 body 部分创建一个节点，指定一个 id


```javascript
<div id="mountNode"></div>
```
**2. 使用组件生成图表**
 - 引入图表需要的组件
 - 用组件组装成需要的图表
 - 把图表渲染到 mountNode 节点上
 

```javascript
import React from 'react';
import ReactDOM from 'react-dom';
import { Chart, Geom, Axis, Tooltip, Legend, Coord } from 'bizcharts';

// 数据源
const data = [
  { genre: 'Sports', sold: 275, income: 2300 },
  { genre: 'Strategy', sold: 115, income: 667 },
  { genre: 'Action', sold: 120, income: 982 },
  { genre: 'Shooter', sold: 350, income: 5271 },
  { genre: 'Other', sold: 150, income: 3710 }
];

// 定义度量
const cols = {
  sold: { alias: '销售量' },
  genre: { alias: '游戏种类' }
};

// 渲染图表
ReactDOM.render((
  <Chart width={600} height={400} data={data} scale={cols}>
      <Axis name="genre" title/>
      <Axis name="sold" title/>
      <Legend position="top" dy={-20} />
      <Tooltip />
      <Geom type="interval" position="genre*sold" color="genre" />
    </Chart>
), document.getElementById('mountNode'));
```
## 三、遇到的坑（暂时）
在引入成功后，我就以之前项目需求来开始研究这个图表。
于是就遇到了以下的坑（我知道这都是暂时的表象）：
**1. 图表宽度不自适应**

![在这里插入图片描述](https://img-blog.csdnimg.cn/2020030420052257.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mzg4OTI4Ng==,size_16,color_FFFFFF,t_70)
就像上面这种没有自适应的情况，之前用ECharts时遇到要自适应也挺麻烦的，这次遇到就仔细查看了一下官方文档，发现这么一个属性：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200304200936524.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mzg4OTI4Ng==,size_16,color_FFFFFF,t_70)
于是加入了以后成功自适应。

```javascript
<Chart height={400} data={data} scale={cols} forceFit>
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200304201025117.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mzg4OTI4Ng==,size_16,color_FFFFFF,t_70)
**2. 元素过多宽度不够图表爆炸**
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200304211402923.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mzg4OTI4Ng==,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200304211458123.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mzg4OTI4Ng==,size_16,color_FFFFFF,t_70)
很明显看到一旦宽度不够，图表底部legend会不见几个，文字则已经挤成一团。这种情况目前我还没找到直接的解决方法，如果出现这种情况，我目前只能把他们分类，限制选择个数，让用户自己选择来展示。
## 四、惊喜的发现
**1. 疑似战胜了雷达图文字越界**
在使用其它诸如ECharts的雷达图时，我心很累的发现一旦限制了外层的大小，雷达图周围的文字就会越界：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200304205107842.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mzg4OTI4Ng==,size_16,color_FFFFFF,t_70)
此刻我可以通过改变图表大小等属性暂时解决，但是当出现更多文字的时候，不可避免的再度越界。
而在使用BizCharts的雷达图后，我发现只要使用了上文说的`forceFit`，就没有这个问题的出现了。（也许是还没发现）
## 结语
之前很开心地发现了这个图表库。自我感觉比Echarts好用的多。结果现在有点绝望，每个图表库都有各自的缺点和优点。另外在研究过程中发现了一个前辈的采坑过程，里面很多虽然我没有用到但是真的有用：[https://juejin.im/post/5c0f45edf265da61327f285c](https://juejin.im/post/5c0f45edf265da61327f285c)
