---
title: Immutable 与 Immer
date: 2018-11-06 15:47:53
author: 嫩牛五方---chs97
tags:
	- 前端
	- 15级
---
# Immutable 与 Immer

### Mutable

在前端中，对象是一个很重要的类型，为了优化以及尽量节省内存，所以对象保存的都是一个引用。比如：

```javascript
const obj = {a: 1}
const b = obj
b.a = 2
obj.a === 2// true
```

这就是副作用，`b = obj`  是将`obj`的引用赋值给`b` 所以`b`和`obj`在内存中的地址是一样的，导致了`b`修改了`obj`也会跟着修改。

在js中，有很多方法都是有副作用的。比如`Array`的方法`push shift `等等。。。这些方法都会改变原来的数组，所以叫做有副作用。副作用其实是很危险的，特别对于构建比较大型的应用以及团队合作来说，一个对象被多次引用，某个地方修改了这个对象可能导致整个应用崩溃或者产生一个无法理解的bug。举个例子。

之前做的某游戏中，物体对象有个属性，引用了一个公共的对象，在开发中对象之间的引用很常见的。

```javascript
const path = [{x: 0, y: 0}, ......]
const fish = {path: path}
// path是fish的移动轨迹，path[0]是fish的出生点
// 在过程中，fish.path进行了移动的才做 及 path[i].x += dx; path[i].y += dy
```

但是这个`dx` `dy`很小，游戏运行的时间长了以后，发现某些物体突然在屏幕中出现。

这就是引用带来的副作用，导致整个项目很难维护，A开发者不知道B开发者在哪里修改了这些对象，所以遇到需要引用一个对象的时候，尽量使用`deepCopy`

在`函数式`编程中，不可变、无副作用是一个是一个很重要的原则。

### Immutable

#### 无副作用

无副作用就是不会改变原始值包括传入的参数以及其他作用域下的变量。

在js中有许多方法就是无副作用的，比如Array的`map` `reduce`等等。

`map`返回的就是一个新的数组，而不是修改原来的数组。

`string`的方法都是无副作用的，方法都会返回一个新的string，比如`slice`，

```javascript
const str = '12345678'
str.slice(2, 3) // "3"
str // "12345678"
```

#### 不可变性

创建之后就不能修改了。如果修改了就返回一个新的对象，而不是被修改之后的旧的对象。

同样的`string`就是一个不可变的， `number`也是不可变的。

我们希望对象也能是不可变的，这样能带来许多好处。

比如不需要并发锁，因为所有对象都是独一无二的，并且是不可变的。

降低Mutable带来的复杂度，因为不需要去考虑这个对象被谁引用了，修改了他会带来什么后果。

#### 好处

##### 1.更容易检测数据变化

如果是引用，修改完的对象和旧对象是相等的。比如

```javascript
const a = {a: 1}
const b = a
b.a = 2
a === b // true
// 期望
a.a === 1 && b.a === 2 && a !== b
```

如果对象是不可变性的，就可以得到我们期望的结果，并且只需要简单的对比2个对象的引用是否一致就知道对象是否发生了变化。

##### 2.安全

所有的操作都不会影响到原对象，不会影响到其他逻辑。

#### 怎么做

不可变还是很容易的，只要对对象做一个深拷贝，就能做到immutable，但是程序涉及到的对象操作实在是太多了，每一次操作都需要拷贝一份，一个对象就是一棵树，每次拷贝一次这棵树消耗的代价实在是太大。

[Immutable.js](https://facebook.github.io/immutable-js/) 是比较流行的一个immutable的库，该库自己维护了一个数据结构，优化了每次深拷贝的消耗，对原始对象尽可能的复用。有个流行的图：

![img](http://zhenhua-lee.github.io/img/immutable/change.gif)

每次修改对象只需要对路径上的节点进行处理，其他节点都可以直接复用，这里运用了一个惰性的思想，如果这个子树没操作过就不需要拷贝，等什么时候使用到了再进行拷贝。
通过惰性以及复用的这种思想来尽量减少拷贝的操作。


### Immer
immer 是使用`Proxy`和`Object.defineProperty`来做到immutable以及懒初始化的优化。

![immer-hd.png](https://github.com/mweststrate/immer/raw/master/images/hd/immer.png)

immer有这几个过程，current state 通过 produce 函数变成 draft state ，draft state 是immer对current的一些处理，使得方便优化immutable，提高效率。immer会将current state 包装内部自己的一个对象：

```text
{
  modified, // 是否被修改过
  finalized, // 是否已经完成（所有 setter 执行完，并且已经生成了 copy）
  parent, // 父级对象
  base, // 原始对象（也就是 obj）
  copy, // base（也就是 obj）的浅拷贝，使用 Object.assign(Object.create(null), obj) 实现
  proxies, // 存储每个 propertyKey 的代理对象，采用懒初始化策略
}
```

包装之后就方便进行惰性初始化等操作了。 最后执行完所有的更改操作后，通过包装的这些对象，进行再次处理，返回 next state。

immer 通过劫持 object的getter和setter。来做到懒初始化的，当访问到某个子树的时候（把对象看成一棵树）才会对这棵子树进行一次初始化（生成代理对象）

最后返回修改后结果的一个对象，这个对象通过树上modified的标记来处理，如果这颗子树没有被标记成modified，那么会返回原始对象。举个例子：

```javascript
const currentState = {
    a: {value: 1},
    b: {value: 2}
}
```

```javascript
const nextState = produce((draft) => {
    draft.a.value = 3
})
nextState.b === currentState.b // true
// draft.a.modified = true draft.b.modified = false
// 所以在draft中 b是没有被修改过的，直接返回base
```

一个大概的思路就是：

1. 修改不影响原始对象
2. 返回新的对象
3. 性能要高（惰性 懒初始化），减少拷贝次数

#### Proxy

proxy是es6的新特性，使用方法很简单，作用就是对对象做一个代理。

```javascript
const obj = {}
const handler = {
    get(target, props) {
        return 'getter'
    }
}
const proxy = new Proxy(obj, handler)
proxy.a === 'getter' // true
```

immer的源码还是算比较少的，建议自己阅读一遍源码。主要思想就是惰性和复用，并且把对象考虑成一棵树。

#### react的优化

在react的中，有一个方法叫做`shouldComponentUpdate` 这个方法可以让我们控制该组件是否需要`rerender`，因为在`PureComponent`中是否需要`rerender`是通过对浅比较来判断是否需要重新渲染的。如下demo，点击`same count` render次数不会变，，但是点击`same data` render次数是会增加的，尽管点击前后的state是一样的。

[![Edit qkq6w3vpz9](https://codesandbox.io/static/img/play-codesandbox.svg)](https://codesandbox.io/s/qkq6w3vpz9)

如果可以通过一种方式让`state`的更改能够通过简单的比较得到是否发生过变更，就能优化掉这个渲染次数的问题了。

immer就可以做到。

[![Edit 9yl2wlpr7o](https://codesandbox.io/static/img/play-codesandbox.svg)](https://codesandbox.io/s/9yl2wlpr7o)

### 总结

immutable在前端的应用还是很广的，特别是在react上，可以做很大的性能提升。

很多人潜意识中immutable就是配合react redux等使用，但是其实并不是

[![Edit Vue Template](https://codesandbox.io/static/img/play-codesandbox.svg)](https://codesandbox.io/s/r0z448jxm4)

同样的，在handleSameData方法中，{title: '123'}对vue来说是一个新的对象，所以会发起重新渲染。

但是handleSameDataImmer这就不会进行重新渲染。

1. 对于各个框架的优化手段 其实最重要的就是一个 渲染次数，如果能降低不必要的渲染，那么应用的性能将大幅度提高。

2. 对于一些函数应该尽量是无副作用的，相同的入参的返回值应该是相同的。

### 参考

[精读《Immer.js》源码](https://zhuanlan.zhihu.com/p/34691516)

