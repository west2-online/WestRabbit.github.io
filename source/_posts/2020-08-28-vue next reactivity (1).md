---
title:      "vue next reactivity (1)"
date:       2020-08-28
author:     "XieLeRu"
tags:
     - 18级
     - 前端
---

## 开始

本文将系统的了解vue next中的reactivity部分，文章内容会很长，会涉及到许多知识，阅读本篇文章至少需要

1. js原型和原型链基础
2. ts基础

对于vue3涉及到的proxy部分，最好也先有个了解，不过文章会做阐述。本篇文章是该系列的第一篇文章。

## 起步

首先对reactive有一个感性的认识，reactive即响应式，我们下面用reactive函数创建了一个proxy对象，用effect函数创建了一个副作用函数，创建的时候将会立即执行一次，当obj的text发生改变的时候这个函数将会执行

最后document.body.innerText 转变为"hello world"

~~突然感觉这里和React中的useEffect莫名相似~~

```ts
import { effect, reactive } from '@vue/reactivity'
const obj = reactive({ text: 'hello' })
effect (() => {
  document.body.innerText = obj.text
})

setTimeout(() => {
  obj.text += 'world'
}, 1000);
```

## proxy

如果你对proxy已经了解过了，可以跳过这一块

ES6原生提供Proxy构造函数，用来生成Proxy实例

```ts
const proxy = new Proxy(target, handler);
```

参数中第一个是目标对象（被代理的对象），第二个是用来定制拦截行为的

我们先看一个小例子

```ts
const obj = new Proxy({}, {
  set: function (target, propKey, value, receiver) {
    console.log(`setting ${propKey}!`);
    return Reflect.set(target, propKey, value, receiver);
  }
});
```

备注： 这里首先知道一下Reflect，对于proxy里面的拦截行为，其实在Reflect中都有一个对应的方法，这些方法是原先的默认方法，也就是如果你不自己写get这个方法，会默认执行这个方法，所以我们在做自己的事情后，也要调用原先的这个方法，保证原先默认操作的执行。

上述我们代理了一个`{}`（空）对象，并且定制了set的行为,如果我们尝试对obj进行写操作，给obj赋值count

```ts
obj.count = 1
```

那么控制台将会显示

```ts
//  setting count!
```

如果我们有下面这个函数

```ts
const fn = () => {
    const num2 = obj.num1
    document.body.innerText = num2.toString()
}
```

这个函数是依赖于obj的num1的，我们又可以通过proxy监听num1的设置（通过set方法）

这样我们就可以在里面去触发这个函数

```ts
console.log(`setting ${propKey}!`);

// 以下是替换
const oldValue = target[propKey]
const newValue = value
fn()
// ...statement
```

这样num2的值在每一次obj.num1的值发生变化的时候，都可以动态的发生改变

以上只是一个非常小的demo，我们非常初步的实现了响应式, 实际上Vue源代码非常复杂

刚才说到proxy的第二个参数handler，这个参数里面有许多可以定制的方法，具体可以参考ES文档，下面举出几种比较常用的方法

```ts
// 其中target表示目标对象，propertyKey表示键，receiver表示代理对象(proxy实例)
const get = (target, propertyKey, receiver) => {}

// value代表新的值
const set = (target, propertyKey, value, receiver) => {}

// 这个函数值对in运算符生效，对for...in循环不生效
const has = (target, key) => {}

// 对对象属性使用delete方法的时候触发
const deleteProperty = (target, key) => {}

// 在 for..in || Object.getOwnPropertyNames() || 
// Object.getOwnPropertySymbols() || Object.keys() 下触发
const ownKeys = (target) => {}
```

如果还想更一步的了解proxy，可以参考https://es6.ruanyifeng.com/#docs/proxy，这里不再赘述

## Reactive

我们接下来详细的看一下源码里的reactive是怎么实现的

```ts
export function reactive(target: object) {
  // if trying to observe a readonly proxy, return the readonly version.
  if (target && (target as Target)[ReactiveFlags.IS_READONLY]) {
    return target
  }
  return createReactiveObject(
    target,
    false,
    mutableHandlers,
    mutableCollectionHandlers
  )
}
```

首先判断一下这个传进来的对象是否是只读的，如果是只读的话，直接返回。下面是ReactiveFlags的一些定义

```ts
export const enum ReactiveFlags {
  SKIP = '__v_skip', // 跳过，不被代理
  IS_REACTIVE = '__v_isReactive', // 是响应式的
  IS_READONLY = '__v_isReadonly', // 是只读的
  RAW = '__v_raw' // 存放原始对象的引用
}
```

如果我们预先设置了这个是跳过的不被代理的，那么就不会被做成响应式的

```ts
import { ReactiveFlags, reactive, isReactive } from '@vue/reactivity'
const obj = {
  [ReactiveFlags.skip]: true
}
const proxyObj = reactive(obj)
console.log(isReactive(proxyObj)) // false
```

这些ReactiveFlags一般情况下不会用到，在一些高级场景可能会用到，因此可以不必太在意这些值。

我们看一下createReactiveObject这个方法

函数传递四个参数，第一个参数表明原始对象，第二个参数是否只读，第三个和第四个参数分别对集合元素和非集合元素传递了proxy构造函数的第二个参数handler

首先如果目标对象不是对象，那么是不可代理的，直接返回

如果目标对象已经是一个proxy了，而且不是只读的响应式proxy，这个时候可能调用了readonly方法，也需要重新生成proxy

```ts
function createReactiveObject(
  target: Target,
  isReadonly: boolean,
  baseHandlers: ProxyHandler<any>,
  collectionHandlers: ProxyHandler<any>
) {
  if (!isObject(target)) {
    if (__DEV__) {
      console.warn(`value cannot be made reactive: ${String(target)}`)
    }
    return target
  }
  // target is already a Proxy, return it.
  // exception: calling readonly() on a reactive object
  if (
    target[ReactiveFlags.RAW] &&
    !(isReadonly && target[ReactiveFlags.IS_REACTIVE])
  ) {
    return target
  }
  ...
}
```

接下来根据是否只读获取到map，判断是否已经存在于map中，存在就返回，否则调用getTargetType函数获取目标对象类型

```ts
const proxyMap = isReadonly ? readonlyMap : reactiveMap
  const existingProxy = proxyMap.get(target)
  if (existingProxy) {
    return existingProxy
  }
  // only a whitelist of value types can be observed.
  const targetType = getTargetType(target)
  if (targetType === TargetType.INVALID) {
    return target
  }
  const proxy = new Proxy(
    target,
    targetType === TargetType.COLLECTION ? collectionHandlers : baseHandlers
  )
  proxyMap.set(target, proxy)
  return proxy
```

以下是getTargetType函数以及一些依赖，当target内部有skip这个标志的时候，表示这个对象跳过，不监听

```ts
const enum TargetType {
  INVALID = 0,
  COMMON = 1,
  COLLECTION = 2
}

function targetTypeMap(rawType: string) {
  switch (rawType) {
    case 'Object':
    case 'Array':
      return TargetType.COMMON
    case 'Map':
    case 'Set':
    case 'WeakMap':
    case 'WeakSet':
      return TargetType.COLLECTION
    default:
      return TargetType.INVALID
  }
}

function getTargetType(value: Target) {
  return value[ReactiveFlags.SKIP] || !Object.isExtensible(value)
    ? TargetType.INVALID
    : targetTypeMap(toRawType(value))
}
```

这里的toRawType函数比较有意思

```ts
export const objectToString = Object.prototype.toString
export const toTypeString = (value: unknown): string =>
  objectToString.call(value)
export const toRawType = (value: unknown): string => {
  return toTypeString(value).slice(8, -1)
}
```

我们如果将对象用Object原型上的toString方法打印会出现[Object, Object]这样

通过这样我们就可以判断对象的类型，通过返回的值，我们会调用不同的handle

```ts
const proxy = new Proxy(
    target,
    targetType === TargetType.COLLECTION ? collectionHandlers : baseHandlers
  )
proxyMap.set(target, proxy)
return proxy
```

刚才我们使用了reactive方法使得数据变为响应式的，实际上，还有三种类似的方法。

| 方法            | 作用                                                  |
| --------------- | ----------------------------------------------------- |
| shallowReactive | 定义浅响应式数据                                      |
| readonly        | 定义只读的响应式数据                                  |
| shallowReadonly | 定义只读的浅响应式数据,这意味着深层次的数据可以被修改 |

接下来我们用isReactive以及isReadonly以及isProxy方法看一下他们的表现

实际上isProxy 是 isReactive 以及 isReadonly 的或

```ts
export function isReactive(value: unknown): boolean {
  if (isReadonly(value)) {
    return isReactive((value as Target)[ReactiveFlags.RAW])
  }
  return !!(value && (value as Target)[ReactiveFlags.IS_REACTIVE])
}

export function isReadonly(value: unknown): boolean {
  return !!(value && (value as Target)[ReactiveFlags.IS_READONLY])
}

export function isProxy(value: unknown): boolean {
  return isReactive(value) || isReadonly(value)
}
```

```ts
import { 
  effect,
  reactive,
  shallowReactive,
  readonly,
  shallowReadonly,
  isReactive,
  isReadonly，
  isProxy
} from '@vue/reactivity'

const reactiveProxy = reactive({ foo: { bar: 1 } })
console.log(isReactive(reactiveProxy)) // true
console.log(isReadonly(reactiveProxy)) // false
console.log(isProxy(reactiveProxy)) // true
console.log(isReactive(reactiveProxy.foo)) // true

const shallowReactiveProxy = shallowReactive({ foo: { bar: 1 } })
console.log(isReactive(shallowReactiveProxy)) // true
console.log(isReadonly(shallowReactiveProxy)) // false
console.log(isProxy(shallowReactiveProxy)) // true
console.log(isReactive(shallowReactiveProxy.foo)) // false

const readonlyProxy = readonly({ foo: 1 })
console.log(isReactive(readonlyProxy)) // false
console.log(isReadonly(readonlyProxy)) // true
console.log(isProxy(readonlyProxy)) // true

const shallowReadonlyProxy = shallowReadonly({ foo: 1 })
console.log(isReactive(shallowReadonlyProxy)) // false
console.log(isReadonly(shallowReadonlyProxy)) // true
console.log(isProxy(shallowReadonlyProxy)) // true

// 再看看他们在实际运转的时候的表现
effect(() => {
  console.log(reactiveProxy.foo.bar)
})
reactiveProxy.foo.bar = 2 // 有效
reactiveProxy.foo = { bar : 2 } // 有效

effect(() => {
  console.log(shallowReactiveProxy.foo.bar)
})
shallowReactiveProxy.foo.bar = 2 // 无效
shallowReactiveProxy.foo = { bar : 2 } // 有效

effect(() => {
  console.log(readonlyProxy.foo.bar)
})
readonlyProxy.foo.bar = 2 // Set operation on key "bar" failed: target is readonly.
readonlyProxy.foo = { bar : 2 } // Set operation on key "foo" failed: target is readonly.

effect(() => {
  console.log(shallowReadonlyProxy.foo.bar)
})
shallowReadonlyProxy.foo.bar = 2 // ok
shallowReadonlyProxy.foo = { bar : 2 } // Set operation on key "foo" failed: target is readonly.
```



## 小结

关于vue next reactivity api第一篇到这里就暂时先告一段落了，本节比较粗略的说明了proxy的用法，介绍了reactive是什么东西，看了一些源码，了解了内部的一些实现。接下来会深入这四个reactive函数内部去看proxy的handler是怎么样的。
