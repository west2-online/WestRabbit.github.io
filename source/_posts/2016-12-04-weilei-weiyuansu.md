---
title: 伪类和伪类元素的学习笔记
date: 2016-11-23 19:22:31
author: 二七
catalog: true
tags:
     - 14级
     - Web
---

* **伪类：是一个真实 HTML 元素上的一个特殊的状态。可以认为是浏览器在特定条件下将一个虚拟的类自动应用于某个元素。它的功能和class有些类似。**

* **伪元素：是 HTML文档的一部分，尽管它不是真实的 HTML 元素，但是 CSS 允许你为它设置样式。就像是虚拟的 HTML 元素——尽管它没有真实的 HTML标签，但你仍可以为其添加样式。**

# 伪类及伪元素

## 伪类

>:link
>:hover
>:active
>:visited
>:focus
>:first-child
>:lang
>:nth-child(even)
>:first-child

***

## 伪元素

>::first-letter
>::first-line
>::before
>::after

***

## 常用的伪类的使用方法：

```html
<style>
.div1 {
	border: 1px solid #000;
}
.div2 {
	float: left; /*当内部元素有浮动时div1的border会失效*/
}
.div1:after {
	content: '11';
	width: 1px;
	opacity: 0; /*可以用伪类来达到清除浮动的效果而不用多写一个元素*/
}
</style>

<body>
<div class="div1">
	<div class="div2">2333</div>
</div>
</body>
```

* **所以清除浮动：**

```css
.clearfix:after { display: block; clear: both; content: ""; visibility: hidden; height: 0 }
```

* **first-line**
将一整块内容中的第一行挑出来写不一样的样式，可以省去因为第一行内容变多变少然后还要改为了设置样式弄得<span>或者<h2>等等的位置什么的

同理还有first-child（有多个div1，取第一个div1设置样式），first-letter（第一个字母）等等

* **用于修改input样式**
因为伪类和伪元素具有css和class的部分功能，可以叠用
（注意双冒号 ：： 是伪元素）
例如

```
	input[type=”search”]::-webkit-search-results-decoration::after
```

或者

```
	input[type=”search”]:focus::-webkit-search-results-decoration::after
```

可以省去更多的html的块元素，使得代码更加简洁