---
title: "JavaScript的函数柯里化"
date:  2021-10-30 16:23:30
tags:
- 19 级
- JavaScript
- React
author: "郑世杰"
---

# 函数柯里化

## 1、高阶函数

所谓高阶函数，就是函数中可以传入另一个函数作为参数的函数。

> * 若一函数，接受的参数也是一个函数，则函数为高阶函数
> * 若一函数，返回值也是一个函数，则函数为高阶函数

```js
//接受的参数为函数
function add1(x, y, f) {
    return f(x) + f(y);
}
add1(-5, 6, Math.abs);  // 11

//返回值为函数
function add2(x){
    return function(y){
        return x + y
    }
}
add2(4)(6)  // 10
```



## 2、函数柯里化的概念

柯里化是一种高阶函数的特殊用法。柯里化是把接受多个参数的函数变为接受一个参数的函数，然后返回接受剩下参数的函数的技术。

```js
function multiply(a,b,c){
    return a,b,c;
}
//柯里化后
function myMultiply(a){
    return (b)=>{
        return (c)=>{
            return a*b*c;
        }
    }
}
```

#### 柯里化应用的好处

1.  避免重复使用具有相同参数的函数

   ```js
   var func = myMultiply(5)(6);
   func(7)  // 5*6*7
   func(8)  //5*6*8
   func(9)  //5*6*8
   ```

2. 避免重复执行某些代码

   ```js
   //浏览器的兼容问题
   var func = (function(){
       if (document.addEventListener) {
           return function(element, event, handler) {
               if (element && event && handler) {
                   element.addEventListener(event, handler, false);
               }
           };
       } else {
           return function(element, event, handler) {
               if (element && event && handler) {
                   element.attachEvent('on' + event, handler);
               }
           };
       }
   })();
   ```

   

## 3、React中函数柯里化的应用

#### React事件绑定

```jsx
//修改state
saveName = (event)=>{
	this.setState({name:event.target.value})
}
savePassword = (event)=>{
	this.setState({password:event.target.value})
}
<input onChange="{this.saveName}" type="text" name="name" />

//柯里化后
save = (dataType)=>{
    return (event)=>{
        this.setState({[dataType]:event.target.value})
    }
}

<input onChange="{this.save('name')}" type="text" name="name" />
```

> 避免了重复定义相似的函数

#### 补充

不用柯里化的事件绑定

```jsx
saveDate = (dataType,value)=>{
    this.setState({[dataType]:value});
}
<input onChange="{(event)=>{this.saveDate('name',event.target.value)}}" type="text" name="name" />
```

