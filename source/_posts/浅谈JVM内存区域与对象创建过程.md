---
title: 浅谈JVM内存区域与对象创建过程
date: 2018-04-13 13:49:19
tags: ['JVM','对象创建','内存模型']
---



> 上次面试本以为熟悉了这块内容，但实际上自己还是对这块知识的概念挺模糊的。

<!--more-->

# JVM内存模型

## 概述

> Java虚拟机在执行Java程序的过程中会把它所管理的内存划分为若干个不同的数据。这些区域都有各自的用途，以及创建和销毁的时间，有的区域随着虚拟机进程的启动而存在，有些区域则是依赖用户线程的启动和结束而建立和销毁。Java虚拟机所管理的内存如下图所示。

![JVM内存模型_看图王.jpg](https://i.loli.net/2018/04/13/5ad051fd3f6c5.jpg)



## 程序计数器

> 是一块较小的内存空间，它可以看作是当前线程所执行的字节码的行号指示器。
>
> 由于Java虚拟机的多线程是通过线程轮流切换并分配处理器的执行时间的方式来实现的，在任何一个确定的时间，一个处理器（对于多核处理器来说是一个内核）都只会执行一个线程中的指令。因此，**为了线程切换后能恢复到正确的执行位置**，每条线程都需要有一个独立的程序计数器，各条线程之间计数器互不影响，独立存储，因此这块内存区域是**“线程私有”**的。



## 虚拟机栈

> 与程序计数器一样，虚拟机栈也是**“线程私有”**的，它的生命周期与线程相同。一般存放了编译期可知的各种基本数据类型、对象引用和returnAddress类型（指向了一条字节码指令的地址）



## 本地方法栈

> 本地方法栈与虚拟机栈所发挥的作用是非常相似的，它们之间的区别不过是虚拟机栈执行Java方法（也就是字节码）服务，而本地方法栈则为虚拟机使用Native方法服务。



## Java堆

> 对于大多数应用来说，Java堆 是Java虚拟机所管理的内存中最大的一块。Java堆是被所有**线程共享**的一块内存区域，在虚拟机启动时创建。主要存放了对象的实例和数组。Java堆也是垃圾回收的主要区域。



### 新生代

#### Eden区

> 对象一般优先再Eden区上分配。如果空间不够会触发一次young GC，如果还不够，那么则触发一次full GC。

#### From Survivor与To Survivor

> 新生代中，每次使用的是Eden区和一个survivor区，这个区就是from survivor。当发生young GC时，会将Eden区和from survivor区中还存活着的对象复制到to survivor区中，然后清理掉Eden区和from survivor区。from survivor区和to survivor都是相对来说的~~~ 一般情况下，HotSpot虚拟机默认的Eden区和一个survivor区的大小比例为8:1。



### 老年代

> 老年代一般存放的都是经过**分配担保**的大对象和经过数次GC并且经过**动态年龄判定**的对象。

- 分配担保：写了一半感觉自己解释不清楚，放个[链接](https://www.jianshu.com/p/62c37dc7d638)吧。。。
- 动态年龄判定：如果再survivor区中相同年龄所有对象大小的总和大于survivor空间的一半，那么年龄大于等于该年龄的对象就可以直接进入老年代，无需等到MaxTenuringThreshold（默认为15岁，即经过15次GC后才会进入老年代）中要求的年龄。



## 方法区

> 方法区与堆一样，是各个**线程共享**的内存区域，它用于存储**已被虚拟机加载的类信息、常量、静态变量、即时编译期编译后的代码等数据**。



### 运行时常量池

> 运行时常量池是方法区的一部分。主要用于存储编译期生成的各种字面量和符号引用。



# 对象创建过程

![JVM内存分配策略](https://i.loli.net/2018/04/15/5ad35745e57bc.jpg)



> 对象的内存分配，往大方向讲，就是在堆上分配（但也可能经过JIT编译后被拆散为标量类型并简介地栈上分配），对象主要分配在新生代的Eden区上，如果启动了本地线程分配缓冲，将按线程优先在TLAB上分配。少数情况下也可能会直接分配在老年代，分配的规则并不是百分百固定的，其细节取决于当前使用的是哪一种垃圾收集器组合，还有虚拟机中与内存相关的参数的设置。



## 对象在Eden区中分配

```java
//VM 参数:-verbose:gc -Xms20m -Xmx20m -Xmn10m  -XX:+PrintGCDetails -XX:+SurvivorRatio=8
public class Test {  
    private static final int _1MB = 1024*1024;  
  
    public static void main(String[] args) {
        byte[] allocation1,allocation2,allocation3,allocation4;
        allocation1 = new byte[2 * _1MB];
        allocation2 = new byte[2 * _1MB];
        allocation3 = new byte[2 * _1MB];
        allocation4 = new byte[4 * _1MB];
    }
}
```

如上述代码所述，allocation1，allocation2，allocation3都会分配在Eden区中，而在为allocation4分配内存时会发生一次young GC，又因为allocation1，allocation2，allocation3都不需要被回收，to survivor区又放不下，所以他们会被移至老年代，然后allocation4被分配在Eden区。



## 大对象直接进入老年代

```java
/**  
* VM参数：-verbose:gc -Xms20M -Xmx20M -Xmn10M -XX:SurvivorRatio=8 
* -XX:PretenureSizeThreshold=3145728 
*/
public class Test {  
    private static final int _1MB = 1024*1024;  
  
    public static void main(String[] args) {
        byte[] allocation;  
　  	   allocation = new byte[4 * _1MB]; 
    }
}
```

如上述代码所示，allocation会直接分配到老年代中。



# JVM常用控制参数

- -Xms设置堆的最小空间大小。
- -Xmx设置堆的最大空间大小。
- -XX:NewSize设置新生代最小空间大小。
- -XX:MaxNewSize设置新生代最大空间大小。
- -XX:PermSize设置永久代最小空间大小。
- -XX:MaxPermSize设置永久代最大空间大小。
- -Xss设置每个线程的堆栈大小。



# 参考链接

http://www.ityouknow.com/jvm/2017/08/25/jvm-memory-structure.html

https://juejin.im/post/5a14de6751882555cc417df7













> 大部分正义感都是虚伪的 聊胜于无   —————————————————————————沃德彭尤·胡某