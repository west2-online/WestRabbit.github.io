---
title: HTML块级居中大法整理
date: 2016-12-04 12:50:31
author: 二七
catalog: true
tags:
     - 14级
     - Web
---

# 一
* **最常见水平居中：**
```css
	◆margin: 0 auto;
```

* **垂直水平居中：**
```css
◆absolute/fixed（脱离文档）
.div{
	position: absolute;/fixed
	width: 100px;
	height: 100px;
	background: red;
	left: 50%;
	top: 50%;
	margin-left: -50px;
	margin-top: -50px;
}
```

优点：所有浏览器都支持
缺点：div元素必须固定宽高，灵活性差
适用：固定宽高的div

# 二
```css
◆absolute/fixed
.div{
	position: absolute;/fixed
	margin: auto;
	top: 0;
	left: 0
	bottom: 0;
	right: 0;
	width: 100px;
	height: 100px;
	background: red;
}（会根据文档自动计算距离）
```

缺点：IE6、IE7不支持
优点：高度长度可以自适应，适合用在移动端

# 三

```css
◆table-cell
.div{
	background: red;
	display: table-cell;
	vertical-align: middle;
	width: 100px;
	height: 100px;
	text-align: center;
}
```

缺点：高度无法到达100%（在外层套一层table-cell）；添加浮动后垂直居中会失效，只能实现行内元素的水平居中，块级元素不能水平居中（在块级元素中添加 margin: 0 auto就能居中）；IE7以下不支持
优点：适用的场景多

# 四
```
◆text-align: center
```
只支持行内元素，行内块级元素居中

***

CSS3居中一
针对盒子模型
```css
.div{
	height: 100px;
	display: -webkit-box;
	-webkit-box-pack: center;
	-webkit-box-align: center;
}
```
缺点：无法再IE9以下使用
优点：无论里面有多少元素都能居中
适用：除了IE浏览器都超好用

CSS3居中二
```css
.div{
	width: 100px;
	position: absolute;
	top: 50%;
	left: 50%;
	-webkit-transform: translate(-50%,-50%);
	-ms-transform: translate(-50%,-50%);
	transform: translate(-50%,-50%);
	background: red;
}
```
缺点：IE9以下不支持
优点：高度和宽度可以随意定义
适用：移动端

# 神奇的方法
```css
.div:before {
	content: ‘ ‘;
	display: inline-block;
	height: 100%;
	vertical-align: middle;
	background: #000;
}
.dived {
	display: inline-block;
	vertical-align: middle;
	width: 100px;
	background: red;
}
```

以before作为参考线居中
优点：可以随意改变宽度和高度
***