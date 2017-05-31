---
title: Java设计模式
date: 2017-05-31 12:49:34
author: "Csming"
catalog: true
tags:
     - 14级
     - Java
---
>模式是在特定环境下人们解决某类重复出现问题的一套成功或有效的解决方案
>在软件开发生命周期的每一个阶段都存在着一些被认同的模式

# 什么是设计模式

>设计模式(Design Pattern)是一套被反复使用、多数人知晓的、经过分类编目的、代码设计经验的总结，使用设计模式是为了可重用代码、让代码更容易被他人理解并且保证代码可靠性。

**设计模式一般包含模式名称、问题、目的、解决方案、效果等组成要素，其中关键要素是模式名称、问题、解决方案和效果**

***

设计模式分为三类：**5种创建型模式**、**7种结构型模式**、**11种行为型模式**

***

# 23种设计模式

**创建型模式：**单例模式、抽象工厂模式、建造者模式、工厂模式、原型模式。

**结构型模式：**适配器模式、桥接模式、装饰模式、组合模式、外观模式、享元模式、代理模式。

**行为型模式：**模版方法模式、命令模式、迭代器模式、观察者模式、中介者模式、备忘录模式、解释器模式、状态模式、策略模式、职责链模式(责任链模式)、访问者模式。

复习的时候，发现了一个很有趣的开源代码：https://github.com/youlookwhat/DesignPattern

***

# 设计模式的六大原则

* **1.开闭原则：**对扩展开放，对修改关闭；

* **2.里氏替换原则：**任何基类可以出现的地方，子类一定可以出现
LSP是继承复用的基石，只有当衍生类可以替换掉基类，软件单位的功能不受到影响时，基类才能真正被复用，而衍生类也能够在基类的基础上增加新的行为

* **依赖倒转原则：** 开闭原则的基础；针对接口变成，依赖于抽象而不依赖于具体；

* **接口隔离原则：**使用多个隔离的接口，比使用单个接口要好

* **最少知道原则：**一个实体应当尽量少的与其他实体之间发生相互作用，使得系统功能模块相对独立（高内聚低耦合）

* **合成复用原则：**尽量使用合成/聚合的方式，而不是使用继承

# 创建型模式

## 单例模式

单例模式，保证JVM中只有一个单例对象并且该对象只有一个实例存在；

**好处：**
1、某些类创建比较频繁，对于一些大型的对象，这是一笔很大的系统开销

2、省去了new操作符，降低了系统内存的使用频率，减轻GC压力

3、有些类如交易所的核心交易引擎，控制着交易流程，如果该类可以创建多个的话，系统完全乱了。（比如一个军队出现了多个司令员同时指挥，肯定会乱成一团），所以只有使用单例模式，才能保证核心交易服务器独立控制整个流程。

**坏处：**之前看过内存溢出相关的文章，单例模式的滥用似乎很容易发生内存溢出；

eg：(栗子来自于：http://blog.csdn.net/zhangerqing/article/details/8194653)

```java

public class Singleton{
	
	private static Singleton instance = null;

	//私有构造方法，防止被实例化
	private Singleton(){}


	//synchronized 用作线程保护

	public static Singleton getInstance(){
		synchronized (instance) {  
                if (instance == null) {  
                    instance = new Singleton();  
                }  
            }  
		return instance;
	}

	//如果对象被用于序列化，可以保证对象在序列化前后保持一致；
	public Object readResolce(){
		return instance;
	}
}

/*优化：*/

private static class SingletonFactory{           
        private static Singleton instance = new Singleton();           
    }           
    public static Singleton getInstance(){           
        return SingletonFactory.instance;           
    } 
} 


```

**可以利用内部类来维护单例的实现：**

```java

public class Singleton {  
  
    /* 私有构造方法，防止被实例化 */  
    private Singleton() {  
    }  
  
    /* 此处使用一个内部类来维护单例 */  
    private static class SingletonFactory {  
        private static Singleton instance = new Singleton();  
    }  
  
    /* 获取实例 */  
    public static Singleton getInstance() {  
        return SingletonFactory.instance;  
    }  
  
    /* 如果该对象被用于序列化，可以保证对象在序列化前后保持一致 */  
    public Object readResolve() {  
        return getInstance();  
    }  
}  

```



## 工厂模式

工厂模式分为：普通工厂模式，多个工厂模式，静态工厂模式

### 普通工厂模式

建立一个工厂类，对实现了同一接口的一些类进行实例的创建

eg:来自：http://blog.csdn.net/zhangerqing/article/details/8194653

```java

public interface Sender{
	public void Send();
}

//Class_1

public class MailSender implements Sender{
	@Override
	public void Send(){
		...
	}
}

//Class_2

public class SmsSender implements Sender{
	@Override
	public void Send(){
		...
	}
}

/ **
/ * Factory:
** /

public class SendFactory{
	
	public Sender produce(String type){
		if("mail".equals(type)){
			
			return new MailSender();
		
		}else if("sms".equals(type)){
			retrun new SmsSender();
		}else{
			...
			return null;
		}
	}
}

//Test

public class FactoryTest{
	public static void main(String[] args){
		SendFactory factory = new SendFactory();
		Sender sender = factory.produce("sms")';
		sender.Send();
	}
}
```

### 多个工厂方法模式
对普通工厂方法模式的费劲，在普通工厂方法模式中，如果传递的字符串出错，则不能创建对象，而多个工厂方法模式是提供多个工厂方法，分别创建对象

eg：将Factory修改为：

```java

public class SendFactory{
	public Sender produceMail(){
		return new MailSender();
	}

	publc Sender produceSms(){
		return new SmsSender();
	}
}

```

### 静态工厂方法模式：

将工厂类中的方法设为静态，不需要创建实例；

***

## 抽象工厂模式

普通的工厂方法，类的创建以来于工厂类；扩展程序的时候，需要对工厂类进行修改；

抽象工厂模式，创建多个工厂类，在增加新的功能的时候没直接增加新的工厂类，不需要修改之前的代码；

eg：利用刚才的两个类：

```java

//提供一个接口

public interface Provider {  
    public Sender produce();  
}  

//创建两个工厂类

public class SendMailFactory implements Provider {  
      
    @Override  
    public Sender produce(){  
        return new MailSender();  
    }  
}  

public class SendSmsFactory implements Provider{  
  
    @Override  
    public Sender produce() {  
        return new SmsSender();  
    }  
}

//Test

public class Test {  
  
    public static void main(String[] args) {  
        Provider provider = new SendMailFactory();  
        Sender sender = provider.produce();  
        sender.Send();  
    }  
}    

```


## 建造者模式（Builder）

>建造者模式则是将各种产品集中起来进行管理，用来创建复合对象，所谓复合对象就是指某个类具有不同的属性，其实建造者模式就是前面抽象工厂模式和最后的Test结合起来得到的

感觉安卓开发的过程中，好像很经常会用到Builder这个东西；比如说okhttp的FormBody的构造；

eg:

```java

public class Builder {  
      
    private List<Sender> list = new ArrayList<Sender>();  
      
    public void produceMailSender(int count){  
        for(int i=0; i<count; i++){  
            list.add(new MailSender());  
        }  
    }  
      
    public void produceSmsSender(int count){  
        for(int i=0; i<count; i++){  
            list.add(new SmsSender());  
        }  
    }  
}  

public class Test {  
  
    public static void main(String[] args) {  
        Builder builder = new Builder();  
        builder.produceMailSender(10);  
    }  
}  

```

***
还有另一种形式（在effective Java中看到的）

```java

public class NutritionFacts {  
    private final int servingSize;  
    private final int servings;  
    private final int calories;  
    private final int fat;  
    private final int sodium;  
    private final int carbohydrate;  
  
    public static class Builder {  
        // Required parameters  
        private final int servingSize;  
        private final int servings;  
  
        // Optional parameters - initialized to default values  
        private int calories      = 0;  
        private int fat           = 0;  
        private int carbohydrate  = 0;  
        private int sodium        = 0;  
  
        public Builder(int servingSize, int servings) {  
            this.servingSize = servingSize;  
            this.servings    = servings;  
        }  
  
        public Builder calories(int val)  
            { calories = val;      return this; }  
        public Builder fat(int val)  
            { fat = val;           return this; }  
        public Builder carbohydrate(int val)  
            { carbohydrate = val;  return this; }  
        public Builder sodium(int val)  
            { sodium = val;        return this; }  
  
        public NutritionFacts build() {  
            return new NutritionFacts(this);  
        }  
    }  
  
    private NutritionFacts(Builder builder) {  
        servingSize  = builder.servingSize;  
        servings     = builder.servings;  
        calories     = builder.calories;  
        fat          = builder.fat;  
        sodium       = builder.sodium;  
        carbohydrate = builder.carbohydrate;  
    }  
  
    public static void main(String[] args) {  
        NutritionFacts cocaCola = new NutritionFacts.Builder(240, 8).  
            calories(100).sodium(35).carbohydrate(27).build();  
    }  
}  

```

在类的内部定义一个Builder内部类，通过Builder中的函数，设置对象的各种值，最后build为我们的类对象；
感觉这个跟平时开发中用到的东西比较像；

**我觉得，利用Builder模式，就解决了多个构造函数的问题**

## 原型模式

该模式的思想就是将一个对象作为原型，对其进行复制、克隆，产生一个和原对象类似的新对象

eg：栗子来自：http://blog.csdn.net/zhangerqing/article/details/8194653

```java

/ **
/ * 原型类
/ **

public class Prototype implements Cloneable {  
  
    public Object clone() throws CloneNotSupportedException {  
        Prototype proto = (Prototype) super.clone();  
        return proto;  
    }  
}  

```

一个原型类，只需要实现Cloneable接口，覆写clone方法，此处clone方法可以改成任意的名称（Cloneable接口是个空接口，你可以任意定义实现类的方法名）

>Object类中，clone()是native的


>浅复制：将一个对象复制后，基本数据类型的变量都会重新创建，而引用类型，指向的还是原对象所指向的。
>深复制：将一个对象复制后，不论是基本数据类型还有引用类型，都是重新创建的。简单来说，就是深复制进行了完全彻底的复制，而浅复制不彻底。

```java

public class Prototype implements Cloneable, Serializable {  
  
    private static final long serialVersionUID = 1L;  
    private String string;  
  
    private SerializableObject obj;  
  
    /* 浅复制 */  
    public Object clone() throws CloneNotSupportedException {  
        Prototype proto = (Prototype) super.clone();  
        return proto;  
    }  
  
    /* 深复制 */  
    public Object deepClone() throws IOException, ClassNotFoundException {  
  
        /* 写入当前对象的二进制流 */  
        ByteArrayOutputStream bos = new ByteArrayOutputStream();  
        ObjectOutputStream oos = new ObjectOutputStream(bos);  
        oos.writeObject(this);  
  
        /* 读出二进制流产生的新对象 */  
        ByteArrayInputStream bis = new ByteArrayInputStream(bos.toByteArray());  
        ObjectInputStream ois = new ObjectInputStream(bis);  
        return ois.readObject();  
    }  
  
    public String getString() {  
        return string;  
    }  
  
    public void setString(String string) {  
        this.string = string;  
    }  
  
    public SerializableObject getObj() {  
        return obj;  
    }  
  
    public void setObj(SerializableObject obj) {  
        this.obj = obj;  
    }  
  
}  
  
class SerializableObject implements Serializable {  
    private static final long serialVersionUID = 1L;  
}  

```

***

# 结构型模式

## 适配器模式

将一个类的接口转换成客户期望的另一个接口，适配器让原本接口不兼容的类可以相互合作

eg:

```java

public class Mobile  
{  
    /** 
     * 充电 
     * @param power  
     */  
    public void inputPower(V5Power power)  
    {  
        int provideV5Power = power.provideV5Power();  
        System.out.println("手机（客户端）：我需要5V电压充电，现在是-->" + provideV5Power + "V");  
    }  
}  

public interface V5Power  
{  
    public int provideV5Power();  
}  

```



```java

public class V220Power  
{  
    /** 
     * 提供220V电压 
     * @return 
     */  
    public int provideV220Power()  
    {  
        System.out.println("我提供220V交流电压。");  
        return 220 ;   
    }  
}  

```

适配器：

```java

public class V5PowerAdapter implements V5Power  
{  
    /** 
     * 组合的方式 
     */  
    private V220Power v220Power ;  
      
    public V5PowerAdapter(V220Power v220Power)  
    {  
        this.v220Power = v220Power ;  
    }  
  
    @Override  
    public int provideV5Power()  
    {  
        int power = v220Power.provideV220Power() ;  
        //power经过各种操作-->5   
        System.out.println("适配器：我悄悄的适配了电压。");  
        return 5 ;   
    }   
      
}  

```

```java

/ **
/ * Test
/ **

public class Test  
{  
    public static void main(String[] args)  
    {  
        Mobile mobile = new Mobile();  
        V5Power v5Power = new V5PowerAdapter(new V220Power()) ;   
        mobile.inputPower(v5Power);  
    }  
}  

```

## 桥接模式

将抽象部分与它的实现部分分离，使它们都可以独立地变化。它是一种对象结构型模式，又称为柄体(Handle and Body)模式或接口(Interface)模式。

**主要特点是把抽象（abstraction）与行为实现（implementation）分离开来**

**Client**Bridge模式的使用者

**Abstraction**抽象类接口（接口或抽象类）；维护对行为实现（Implementor）的引用

**Refined Abstraction**Abstraction子类

**Implementor**行为实现类接口 (Abstraction接口定义了基于Implementor接口的更高层次的操作)

**ConcreteImplementor**Implementor子类

eg：栗子来自：http://blog.csdn.net/shaopeng5211/article/details/8827507

```java

/ **
/ * implemeentor
/ * 行为实现类接口 (Abstraction接口定义了基于Implementor接口的更高层次的操作)
/ **

public interface Engine{
	public void installEngine();
}

/ **
/ * ConcreteImplemeentor
/ * Implementor子类
/ **

public class Engine2000 implements Engine{
	@Override
	public void installEngine(){
		...
	}
}

public class Engine2200 implements Engine{
	@Override
	public void installEngine(){
		...
	}
}

/ **
/ * Abstraction
/ * 抽象类接口（接口或抽象类）；维护对行为实现（Implementor）的引用
/ **

public abstract class Car{
	private Engine engine;

	publc Car(Engine engine){
		this.engine = engine;
	}

	public Engine getEngine(){
		return engine;
	}

	public void setEngine(Engine engine){
		this.engine = engine;
	}

	public abstract void installEngine();
}

/ **
/ * Refined Abstraction
/ * Abstraction子类
/ **

public class Bus extends Car{
	public Bus(Engine engine){
		super(engine);
	}

	@Override
	public void installEngine(){
		...
		this.getEngine().installEngine();
	}
}

public class Jeep extends Car {  
  
    public Jeep(Engine engine) {  
        super(engine);  
    }  
    @Override  
    public void installEngine() {  
        System.out.print("Jeep:");  
        this.getEngine().installEngine();  
    }  
  
} 

/ **
/ * Client
/ * Bridge模式的使用者
/ **

public class MainClass{
	public static void main(String args[]){
		Engine engine2000 = new Engine2000();  
        Engine engine2200 = new Engine2200();  
          
        Car bus = new Bus(engine2000);  
        bus.installEngine();  
          
        Car jeep = new Jeep(engine2200);  
        jeep.installEngine();  
	}
}


```

感觉桥接模式，有点像……画画的时候，有各种笔，然后颜料也有各种颜色；然后不同的笔可以蘸不同的颜料；

## 装饰者模式

若要扩展功能，装饰者提供了比集成更有弹性的替代方案，动态地将责任附加到对象上

>当我们设计好了一个类，我们需要给这个类添加一些辅助的功能，并且不希望改变这个类的代码，这时候就是装饰者模式大展雄威的时候了

**体现了开闭原则**

eg：栗子来自：http://blog.csdn.net/lmj623565791/article/details/24269409
这个博主用了游戏系统中装备加宝石，提升属性当做栗子，感觉很有意思；

```java

/** 
 * 装备的接口 
 *  
 * @author zhy 
 *  
 */  
public interface IEquip  
{  
  
    /** 
     * 计算攻击力 
     *  
     * @return 
     */  
    public int caculateAttack();  
  
    /** 
     * 装备的描述 
     *  
     * @return 
     */  
    public String description();  
}  

/** 
 * 武器 
 * 攻击力20 
 * @author zhy 
 *  
 */  
public class ArmEquip implements IEquip  
{  
  
    @Override  
    public int caculateAttack()  
    {  
        return 20;  
    }  
  
    @Override  
    public String description()  
    {  
        return "屠龙刀";  
    }  
  
}  

/** 
 * 戒指 
 * 攻击力 5 
 * @author zhy 
 * 
 */  
public class RingEquip implements IEquip  
{  
  
    @Override  
    public int caculateAttack()  
    {  
        return 5;  
    }  
  
    @Override  
    public String description()  
    {  
        return "圣战戒指";  
    }  
  
}  

```

装饰器：

```java

/** 
 * 装饰品的接口 
 * @author zhy 
 * 
 */  
public interface IEquipDecorator extends IEquip  
{  
      
}  

/** 
 * 蓝宝石装饰品 
 * 每颗攻击力+5 
 * @author zhy 
 *  
 */  
public class BlueGemDecorator implements IEquipDecorator  
{  
    /** 
     * 每个装饰品维护一个装备 
     */  
    private IEquip equip;  
  
    public BlueGemDecorator(IEquip equip)  
    {  
        this.equip = equip;  
    }  
  
    @Override  
    public int caculateAttack()  
    {  
        return 5 + equip.caculateAttack();  
    }  
  
    @Override  
    public String description()  
    {  
        return equip.description() + "+ 蓝宝石";  
    }  
  
}  

/** 
 * 黄宝石装饰品 
 * 每颗攻击力+10 
 * @author zhy 
 *  
 */  
public class YellowGemDecorator implements IEquipDecorator  
{  
    /** 
     * 每个装饰品维护一个装备 
     */  
    private IEquip equip;  
  
    public YellowGemDecorator(IEquip equip)  
    {  
        this.equip = equip;  
    }  
  
    @Override  
    public int caculateAttack()  
    {  
        return 10 + equip.caculateAttack();  
    }  
  
    @Override  
    public String description()  
    {  
        return equip.description() + "+ 黄宝石";  
    }  
  
}  

/** 
 * 红宝石装饰品 每颗攻击力+15 
 *  
 * @author zhy 
 *  
 */  
public class RedGemDecorator implements IEquipDecorator  
{  
    /** 
     * 每个装饰品维护一个装备 
     */  
    private IEquip equip;  
  
    public RedGemDecorator(IEquip equip)  
    {  
        this.equip = equip;  
    }  
  
    @Override  
    public int caculateAttack()  
    {  
        return 15 + equip.caculateAttack();  
    }  
  
    @Override  
    public String description()  
    {  
        return equip.description() + "+ 红宝石";  
    }  
  
}  

//Test!

package com.zhy.pattern.decorator;  
  
public class Test  
{  
    public static void main(String[] args)  
    {  
        // 一个镶嵌1颗红宝石，1颗蓝宝石的武器  
        System.out.println(" 一个镶嵌1颗红宝石，1颗蓝宝石,1颗黄宝石的武器");  
        equip = new RedGemDecorator(new BlueGemDecorator(new YellowGemDecorator(new ArmEquip())));  
        System.out.println("攻击力  : " + equip.caculateAttack());  
        System.out.println("描述 :" + equip.description());  
        System.out.println("-------");  
    }  
}  

```

## 组合模式

将对象组合成树形结构以表示“部分-整体”的层次结构。Composite使得用户对单个对象和组合对象的使用具有一致性。


## 外观模式

提供一个统一的接口，用来访问子系统中的一群接口，外观定义了一个高层的接口，让子系统更容易使用

eg:栗子来自：http://blog.csdn.net/lmj623565791/article/details/25837275

```java

public class HomeTheaterFacade  
{  
    private Computer computer;  
    private Player player;  
    private Light light;  
    private Projector projector;  
    private PopcornPopper popper;  
  
    public HomeTheaterFacade(Computer computer, Player player, Light light, Projector projector, PopcornPopper popper)  
    {  
        this.computer = computer;  
        this.player = player;  
        this.light = light;  
        this.projector = projector;  
        this.popper = popper;  
    }  
  
    public void watchMovie()  
    {  
        /** 
         *  1、打开爆米花机 
            2、制作爆米花 
            3、将灯光调暗 
            4、打开投影仪 
            5、放下投影仪投影区 
            6、打开电脑 
            7、打开播放器 
            8、将播放器音调设为环绕立体声 
         */  
        popper.on();  
        popper.makePopcorn();  
        light.down();  
        projector.on();  
        projector.open();  
        computer.on();  
        player.on();  
        player.make3DListener();  
    }  
      
    public void stopMovie()  
    {  
        popper.off();  
        popper.stopMakePopcorn();  
        light.up();  
        projector.close();  
        projector.off();  
        player.off();  
        computer.off();  
    }  
}  

```

感觉像是，将要做一件事的所有步骤，整合到一块；


## 享元模式

享元模式采用一个共享来避免大量拥有相同内容对象的开销；享元模式以共享的方式高效地支持大量的细粒度对象

像String型，就是享元模式的；

## 代理模式

为其他对象提供一种代理以控制对这个对象的访问

>在某些情况下，一个客户不想或者不能直接引用另一个对象，而代理对象可以在客户端和目标对象之间起到中介的作用


**抽象角色：**声明真实对象和代理对象的共同接口；

**代理角色：**代理对象角色内部含有对真实对象的引用，从而可以操作真实对象，同时代理对象提供与真实对象相同的接口以便在任何时刻都能代替真实对象。同时，代理对象可以在执行真实对象操作时，附加其他的操作，相当于对真实对象进行封装。

**真实角色：**代理角色所代表的真实对象，是我们最终要引用的对象。

### 静态代理

eg：栗子来自《Java与模式》

```java

/** 
 * 抽象角色 
 */  
public abstract class Subject {  
    public abstract void request();  
}  

/** 
 * 真实的角色 
 */  
public class RealSubject extends Subject {  
  
    @Override  
    public void request() {  
        // TODO Auto-generated method stub  
  
    }  
  
}  

/** 
 * 静态代理，对具体真实对象直接引用 
 * 代理角色，代理角色需要有对真实角色的引用， 
 * 代理做真实角色想做的事情 
 */  
public class ProxySubject extends Subject {  
      
    private RealSubject realSubject = null;  
      
    /** 
     * 除了代理真实角色做该做的事情，代理角色也可以提供附加操作， 
     * 如：preRequest()和postRequest() 
     */  
    @Override  
    public void request() {  
        preRequest();  //真实角色操作前的附加操作  
          
        if(realSubject == null){  
            realSubject =  new RealSubject();  
        }  
        realSubject.request();  
          
        postRequest();  //真实角色操作后的附加操作  
    }  
  
    /** 
     *  真实角色操作前的附加操作 
     */  
    private void postRequest() {  
        // TODO Auto-generated method stub  
          
    }  
  
    /** 
     *  真实角色操作后的附加操作 
     */  
    private void preRequest() {  
        // TODO Auto-generated method stub  
          
    }  
}  

/** 
 *  客户端调用  
 */  
public class Main {  
    public static void main(String[] args) {  
        Subject subject = new ProxySubject();  
        subject.request();  //代理者代替真实者做事情  
    }  
}  

```

### 动态代理

Java动态代理类位于Java.lang.reflect包下

ge：

```java

package dynamicproxy;  
  
/** 
 * 抽象角色 
 * 这里应改为接口 
 */  
public interface Subject {  
    void request();  
}  

  
/** 
 * 真实的角色 
 * 实现接口 
 */  
public class RealSubject implements Subject {  
  
    @Override  
    public void request() {  
        // TODO Auto-generated method stub  
  
    }  
  
}  

import java.lang.reflect.InvocationHandler;  
import java.lang.reflect.Method;  
  
/** 
 * 动态代理， 它是在运行时生成的class，在生成它时你必须提供一组interface给它， 然后该class就宣称它实现了这些interface。 
 * 你当然可以把该class的实例当作这些interface中的任何一个来用。 当然啦，这个Dynamic 
 * Proxy其实就是一个Proxy，它不会替你作实质性的工作， 在生成它的实例时你必须提供一个handler，由它接管实际的工作。 
 */  
public class DynamicSubject implements InvocationHandler {  
  
    private Object sub; // 真实对象的引用  
  
    public DynamicSubject(Object sub) {  
        this.sub = sub;  
    }  
  
    @Override  
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {  
        System.out.println("before calling " + method);   
        method.invoke(sub,args);   
        System.out.println("after calling " + method);   
        return null;   
    }  
  
}  

import java.lang.reflect.Constructor;  
import java.lang.reflect.InvocationHandler;  
import java.lang.reflect.Proxy;  
  
public class Main {  
    public static void main(String[] args) throws Throwable {  
        RealSubject rs = new RealSubject();  
        InvocationHandler handler = new DynamicSubject(rs);  
        Class cls = rs.getClass();  
        //以下是分解步骤  
        /* 
        Class c = Proxy.getProxyClass(cls.getClassLoader(), cls.getInterfaces()); 
        Constructor ct = c.getConstructor(new Class[]{InvocationHandler.class}); 
        Subject subject =(Subject) ct.newInstance(new Object[]{handler}); 
        */  
          
        //以下是一次性生成  
        Subject subject = (Subject)Proxy.newProxyInstance(cls.getClassLoader(),cls.getInterfaces(), handler);  
        subject.request();  
    }  
}  

```

***

# 行为型模式

## 模版方法模式

定义了一个算法的骨架，而将一些步骤延迟到子类中，模版方法使得子类可以在不改变算法结构的情况下，重新定义算法的步骤

模版方法定义了一个算法的步骤，并且允许子类为一个或多个步骤提供实现


eg:栗子来自：http://blog.csdn.net/lmj623565791/article/details/26276093
博主的WorkDay栗子；

```java

public abstract class Worker  
{  
    protected String name;  
  
    public Worker(String name)  
    {  
        this.name = name;  
    }  
  
    /** 
     * 记录一天的工作 
     */  
    public final void workOneDay()  
    {  
  
        System.out.println("-----------------work start ---------------");  
        enterCompany();  
        computerOn();  
        work();  
        computerOff();  
        exitCompany();  
        System.out.println("-----------------work end ---------------");  
  
    }  
  
    /** 
     * 工作 
     */  
    public abstract void work();  
  
    /** 
     * 关闭电脑 
     */  
    private void computerOff()  
    {  
        System.out.println(name + "关闭电脑");  
    }  
  
    /** 
     * 打开电脑 
     */  
    private void computerOn()  
    {  
        System.out.println(name + "打开电脑");  
    }  
  
    /** 
     * 进入公司 
     */  
    public void enterCompany()  
    {  
        System.out.println(name + "进入公司");  
    }  
  
    /** 
     * 离开公司 
     */  
    public void exitCompany()  
    {  
        System.out.println(name + "离开公司");  
    }  
  
}  

/ *
* codeMokey
/ * 
public class ITWorker extends Worker  
{  
  
    public ITWorker(String name)  
    {  
        super(name);  
    }  
  
    @Override  
    public void work()  
    {  
        System.out.println(name + "写程序-测bug-fix bug");  
    }  
  
}  

public class HRWorker extends Worker  
{  
  
    public HRWorker(String name)  
    {  
        super(name);  
    }  
  
    @Override  
    public void work()  
    {  
        System.out.println(name + "看简历-打电话-接电话");  
    }  
  
}  


public class Test  
{  
    public static void main(String[] args)  
    {  
  
        Worker it1 = new ITWorker("鸿洋");  
        it1.workOneDay();  
        Worker it2 = new ITWorker("老张");  
        it2.workOneDay();  
        Worker hr = new HRWorker("迪迪");  
        hr.workOneDay();  
        Worker qa = new QAWorker("老李");  
        qa.workOneDay();  
        Worker pm = new ManagerWorker("坑货");  
        pm.workOneDay();  
  
    }  
}  

```


## 命令模式

将“请求”封装成对象，以便使用不同的请求、队列或者日志来参数化其他对象

>将请求封装成对象，将动作请求者和动作执行者解耦

eg:栗子来自：http://blog.csdn.net/lmj623565791/article/details/24602057

```java

/** 
 * 门 
 * @author zhy 
 * 
 */  
public class Door  
{  
    public void open()  
    {  
        System.out.println("打开门");  
    }  
  
    public void close()  
    {  
        System.out.println("关闭门");  
    }  
  
}  

/** 
 * 电灯 
 * @author zhy 
 * 
 */  
public class Light  
{  
    public void on()  
    {  
        System.out.println("打开电灯");  
    }  
  
    public void off()  
    {  
        System.out.println("关闭电灯");  
    }  
}  

/** 
 * 电脑 
 * @author zhy 
 * 
 */  
public class Computer  
{  
    public void on()  
    {  
        System.out.println("打开电脑");  
    }  
      
    public void off()  
    {  
        System.out.println("关闭电脑");  
    }  
}  

/**
 * 封装命令
 *
 */

public interface Command  
{  
    public void execute();  
}  

/** 
 * 关闭电灯的命令 
 * @author zhy 
 * 
 */  
public class LightOffCommond implements Command  
{  
    private Light light ;   
      
    public LightOffCommond(Light light)  
    {  
        this.light = light;  
    }  
  
    @Override  
    public void execute()  
    {  
        light.off();  
    }  
  
}  

/** 
 * 打开电灯的命令 
 * @author zhy 
 * 
 */  
public class LightOnCommond implements Command  
{  
    private Light light ;   
      
    public LightOnCommond(Light light)  
    {  
        this.light = light;  
    }  
  
    @Override  
    public void execute()  
    {  
        light.on();  
    }  
  
}  


/** 
 * 开电脑的命令 
 * @author zhy 
 * 
 */  
public class ComputerOnCommond implements Command  
{  
    private Computer computer ;   
      
    public ComputerOnCommond( Computer computer)  
    {  
        this.computer = computer;  
    }  
  
    @Override  
    public void execute()  
    {  
        computer.on();  
    }  
  
}  

/** 
 * 关电脑的命令 
 * @author zhy 
 * 
 */  
public class ComputerOffCommond implements Command  
{  
    private Computer computer ;   
      
    public ComputerOffCommond( Computer computer)  
    {  
        this.computer = computer;  
    }  
  
    @Override  
    public void execute()  
    {  
        computer.off();  
    }  
      
      
  
}  

/** 
 * 控制器面板，一共有9个按钮 
 *  
 * @author zhy 
 *  
 */  
public class ControlPanel  
{  
    private static final int CONTROL_SIZE = 9;  
    private Command[] commands;  
  
    public ControlPanel()  
    {  
        commands = new Command[CONTROL_SIZE];  
        /** 
         * 初始化所有按钮指向空对象 
         */  
        for (int i = 0; i < CONTROL_SIZE; i++)  
        {  
            commands[i] = new NoCommand();  
        }  
    }  
  
    /** 
     * 设置每个按钮对应的命令 
     * @param index 
     * @param command 
     */  
    public void setCommand(int index, Command command)  
    {  
        commands[index] = command;  
    }  
  
    /** 
     * 模拟点击按钮 
     * @param index 
     */  
    public void keyPressed(int index)  
    {  
        commands[index].execute();  
    }  
  
}  


public class NoCommand implements Command  
{  
    @Override  
    public void execute()  
    {  
  
    }  
  
}  


public class Test  
{  
    public static void main(String[] args)  
    {  
        /** 
         * 三个家电 
         */  
        Light light = new Light();  
        Door door = new Door();  
        Computer computer = new Computer();  
        /** 
         * 一个控制器，假设是我们的app主界面 
         */  
        ControlPanel controlPanel = new ControlPanel();  
        // 为每个按钮设置功能  
        controlPanel.setCommand(0, new LightOnCommond(light));  
        controlPanel.setCommand(1, new LightOffCommond(light));  
        controlPanel.setCommand(2, new ComputerOnCommond(computer));  
        controlPanel.setCommand(3, new ComputerOffCommond(computer));  
        controlPanel.setCommand(4, new DoorOnCommond(door));  
        controlPanel.setCommand(5, new DoorOffCommond(door));  
  
        // 模拟点击  
        controlPanel.keyPressed(0);  
        controlPanel.keyPressed(2);  
        controlPanel.keyPressed(3);  
        controlPanel.keyPressed(4);  
        controlPanel.keyPressed(5);  
        controlPanel.keyPressed(8);// 这个没有指定，但是不会出任何问题，我们的NoCommand的功劳  
  
          
  
    }  
}  

```

## 迭代器模式

提供一种方法访问一个容器对象中各个元素，而又不暴露该对象的内部细节

* **结构：**

**抽象容器：**一般是一个接口，提供一个iterator()方法，例如java中的Collection接口，List接口，Set接口等。

**具体容器：**就是抽象容器的具体实现类，比如List接口的有序列表实现ArrayList，List接口的链表实现LinkList，Set接口的哈希列表的实现HashSet等。

**抽象迭代器：**定义遍历元素所需要的方法，一般来说会有这么三个方法：取得第一个元素的方法first()，取得下一个元素的方法next()，判断是否遍历结束的方法isDone()（或者叫hasNext()），移出当前对象的方法remove();

**迭代器实现：**实现迭代器接口中定义的方法，完成集合的迭代。

eg:栗子来自：http://blog.csdn.net/zhengzhb/article/details/7610745

```java

interface Iterator {  
    public Object next();  
    public boolean hasNext();  
}  
class ConcreteIterator implements Iterator{  
    private List list = new ArrayList();  
    private int cursor =0;  
    public ConcreteIterator(List list){  
        this.list = list;  
    }  
    public boolean hasNext() {  
        if(cursor==list.size()){  
            return false;  
        }  
        return true;  
    }  
    public Object next() {  
        Object obj = null;  
        if(this.hasNext()){  
            obj = this.list.get(cursor++);  
        }  
        return obj;  
    }  
}  
interface Aggregate {  
    public void add(Object obj);  
    public void remove(Object obj);  
    public Iterator iterator();  
}  
class ConcreteAggregate implements Aggregate {  
    private List list = new ArrayList();  
    public void add(Object obj) {  
        list.add(obj);  
    }  
  
    public Iterator iterator() {  
        return new ConcreteIterator(list);  
    }  
  
    public void remove(Object obj) {  
        list.remove(obj);  
    }  
}  
public class Client {  
    public static void main(String[] args){  
        Aggregate ag = new ConcreteAggregate();  
        ag.add("小明");  
        ag.add("小红");  
        ag.add("小刚");  
        Iterator it = ag.iterator();  
        while(it.hasNext()){  
            String str = (String)it.next();  
            System.out.println(str);  
        }  
    }  
}  

```

## 观察者模式

定义了对象之间的一对多的依赖，这样一来，当一个对象改变时，它的所有的依赖者都会收到通知并自动更新

eg:栗子来自：http://blog.csdn.net/lmj623565791/article/details/24179699

```java

/** 
 * 主题接口，所有的主题必须实现此接口 
 *  
 * @author zhy 
 *  
 */  
public interface Subject  
{  
    /** 
     * 注册一个观察着 
     *  
     * @param observer 
     */  
    public void registerObserver(Observer observer);  
  
    /** 
     * 移除一个观察者 
     *  
     * @param observer 
     */  
    public void removeObserver(Observer observer);  
  
    /** 
     * 通知所有的观察着 
     */  
    public void notifyObservers();  
  
}  

/** 
 * @author zhy 所有的观察者需要实现此接口 
 */  
public interface Observer  
{  
    public void update(String msg);  
  
}  

import java.util.ArrayList;  
import java.util.List;  
  
public class ObjectFor3D implements Subject  
{  
    private List<Observer> observers = new ArrayList<Observer>();  
    /** 
     * 3D彩票的号码 
     */  
    private String msg;  
  
    @Override  
    public void registerObserver(Observer observer)  
    {  
        observers.add(observer);  
    }  
  
    @Override  
    public void removeObserver(Observer observer)  
    {  
        int index = observers.indexOf(observer);  
        if (index >= 0)  
        {  
            observers.remove(index);  
        }  
    }  
  
    @Override  
    public void notifyObservers()  
    {  
        for (Observer observer : observers)  
        {  
            observer.update(msg);  
        }  
    }  
  
    /** 
     * 主题更新消息 
     *  
     * @param msg 
     */  
    public void setMsg(String msg)  
    {  
        this.msg = msg;  
          
        notifyObservers();  
    }  
  
}  

//User：

public class Observer1 implements Observer  
{  
  
    private Subject subject;  
  
    public Observer1(Subject subject)  
    {  
        this.subject = subject;  
        subject.registerObserver(this);  
    }  
  
    @Override  
    public void update(String msg)  
    {  
        System.out.println("observer1 得到 3D 号码  -->" + msg + ", 我要记下来。");  
    }  
  
}  

public class Test  
{  
    public static void main(String[] args)  
    {  
        //模拟一个3D的服务号  
        ObjectFor3D subjectFor3d = new ObjectFor3D();  
        //客户1  
        Observer observer1 = new Observer1(subjectFor3d);  
        Observer observer2 = new Observer2(subjectFor3d);  
  
        subjectFor3d.setMsg("20140420的3D号码是：127" );  
        subjectFor3d.setMsg("20140421的3D号码是：333" );  
          
    }  
}  

```

## 中介者模式

用一个中介者对象封装一系列的对象交互，中介者使各对象不需要显示地相互作用，从而使耦合松散，而且可以独立地改变它们之间的交互

* **角色：**

**抽象中介者：**定义好同事类对象到中介者对象的接口，用于各个同事类之间的通信。一般包括一个或几个抽象的事件方法，并由子类去实现。

**中介者实现类：**从抽象中介者继承而来，实现抽象中介者中定义的事件方法。从一个同事类接收消息，然后通过消息影响其他同时类。

**同事类：**如果一个对象会影响其他的对象，同时也会被其他对象影响，那么这两个对象称为同事类。在类图中，同事类只有一个，这其实是现实的省略，在实际应用中，同事类一般由多个组成，他们之间相互影响，相互依赖。同事类越多，关系越复杂。并且，同事类也可以表现为继承了同一个抽象类的一组实现组成。在中介者模式中，同事类之间必须通过中介者才能进行消息传递。

>一般来说，同事类之间的关系是比较复杂的，多个同事类之间互相关联时，他们之间的关系会呈现为复杂的网状结构，这是一种过度耦合的架构，即不利于类的复用，也不稳定。例如有六个同事类对象，假如对象1发生变化，会有4个对象受到影响。如果对象2发生变化，那么会有5个对象受到影响。也就是说，同事类之间直接关联的设计是不好的。

>如果引入中介者模式，那么同事类之间的关系将变为星型结构，任何一个类的变动，只会影响的类本身，以及中介者，这样就减小了系统的耦合。一个好的设计，必定不会把所有的对象关系处理逻辑封装在本类中，而是使用一个专门的类来管理那些不属于自己的行为。

eg:栗子来自：http://blog.csdn.net/jason0539/article/details/45216585

```java

bstract class AbstractColleague {    
    protected int number;    
    
    public int getNumber() {    
        return number;    
    }    
    
    public void setNumber(int number){    
        this.number = number;    
    }    
    //抽象方法，修改数字时同时修改关联对象    
    public abstract void setNumber(int number, AbstractColleague coll);    
}    


class ColleagueA extends AbstractColleague{    
    public void setNumber(int number, AbstractColleague coll) {    
        this.number = number;    
        coll.setNumber(number*100);    
    }    
}    

class ColleagueB extends AbstractColleague{    
        
    public void setNumber(int number, AbstractColleague coll) {    
        this.number = number;    
        coll.setNumber(number/100);    
    }    
}   

//*********************************************************************

abstract class AbstractMediator {    
    protected AbstractColleague A;    
    protected AbstractColleague B;    
        
    public AbstractMediator(AbstractColleague a, AbstractColleague b) {    
        A = a;    
        B = b;    
    }    
    
    public abstract void AaffectB();    
        
    public abstract void BaffectA();    
    
}   

class Mediator extends AbstractMediator {    
    
    public Mediator(AbstractColleague a, AbstractColleague b) {    
        super(a, b);    
    }    
    
    //处理A对B的影响    
    public void AaffectB() {    
        int number = A.getNumber();    
        B.setNumber(number*100);    
    }    
    
    //处理B对A的影响    
    public void BaffectA() {    
        int number = B.getNumber();    
        A.setNumber(number/100);    
    }    
}    


//**********************************************************************

public class Client {    
    public static void main(String[] args){    
    
        AbstractColleague collA = new ColleagueA();    
        AbstractColleague collB = new ColleagueB();    
            
        System.out.println("==========设置A影响B==========");    
        collA.setNumber(1288, collB);    
        System.out.println("collA的number值："+collA.getNumber());    
        System.out.println("collB的number值："+collB.getNumber());    
    
        System.out.println("==========设置B影响A==========");    
        collB.setNumber(87635, collA);    
        System.out.println("collB的number值："+collB.getNumber());    
        System.out.println("collA的number值："+collA.getNumber());    
    }    
}    

```

中介模式能够避免类与类之间的过度耦合；
它将类与类之间的关系转换为星型结构；


## 备忘录模式

>备忘录对象是一个用来存储另外一个对象内部状态的快照的对象。备忘录模式的用意是在不破坏封装的条件下，将一个对象的状态捕捉(Capture)住，并外部化，存储起来，从而可以在将来合适的时候把这个对象还原到存储起来的状态。备忘录模式常常与命令模式和迭代子模式一同使用。

http://www.cnblogs.com/java-my-life/archive/2012/06/06/2534942.html

## 解释器模式

>给定一个语言之后，解释器模式可以定义出其文法的一种表示，并同时提供一个解释器。客户端可以使用这个解释器来解释这个语言中的句子。

http://www.cnblogs.com/java-my-life/archive/2012/06/19/2552617.html

## 状态模式

允许对象在内部状态改变时改变它的行为，对象看起来好像修改了它的类

>当对象的内部状态改变时，它的行为跟随状态的改变而改变了，看起来好像重新初始化了一个类似的。

eg：栗子来自：http://blog.csdn.net/lmj623565791/article/details/26350617

```java

/** 
 * 自动售货机 
 *  
 * @author zhy 
 *  
 */  
public class VendingMachine  
{  
  
    /** 
     * 已投币 
     */  
    private final static int HAS_MONEY = 0;  
    /** 
     * 未投币 
     */  
    private final static int NO_MONEY = 1;  
    /** 
     * 售出商品 
     */  
    private final static int SOLD = 2;  
    /** 
     * 商品售罄 
     */  
    private final static int SOLD_OUT = 3;  
  
    private int currentStatus = NO_MONEY;  
    /** 
     * 商品数量 
     */  
    private int count = 0;  
  
    public VendingMachine(int count)  
    {  
        this.count = count;  
        if (count > 0)  
        {  
            currentStatus = NO_MONEY;  
        }  
    }  
  
    /** 
     * 投入硬币，任何状态用户都可能投币 
     */  
    public void insertMoney()  
    {  
        switch (currentStatus)  
        {  
        case NO_MONEY:  
            currentStatus = HAS_MONEY;  
            System.out.println("成功投入硬币");  
            break;  
        case HAS_MONEY:  
            System.out.println("已经有硬币，无需投币");  
            break;  
        case SOLD:  
            System.out.println("请稍等...");  
            break;  
        case SOLD_OUT:  
            System.out.println("商品已经售罄，请勿投币");  
            break;  
  
        }  
    }  
  
    /** 
     * 退币，任何状态用户都可能退币 
     */  
    public void backMoney()  
    {  
        switch (currentStatus)  
        {  
        case NO_MONEY:  
            System.out.println("您未投入硬币");  
            break;  
        case HAS_MONEY:  
            currentStatus = NO_MONEY;  
            System.out.println("退币成功");  
            break;  
        case SOLD:  
            System.out.println("您已经买了糖果...");  
            break;  
        case SOLD_OUT:  
            System.out.println("您未投币...");  
            break;  
        }  
    }  
  
    /** 
     * 转动手柄购买,任何状态用户都可能转动手柄 
     */  
    public void turnCrank()  
    {  
        switch (currentStatus)  
        {  
        case NO_MONEY:  
            System.out.println("请先投入硬币");  
            break;  
        case HAS_MONEY:  
            System.out.println("正在出商品....");  
            currentStatus = SOLD;  
            dispense();  
            break;  
        case SOLD:  
            System.out.println("连续转动也没用...");  
            break;  
        case SOLD_OUT:  
            System.out.println("商品已经售罄");  
            break;  
  
        }  
    }  
  
    /** 
     * 发放商品 
     */  
    private void dispense()  
    {  
  
        switch (currentStatus)  
        {  
        case NO_MONEY:  
        case HAS_MONEY:  
        case SOLD_OUT:  
            throw new IllegalStateException("非法的状态...");  
        case SOLD:  
            count--;  
            System.out.println("发出商品...");  
            if (count == 0)  
            {  
                System.out.println("商品售罄");  
                currentStatus = SOLD_OUT;  
            } else  
            {  
                currentStatus = NO_MONEY;  
            }  
            break;  
  
        }  
  
    }  
}  

```


## 策略模式

定义了算法族，分别封装起来，让它们之间可相互替换，此模式让算法的变化独立于使用算法的客户

（代码好长啊……懒得贴了）
传送门：http://blog.csdn.net/lmj623565791/article/details/24116745

## 职责链模式(责任链模式)

责任链模式，我记得Android的事件分发机制就是责任链模式的；

之前在学习分发机制的时候有过一些记录：[Android自定义View——事件分发机制学习笔记](https://csming1995.github.io/2017/03/15/Android%E8%87%AA%E5%AE%9A%E4%B9%89View-12/)

>责任链模式是一种对象的行为模式。在责任链模式里，很多对象由每一个对象对其下家的引用而连接起来形成一条链。请求在这个链上传递，直到链上的某一个对象决定处理此请求。发出这个请求的客户端并不知道链上的哪一个对象最终处理这个请求，这使得系统可以在不影响客户端的情况下动态地重新组织和分配责任。

有一种比较简单的解释大概是酱紫的：领导给小组长安排工作，然后小组长再把工作分配给组员；如果组员做的了就揽下来，否则反馈给组长；若组长做的了就揽下来；

* **一种实现：**

![参照图](https://csming1995.github.io/attach/JavaPreview\zrl1.png)

* **角色：**

**抽象处理者(Handler)角色：**定义出一个处理请求的接口。如果需要，接口可以定义 出一个方法以设定和返回对下家的引用。这个角色通常由一个Java抽象类或者Java接口实现。上图中Handler类的聚合关系给出了具体子类对下家的引用，抽象方法handleRequest()规范了子类处理请求的操作。
**具体处理者(ConcreteHandler)角色：**具体处理者接到请求后，可以选择将请求处理掉，或者将请求传给下家。由于具体处理者持有对下家的引用，因此，如果需要，具体处理者可以访问下家。

eg：栗子来自：http://www.cnblogs.com/java-my-life/archive/2012/05/28/2516865.html
《Java与模式》

1.抽象处理者：

```java

public abstract class Handler {
    
    /**
     * 持有后继的责任对象
     */
    protected Handler successor;
    /**
     * 示意处理请求的方法，虽然这个示意方法是没有传入参数的
     * 但实际是可以传入参数的，根据具体需要来选择是否传递参数
     */
    public abstract void handleRequest();
    /**
     * 取值方法
     */
    public Handler getSuccessor() {
        return successor;
    }
    /**
     * 赋值方法，设置后继的责任对象
     */
    public void setSuccessor(Handler successor) {
        this.successor = successor;
    }
    
}

```

2.具体处理者

```java

public class ConcreteHandler extends Handler {
    /**
     * 处理方法，调用此方法处理请求
     */
    @Override
    public void handleRequest() {
        /**
         * 判断是否有后继的责任对象
         * 如果有，就转发请求给后继的责任对象
         * 如果没有，则处理请求
         */
        if(getSuccessor() != null)
        {            
            System.out.println("放过请求");
            getSuccessor().handleRequest();            
        }else
        {            
            System.out.println("处理请求");
        }
    }
 
}

```

3.Test:

```java

public class Client {

    public static void main(String[] args) {
        //组装责任链
        Handler handler1 = new ConcreteHandler();
        Handler handler2 = new ConcreteHandler();
        handler1.setSuccessor(handler2);
        //提交请求
        handler1.handleRequest();
    }
}

```

深入学习：http://www.cnblogs.com/java-my-life/archive/2012/05/28/2516865.html

## 访问者模式


***