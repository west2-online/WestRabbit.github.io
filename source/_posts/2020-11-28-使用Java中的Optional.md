---
title: 使用Java中的Optional
date: 2020-11-28 16:24:18
author: "ZhimengChen"
catalog: true
tags:
     - 19级
     - Java
---

`NullPointerException`是非常常见的异常。由于它，程序往往需要大量使用`if-else`代码块来处理空值，这使得代码看起来不简洁 ~~优雅~~ ，且不方便自己和他人阅读。本文介绍如何用Optional类来处理`null`值问题。

## Optional类
先来看一段代码：
```
String isocode = user.getAddress().getCountry().getIsocode().toUpperCase();
```

这段代码在任何一个方法调用时，都有可能抛出`NullPointerException`。
而通常我们的处理方式是不断地利用`if`代码块来确保上一步的值不为空并执行下一步代码。
```
if (user != null) {
    Address address = user.getAddress();
    if (address != null) {
        Country country = address.getCountry();
        if (country != null) {
            String isocode = country.getIsocode();
            if (isocode != null) {
                isocode = isocode.toUpperCase();
            }
        }
    }
}
```
嗯，一股切割器cutter的味道。

Optional类是Java8为了解决`null`值判断问题，借鉴**google guava**类库的Optional类而引入的一个同名Optional类，使用Optional类可以避免显式的`null`值判断（`null`的防御性检查），避免`null`导致的NPE（`NullPointerException`）。

## 如何创建Optional实例
**Optional类没有公共构造函数。** 但是确提供了三个静态方法在不同情形下根据需求创建Optional实例。

### `Optional.of()`
这个方法要求你传入一个不为空的值（不一定是引用类型，也可以是原始类型），
所以下面这种写法还是会抛出一个`NullPointerException`异常：
```
Optional.of(null);
```

可见Optional并不能完全避免`NullPointerException`，关键在于你是否正确以及规范地使用它。
但大多数情况下，我们使用Optional正是由于无法确定值是否为空。在这种情况下，我们使用下面这个方法。

### `Optional.ofNullable()`
这个方法允许你传入空值或者非空值。

### `Optional.empty()`
这个方法会返回一个包装空值的Optional实例。也许你会觉得它会有点鸡肋（我一开始也是这么认为的）。
考虑以下代码：
```
int dividend = 10, divisor = 0;
int result = dividend / divisor;
Optional<Integer> o = Optional.of(result);
```
显然它会在运行期抛出`ArithmeticException`异常，这使得后续对于`o`的可能存在的操作因异常而终止。

改写如下：
```
int dividend = 10, divisor = 0;
Optional<Integer> o;
try {    
    int result = dividend / divisor;    
    o = Optional.of(result);
} catch (ArithmeticException e) {  
    o = Optional.empty();
}
```

## 访问Optional实例的值
### `get()`
它的源码：
```
public T get() {    
    if (value == null) {
        throw new NoSuchElementException("No value present");    
    }    
   return value;
}
```
当Optional实例包装的是一个空值时，它会抛出`NoSuchElementException`。

所以在调用`get()`方法前我们还是需要判断Optional是否包装空值。
使用`ifPresent()`方法来判断其包装的是否是空值：
```
public static String getGender(Student student) {
    Optional<Student> stuOpt =  Optional.ofNullable(student);
     if(stuOpt.isPresent()) {
            return stuOpt.get().getGender();
     }
       
     return "Unkown";
 }
```
而这其实是一种很糟糕的写法，因为这种用法**不但没有减少null的防御性检查，而且增加了Optional包装的过程，违背了Optional设计的初衷，因此开发中要避免这种糟糕的使用。** 下文会介绍相对更好的写法。

### 获取默认值
 Optional提供了两种方法来返回默认值。
 
 #### `orElse()`
 `orElse()`会在Optional有值时返回它的值，否则就会返回传入的默认值。
 ```
public class Main {
    public static void main(String[] args) {
        System.out.println(getGender(null));    
    }    
    
    public static String getGender(Student student) {
        Student stuOpt = Optional.ofNullable(student).orElse(new Student();        
        return stuOpt.getGender();    
    }
}
```

 #### `orElseGet()`
 `orElseGet()`则稍有不同，它会在Optional有值时返回其值，否则就会执行作为参数传入的Supplier实例的`get()`方法，并返回其执行结果。
```
public class Main {
    public static void main(String[] args) {
        System.out.println(getGender(null));    
    }    
    
    public static String getGender(Student student) {
        Student stuOpt = Optional.ofNullable(student).orElseGet(Student::new);      
        return stuOpt.getGender();    
    }
}
```

#### 两者的不同之处

* `orElse()`是**EAGER**的，也就是说无论Optional的值是否为空，它都会执行。
* `orElseGet()`是**LAZY**的，只有当Optional的值为空时，才会执行。

由于由以上差异，我们要根据业务场景谨慎选择，尤其是涉及服务调用或数据查询等耗时操作时。


## 参考文献

* [【java8新特性】Optional详解](https://www.jianshu.com/p/d81a5f7c9c4e)
* [Java Optional使用的最佳实践](https://www.jdon.com/52008)
* [理解、学习与使用 Java 中的 Optional](https://www.cnblogs.com/zhangboyu/p/7580262.html)

 


