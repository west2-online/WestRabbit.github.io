---
title: 设计模式之简单工厂模式
date: 2016-11-21 23:00:00
tags: 
    - 设计模式
    - 14级
author: RuphiLau
---

## 一、工厂模式的应用场景
> 可以统一管理对象的实例化
> 1、一个接口有多个实现类，使用者在使用的时候，可以传给工厂一个参数，工厂根据这个参数来选择具体的实现类
> 2、一个项目中，new了成百上千个某接口的实现类，然后突然有一天，要把这个实现类换名字了，那么是非常可怕的，因为需要修改成百上千个文件，这时候，如果使用工厂模式，则只要修改一个地方就可以

## 二、参与者
1、接口，规范子类需要实现的方法，同时利用多态的特性，来调用子类
2、接口的实现类
3、工厂类，根据用户需求来分配实现类
4、用户类，用户类只需要统一使用工厂类，即可得到具体的实现类

## 三、实例说明
以形状接口为例，它的实现类是圆形和方形
1、首先，我们要定义一个形状接口，接口中要求实现类能够实现对自身形状的描述，代码如下：
```java
interface Shape {
    public void desc();
}
```
2、我们再来实现圆形和方形，它们均遵守形状接口
```java
class Circle implements Shape {
    public void desc() {
        System.out.println('我是一个圆形');
    }
}
class Square implements Shape {
    public void desc() {
        System.out.println('我是一个方形');
    }
}
```
3、再来定义一个形状工厂，工厂中，返回值应该是接口的类型（多态的特性），然后提供一个由用户提供的选择参数shapeName，工厂则根据这个参数来返回对象，代码如下：
```java
class ShapeFactory {
    public static Shape getInstance(String shapeName) {
        if(shapeName.equalsIngoreCase('circle')) {
            return new Circle();
        } else if(shapeName.equalsIngoreCase('square')) {
            return new Square();
        }
        return null;
    }
}
```
4、接下来就是享受使用工厂模式带来的便利的时候了，且看如下代码：
```java
public class Demo {
    public static void main(String[] args) {
        Shape s1 = ShapeFactory.getInstance('circle');
        Shape s2 = ShapeFactory.getInstance('square');
 
        s1.desc();
        s2.desc();
        // 最后的输出为：
        // 我是一个圆形
        // 我是一个方形
    }
}
```

好处是显然的，比方说，我们在创建circle类的同时，还需要进行一系列复杂的操作，采用工厂模式，就可以把这些操作集合起来放在工厂里，从而用户使用的时候就无需关心内部的实现细节。同时，如果某天，Circle类要改名为SuperCircle了，那么我们也只要在工厂里修改一下就可以，从而可以避免对大量文件进行修改
