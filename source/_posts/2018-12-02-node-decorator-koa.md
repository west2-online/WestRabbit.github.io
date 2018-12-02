---
title: 基于koa实现一个装饰器风格的框架
date: 2018-12-02 22:04:53
author: chs97
tags:
	- Web
	- 15级
    - 嫩牛五方
    - Node
---

## 了解装饰器

装饰器（Decorator）是用来修改类行为的一个函数（语法糖），在许多面向对象语言中都有这个东西。

### 语法

装饰器是一个函数，接受3个参数`target` `name` `descriptor`

- target是被修饰的目标对象
- name是被修饰的属性名
- descriptor是属性的描述

定义一个装饰器函数

```typescript
function setName(target, name, descriptor) {
    target.prototype.name = 'hello world'
}

@setName
class A {
    
}
console.log((new A()).name) // hello world
```

[![Edit TypeScript playground](https://codesandbox.io/static/img/play-codesandbox.svg)](https://codesandbox.io/s/m436r9nnyy)

### 差异

装饰器装饰不同类型的目标是有一些差异的，这些差异体现在装饰函数接受的参数里面。

[![Edit TypeScript playground](https://codesandbox.io/static/img/play-codesandbox.svg)](https://codesandbox.io/s/zq53nz8vj3)

首先对一个类的装饰是由内到外的，先从类的属性开始，从上到下，按顺序修饰，如果类的属性是个方法，那么会先装饰这个方法的属性，再装饰这个方法。如上demo的console

#### 装饰Class

装饰函数接收到的参数 `target`是类的本身，`name`与`descriptor`都是`undefined`

#### 装饰Class的属性

装饰函数接收到的参数`target`是类的原型，也就是`class.prototype`

 `name`为该属性的名字

当这个属性是个函数时：

`descriptor`为该方法的描述，通过`Object.getOwnPropertyDescriptor(obj, prop)`获得

当这个属性非函数时:

`descriptor`为`undefined`

#### 装饰Class方法的参数

装饰函数接受到的参数`target`是类的原型

`name`为该参数的名字

`descriptor`为该参数是这个函数的第几个参数，`index:number`

## 了解Reflect.metadate

Reflect可以理解为反射，可以改变`Object`的一些行为。

[reflect.metadata](https://github.com/rbuckton/reflect-metadata)从名字上看，就是对对象设置一些元数据。

有2个比较重要的api

`Reflect.getMetadata(key, target)`通过`key`获得在`target`上设置的元数据

`Reflect.defineMetadata(key, value, target)`通过`key`设置`value`到`target`上

实现这个2个api不难，通过`weakMap`和`Map`就可以实现了。

这样的数据结构

`weakMap[target, Map[key, value]]`

## koa路由

koa的中间件模型不做介绍，`koa-router`就是个中间件。

路由其实就是映射一个`controller`方法到一个`path`字符串上。

通过`ctx`去`match`匹配到的`path`然后调用这个`controller`方法。

## 简单的例子

在这个例子里面，通过装饰器，来实现绑定一个`Controller`方法到路由上。

首先如上所说的，有以下思路：

1. 装饰器记录`Controller`元数据，实现一个Bind方法，取出元数据绑定到路由上

实现一个装饰器`Router(path)`用来装饰`Controller`的方法

```typescript
import * as koa from "koa";
import * as router from "koa-router";

const koaRouter = new router();
const app = new koa();

function Router(path) {
  return function(target, name) {};
}

function bind(router, controller) {}
class Controller {
  @Router("/hello")
  sayHello(ctx) {
    ctx.body = "say hello";
  }
}
bind(koaRouter, Controller);
app.use(koaRouter.routes());
app.listen(8080);

```

来实现`bind`方法和`Router`装饰器

首先是`Router`装饰器

```typescript
function Router(path) {
  return function(target, name) {
    Reflect.defineMetadata("path", { path, name }, target);
  };
}
// 装饰器如果需要传参得再装饰器上层封装一个函数，然后再返回这个装饰器函数
```

使用`Reflect.metadata`需要在程序的开始`import "reflect-metadata";`

首先是`bind`

```typescript
function bind(router, controller) {
  const meta = Reflect.getMetadata("path", controller.prototype);
  console.log(meta);
  const instance = new controller();
  router.get(meta.path, ctx => {
    instance[meta.name](ctx);
  });
}
```

这里的`bind`也很简单，首先是，装饰器装饰一个方法的`target`是类的原型，所以这边`getMetadata`的`target`应该是`controller.prototype`，meta的属性`path`对应的是`/hello` `name`对应的是`sayHello`，然后就是实例化`controller`，然后通过router去绑定这个`path`和方法。

[![Edit node typescript](https://codesandbox.io/static/img/play-codesandbox.svg)](https://codesandbox.io/s/kw9lm323mr)

打开例子在右边的浏览器输入`/hello`就能看到`say hello`的输出。

## 进入正题

进入正题，开始封装一个不是那么完整的装饰器框架。

先定义一堆的`constants`

```typescript
export enum METHODS {
    GET = 'get',
    POST = 'post',
    PUT = 'put',
    DEL = 'del',
    ALL = 'all'
}

export const PATH = 'DEC_PATH'
export const PARAM = 'DEC_PARAM'
```



### 请求方法

首先是各种请求方法`GET` `POST` `PUT` `DELETE` 

因为现在有了请求方法的区分，所以在收集信息的时候需要加一个字段。

现在收集信息的方法变为

```typescript
import { METHODS, PATH } from "./constants";

export function Route(path: string, verb: METHODS) {
    return function(target, name, descriptor) {
        const meta = Reflect.getMetadata(PATH, target) || []
        meta.push({
            name,
            verb,
            path
        })
        Reflect.defineMetadata(PATH, meta, target)
    }
}
```

可以看见，多了一个`verb`参数表示该`controller`的请求方法

这边用数组是因为，`target`只有这个`controller`要记录的信息不止一个有很多。

通过这个基础方法，再封装一下其他装饰器

```typescript
export function ALL(path: string) {
    return Route(path, METHODS.ALL)
}

export function GET(path: string) {
    return Route(path, METHODS.GET)
}

export function POST(path: string) {
    return Route(path, METHODS.POST)
}

export function PUT(path: string) {
    return Route(path, METHODS.PUT)
}

export function DEL(path: string) {
    return Route(path, METHODS.DEL)
}
```

装饰器写完，这里的`bind`应该和之前的不一样，毕竟`metadata`是个数组，处理起来其实没有区别，加个循环罢了。

```typescript

import * as Router from 'koa-router'
import * as Koa from 'koa'
import { PATH } from './constants';
 export function BindRoutes(koaRouter: Router, controllers: any[]) {
     for(const ctrl of controllers) {
         const pathMeta = Reflect.getMetadata(PATH, ctrl.prototype) || []
         console.log(pathMeta)
         const instance = new ctrl()

         for(const item of pathMeta) {
             const { path, verb, name } = item
             koaRouter[verb](path, (ctx: Koa.Context) => {
                instance[name](ctx)
             })
         }
     }
 }
```

这里的`pathMeta`的输出：

```
[ { name: 'sayHello', verb: 'get', path: '/hello' },
  { name: 'postMessage', verb: 'post', path: '/post' },
  { name: 'putName', verb: 'put', path: '/put' },
  { name: 'delMe', verb: 'del', path: '/del' } ]
```

[![Edit node typescript](https://codesandbox.io/static/img/play-codesandbox.svg)](https://codesandbox.io/s/vm25l81063)

点开例子右边的浏览输入`/get`就能预览得到，控制台也打印出来上面的输出。

### 请求参数

请求方法处理完了，处理一下请求参数。

举个例子

```typescript
getUser(@Body() user, @Param('id') id) {
    
}
```

想要的是，这个`user`参数自动变成`ctx.body`, `id`变为`ctx.params.id`。

如上，绑定路由的时候，`controller`的参数是传进去的，并且，在装饰器对函数参数进行装饰的时候，可以通过`descriptor`获得到这个参数在所有参数里面的第几个位置。所以通过这些特性，可以实现想要的需求。

只要把`bind`方法改写成：

```typescript
instance[name](arg1, arg2, arg3)
// arg1 = ctx.body
// arg2 = ctx.params.id
// arg3 = .....
```

所有能从ctx中获取到的，都可以`ctx.body` `ctx.params` `ctx.query`

同样的，实现一个基础方法，叫做`Inject`来收集参数的信息

```typescript
export function Inject(fn: Function) {
    return function(target, name, descriptor) {
        const meta = Reflect.getMetadata(PARAM, target) || []
        meta.push({
            name,
            fn,
            index: descriptor
        })
        Reflect.defineMetadata(PARAM, meta, target)
    }
}
```

这里的的`fn`必须是个函数，因为需要通过请求的`ctx`拿到需要的值。这里的`index`是该变量在参数中的位置。

实现了`Inject`接下来继续实现其他的装饰器

```typescript
export function Ctx() {
    return Inject(ctx => ctx)
}

export function Body() {
    return Inject(ctx => ctx.request.body)
}

export function Req() {
    return Inject(ctx => ctx.req)
}

export function Res() {
    return Inject(ctx => ctx.res)
}

export function Param(arg) {
    return Inject(ctx => ctx.params[arg])
}

export function Query(arg) {
    return Inject(ctx => ctx.query[arg])
}
```

这些装饰器都很简单，都是基于`Inject`，这个装饰器的函数会先收集起来，后面会用到。

通过自己实现的`bind`函数可以很容易的把需要的参数传入到`controller`中

看一下修改以后的`bind`函数

```typescript

import * as Router from 'koa-router'
import * as Koa from 'koa'
import { PATH, PARAM } from './constants';
 export function BindRoutes(koaRouter: Router, controllers: any[]) {
     for(const ctrl of controllers) {
         const pathMeta = Reflect.getMetadata(PATH, ctrl.prototype) || []
         const argsMeta = Reflect.getMetadata(PARAM, ctrl.prototype) || []
         console.log(argsMeta)
         const instance = new ctrl()

         for(const item of pathMeta) {
             const { path, verb, name } = item
             koaRouter[verb](path, (ctx: Koa.Context) => {
                const args = argsMeta.filter(i => i.name === name).sort((a, b) => a.index - b.index).map(i => i.fn(ctx))
                instance[name](...args, ctx)
             })
         }
     }
 }
```

`args`先`filter`出这个`controller`方法有关的参数，再根据这些参数的`index`排序，排序以后就是`args[i]`的fn函数`ctx => ctx.xxx`的形式，通过执行`fn(ctx)`可以拿到需要的值。

最后执行`controller`的时候把这些值传入，就得到了想要的结果。

所以上面`bind`函数的`args`就是通过装饰器得到的所需要的参数。

这样来使用它们：

```typescript
import { GET, PUT, DEL, POST, Ctx, Param, Body } from "../src";

export class Controller {
    @GET('/:id')
    sayHello (@Ctx() Ctx, @Param('id') id, @Query('name') name) {
        Ctx.body = 'hello' + id + name
    }

    @POST('/post')
    postMessage(@Body() body, ctx) {
        console.log(body)
        ctx.body = 'post'
    }
}
```

当请求进入`sayHello`绑定的路由的时候， `sayHello`会被执行，并且会传入以下参数执行。

`sayHello(ctx, ctx.params['id'], ctx.query['name'], ctx)`

至此，就封装出了一个很简陋的装饰器风格的框架。

[![Edit node typescript](https://codesandbox.io/static/img/play-codesandbox.svg)](https://codesandbox.io/s/xjk5470wrw)

可以在右边的浏览地址输入`123?name=chs97`可以看到`hello123chs97`

## 总结

装饰器还可以做很多事情，在这里主要使用装饰器来记录一些信息，然后通过其他方法获取这些信息出来，进行处理。

装饰器风格的框架可以参考`nestjs` 这是一个完全装饰器风格的框架，和`Sprint boot`非常像，可以尝试体验一下。

还有一些装饰器风格的库：

1. [Typeorm](https://github.com/typeorm/typeorm)装饰器风格的`ORM`框架
2. [routing-controllers](https://github.com/typestack/routing-controllers) 装饰器风格的框架可以使用`express`和`koa`做底层
3. [trafficlight](https://github.com/swimlane/trafficlight)装饰器风格的框架，底层为koa
4. [nestjs](https://github.com/nestjs/nest)一个非常棒的node框架，开发体验非常好