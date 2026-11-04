---
publish: true
title: 「图解设计模式」学习笔记
created: 2022-03-21 15:43:42
modified: 2026-11-04T20:38:01.974Z
---

![](https://github.com/SoaringhawkCheng/blog/blob/master/source/_posts/illustration-of-design-pattern/mind.png?raw=true)

## 零、前言

统一建模语言(UML, Unified Modeling Language)，具体见[「Head first设计模式」学习笔记](https://blog.soaringhawkcheng.tech/%E5%B7%A5%E7%A8%8B/head-first-design-pattern/)

## 一、适应设计模式

### 1.1 迭代器模式 Iterator

### 1.1.1 组成部分

- iterator：定义遍历元素的接口
- concrete iterator：实现遍历元素的接口，并持有对应的concrete aggregator，保存当前的下标
- aggregate：定义创建iterator的接口
- concrete aggregate：实现aggregate的忌口

### 1.1.2 优缺点

- 优点：将遍历和容器实现分离

### 1.2 适配器模式 Adapter

类适配器模式：使用继承和接口实现

对象适配器模式：使用委托

对扩展开放，对修改封闭，符合开闭原则

## 二、交给子类

### 2.1 模板方法模式 Template Method

父类定义处理流程的框架，子类实现具体处理

- abstract class：负责实现模板方法，并声明模板方法中用到的抽象方法
- concrete class：实现abstract class声明的抽象方法

### 2.2 工厂方法模式 Factory Method

#### 2.2.1 简单工厂简介

简单工厂模式不属于23种GOF设计模式，是由一个工厂类根据传入的参数，动态决定应该创建哪一个产品类

- 优点：实现了对象创建和使用的分离，不修改客户端代码即可更改具体产品
- 缺点：更改具体产品需要修改工厂类，违反了开放——封闭原则

#### 2.2.2 工厂方法

工厂方法将工厂抽象化，增加新产品只需增加对应的具体实现工厂类
factory定义工厂接口，product定义产品接口，concrete product和对应的concrete factory实现接口

- 优点：解决了简单工厂不符合开放——封闭原则的问题
- 缺点：增加新产品需要增加工厂类

## 三、生成实例

### 3.1 单例模式 Singleton

[Java中的单例模式](https://zhuanlan.zhihu.com/p/322477346)

#### 3.1.1 饿汉模式

在类加载时就会进行单例的初始化，以后访问时直接使用单例对象即可

```
public class Singleton {
    private static Singleton instance = new Singleton();    
    // 获取实例（单例对象）
    public static Singleton getInstance() {
        return instance;
    }
    private Singleton() {
    }
}
```

#### 3.1.2 懒汉模式

当每次需要使用实例时，再去创建获取实例，而不是在类加载时就将实例创建好

```
public class Singleton {
    // 声明私有对象
    private static Singleton instance;
    // 获取实例（单例对象）
    public static Singleton getInstance() {
        if (instance == null) {
            instance = new Singleton();
        }
        return instance;
    }
    private Singleton() {
    }
}
```

#### 3.1.3 线程安全问题

为了保证懒汉模式的线程安全，简单的做法就是给获取实例的方法上加上 synchronized。但由于整个方法都被 synchronized 所包围，因此增加了同步开销，降低了程序的执行效率。

```
public class Singleton {
    // 声明私有对象
    private static Singleton instance;
    // 获取实例（单例对象）
    public synchronized static Singleton getInstance() {
        if (instance == null) {
            instance = new Singleton();
        }
        return instance;
    }
}
```

#### 3.1.4 优化执行效率

于是为了改进程序的执行效率，我们将 synchronized 放入到方法中，代码会存在着非线程安全的问题

```
public class Singleton {
    // 声明私有对象
    private static Singleton instance;
    // 获取实例（单例对象）
    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                instance = new Singleton();
            }
        }
        return instance;
    }
}
```

#### 3.1.5 双重检测锁

于是就诞生了大名鼎鼎的双重检测锁

```
public class Singleton {
    // 声明私有对象
    private static Singleton instance;
    // 获取实例（单例对象）
    public static Singleton getInstance() {
        // 第一次判断
        if (instance == null) {
            synchronized (Singleton.class) {
                // 第二次判断
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

#### 3.1.6 指令重排问题

但是instance = new Singleton() 这行代码看似是一个原子操作，然而并不是，这行代码最终会被编译成多条汇编指令，它大致的执行流程为以下三个步骤：

1. 给对象实例分配内存空间；
2. 调用对象的构造方法、初始化成员字段；
3. 将 instance 对象指向分配的内存空间

![](https://github.com/SoaringhawkCheng/blog/blob/master/source/_posts/illustration-of-design-pattern/rank.png?raw=true)

为了解决此问题，我们可以使用关键字 volatile 来修饰 instance 对象，这样就可以防止 CPU 指令重排，从而完美地运行懒汉模式

```
public class Singleton {
    // 声明私有对象
    private volatile static Singleton instance;
    // 获取实例（单例对象）
    public static Singleton getInstance() {
        // 第一次判断
        if (instance == null) {
            synchronized (Singleton.class) {
                // 第二次判断
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
    private Singleton() {
    }
}
```

#### 3.1.7 静态内部类

JVM 会在类初始化阶段（也就是类装载阶段）创建一个锁，该锁可以保证多个线程同步执行类初始化的工作，因此在多线程环境下，类加载机制依然是线程安全的。

静态内部类和饿汉方式也有着细微的差别，饿汉方式是在程序启动时就会进行加载，因此可能造成资源的浪费；而静态内部类只有在调用 getInstance() 方法时，才会装载内部类从而完成实例的初始化工作

```
public class Singleton {
    // 静态内部类
    private static class SingletonInstance {
        private static final Singleton instance = new Singleton();
    }
    // 获取实例（单例对象）
    public static Singleton getInstance() {
        return SingletonInstance.instance;
    }
    private Singleton() {
    }
}
```

#### 3.1.8 枚举类

枚举的实现方式不仅是线程安全的，而且只会装载一次，无论是序列化、反序列化、反射还是克隆都不会新创建对象

```
public class Singleton {
    // 枚举类型是线程安全的，并且只会装载一次
    private enum SingletonEnum {
        INSTANCE;
        // 声明单例对象
        private final Singleton instance;
        // 实例化
        SingletonEnum() {
            instance = new Singleton();
        }
        private Singleton getInstance() {
            return instance;
        }
    }
    // 获取实例（单例对象）
    public static Singleton getInstance() {
        return SingletonEnum.INSTANCE.getInstance();
    }
    private Singleton() {
    }
}
```

### 3.2 原型模式 Prototype

- prototype：负责定义用于复制现有实例生成新实例的方法
- concrete prototype：实现prototype的方法
- client：使用复制实例的方法生成新的实例并使用

### 3.3 建造者模式 Builder

builder定义构建接口，concrete builder实现接口

director调用builder的接口生成实例

### 3.4 抽象工厂模式 Abstract Factory

#### 3.4.1 对比工厂方法

工厂方法模式中，我们使用一个工厂创建一个产品，一个具体工厂对应一个具体产品

抽象工厂模式拥有多个抽象产品类（产品族）和一个抽象工厂类， 一个工厂能够提供多个产品对象，即一个产品族

#### 3.4.2 产品概念

产品等级结构：产品等级结构指的是产品的继承结构，例如一个空调抽象类，它有海尔空调、格力空调、美的空调等一系列的子类，那么这个空调抽象类和他的子类就构成了一个产品等级结构。

产品族：产品族是指由同一个工厂生产的，位于不同产品等级结构中的一组产品。比如，海尔工厂生产海尔空调、海尔冰箱，那么海尔空调则位于空调产品族中

#### 3.4.3 优缺点

易于增加新的具体工厂，难以增加新的零件

## 四、分开考虑

### 4.1 桥接模式 Bridge

将类的功能层次结构和实现层次结构分离，不使用继承而使用组合委托

#### 4.1.1 类的层次结构

功能层次结构：extends，实现层次结构：implement

#### 4.1.2 组成结构

- abstraction：持有implementer的实例，并简单封装了implementer的方法
- refined abstraction：在abstraction基础上新增了功能
- implementor：定义了abstraction接口依赖的方法
- concrete implementer：实现了implementer的接口

#### 4.1.3 优缺点

桥接模式可以取代多层继承方案，多层继承方案违背了“单一职责原则”，复用性较差，且类的个数非常多，桥接模式是比多层继承方案更好的解决方法，它极大减少了子类的个数，如下面的示意。

```
                   ----Shape---
                  /            \
         Rectangle              Circle
        /         \            /      \
BlueRectangle  RedRectangle BlueCircle RedCircle
```

```
          ----Shape---                        Color
         /            \                       /   \
Rectangle(Color)   Circle(Color)           Blue   Red
```

### 4.2 策略模式 Strategy

strategy负责定义策略所必须的接口，concrete strategy实现这些接口

context使用组合，持有concrete strategy实例

将策略实现解耦出来，使用委托的方式，可以动态替换算法

## 五、一致性

### 5.1 混合模式 Composite

#### 5.1.1 详细原理

目录树结构使用的设计模式，目录下即有文件也有子目录

容器和内容具有一致性，提供一致性的方法，创造出递归结构的模式

#### 5.1.2 组成结构

- Leaf：叶子节点，不能放入其他叶子
- Composite：容器节点，可以放入Leaf和Composite
- Component：使Leaf和Composite具有一致性的父类

### 5.2 装饰模式 Decorator

- component：定义了被装饰的对象的接口
- decorator：实现并组合了component
- concrete component：实现了被装饰component的接口
- concrete decorator：实现decorator的方法，使用委托

## 六、访问数据结构

### 6.1 访问者模式 Visitor

访问者模式基于混合模式的目录，将组件的处理从组件数据结构中分离出来，提高了组件的独立性

- component：访问者模式中增加accept方法调用visitor.visit(this)
- visitor：visitor对混合模式中的leaf和composite各自声明一个visitor接口
- concrete visitor：实现visitor中的接口

易于增加concrete visitor，不易于增加component，因为每个concrete visitor都需要适配新component

### 6.2 职责链模式 Chain of Responsibility

- handler：有next指针，定义了处理请求的接口
- concrete handler：实现处理请求的接口

弱化了调用方和处理方的关系，可以动态改变职责链

## 七、简单化

### 7.1 外观模式 Facade

facade为错综复杂无法解耦的部分提供统一的高层接口

### 7.2 中介者模式 Mediator

- mediator：定义与colleage角色进行通信和做出决定的接口
- concreate mediator：实现mediator的这些接口
- colleague：定义与mediator进行通信的接口
- concrete colleague：实现colleague的这些接口

colleage之间的交互，统一提交给mediator处理

## 八、管理状态

### 8.1 观察者模式（发布-订阅模式） Observer

- subject：表示观察对象，定义了注册、删除observer的方法，
- concrete subject：
- observer：定义接受observer通知的接口
- concrete observer：实现observer的接口

### 8.2 备忘录模式 Memento

- originator：会在保存自己最新状态时生成memento
- caretaker：通知originator生成memento，或者使用memento恢复状态
- memento：会将originator内部信息整合在一起；提供了宽接口和窄接口，宽接口用于获取恢复获取对象状态的信息，只给originator使用。caretaker只能访问窄接口。

caretaker和originator职责分开，当需要变更快照和恢复策略时，不需要更改originator

### 8.3 状态模式 State

- state：标识状态，定义了一系列接口，接受context作为入参，并调用context的方法处理状态流转
- concrete state：实现了state接口
- context：持有表示当前状态的concrete state，定义了供外部接口调用的接口

监控到消息事件后，context的方法调用持有的concrete state，state的方法又调用context方法更新状态

## 九、避免浪费

### 9.1 享元模式 Flyweight

通过尽量共享实例来避免new出实例

- flyweight：实例会被共享的类
- flyweight factory：是生成flyweight的工厂
- client：使用flyweight factory生成flyweight

### 9.2 代理模式 Proxy

#### 9.2.1 模式原理

- subject：为proxy和real subject提供一致性的接口
- proxy：实现了subject的接口，处理来自client的请求，对于无法处理才会交给real subject，proxy只有在必要时才会生成real subject角色
- real subject：实现了subject接口，由proxy调用，处理proxy无法胜任的工作
- client：使用proxy模式的客户端

#### 9.2.2 模式优缺点

使用proxy来提升处理速度，划分proxy和subject，划分职责不需要修改subject类

## 十、用类来表现

### 10.1 命令模式 Command

- command：定义命令的接口
- concrete command：实现command的接口
- receiver：是command命令的接受者，command执行时会调用receiver
- client：负责生成command并分配receiver
- invoker：执行命令的角色，调用command接口的execute方法

### 10.2 解释器模式 Interpreter

- abstract expression：定义了语法树节点的共同接口
- concrete expression：具体语法节点，从程序级别到变量级别
- context：为解释器进行语法解析提供了必要的上下文信息
- client：读取程序，一行一行解释

具体的回忆自己之前写过的解释器
