---
title: 浅谈GC
date: 2018-04-17 17:05:46
tags: ['JVM','GC']
---

> **垃圾回收**（英语：Garbage Collection），在计算机科学中，缩写为GC是一种自动的[存储器管理](https://zh.wikipedia.org/wiki/%E8%A8%98%E6%86%B6%E9%AB%94%E7%AE%A1%E7%90%86)机制。当一个电脑上的动态存储器不再需要时，就应该予以释放，以让出存储器，这种存储器资源管理，称为**垃圾回收**。垃圾回收器可以让程序员减轻许多负担，也减少程序员犯错的机会。垃圾回收最早起源于[LISP](https://zh.wikipedia.org/wiki/LISP)语言。目前许多语言如[Smalltalk](https://zh.wikipedia.org/wiki/Smalltalk)、[Java](https://zh.wikipedia.org/wiki/Java)、[C#](https://zh.wikipedia.org/wiki/C_Sharp)和[D语言](https://zh.wikipedia.org/wiki/D%E8%AF%AD%E8%A8%80)都支持垃圾回收器。

<!--more-->

# 什么对象需要垃圾回收？

垃圾回收，如何判定哪些对象需要回收呢？

## 引用计数法

> 最早期的垃圾回收实现方法，通过对数据存储的物理空间附加多一个计数器空间，当有其他数据与其相关时则加一，反之相关解除时减一，定期检查各储存对象的计数器，为零的话则认为已经被抛弃而将其所占物理空间回收。是最简单的实现，但存在无法回收循环引用的存储对象的缺陷。

如下图：

![互相引用](https://i.loli.net/2018/04/17/5ad60827157b3.png)

（图片来自[书呆子Rico](https://blog.csdn.net/justloveyou_)）

OK，既然不能用这种方式解决，那么就出现了可达性分析的算法。

## 可达性分析

> 近现代的垃圾回收实现方法，通过定期对若干根储存对象开始遍历，对整个程序所拥有的储存空间查找与之相关的存储对象和没相关的存储对象进行标记，然后将没相关的存储对象所占物理空间回收。

什么意思呢，就是通过一系列的名为"GC  Roots"的对象作为起始点，从这些节点开始向下搜索，搜索所走过的路径称为引用链（Reference Chain），当一个对象到GC Roots没有任何引用链相连，即从GC Roots到这个对象不可达，则证明此对象是不可用的。如图：

![可达性分析](https://pic3.zhimg.com/80/cb06b4bd6d62cf310b7f3014ab5cb2fc_hd.jpg)

左边的对象都是存活的，右边的都是可以回收的。（左边从上往下第一个对象为GC ROOTS）



那么，继续提出问题，什么对象可以作为GC ROOTS？

1. 虚拟机栈（栈帧中的本地变量表）中的引用的对象
2. 方法区中的类静态属性引用的对象
3. 方法区中的常量引用的对象
4. 本地方法栈中JNI（Native方法）的引用对象



# JVM如何进行垃圾回收？

OK，现在JVM知道了哪些对象需要回收了，那么怎么去回收呢？

## 四种垃圾回收算法

### 复制算法

> **“复制算法”将可用内存按容量划分为大小相等的两块，每次只使用其中的一块。当这一块的内存用完了，就将还存活着的对象复制到另外一块上面，然后再把已使用过的内存空间一次清理掉。**这种算法适用于对象存活率低的场景，比如新生代。这样使得每次都是对整个半区进行内存回收，内存分配时也就不用考虑内存碎片等复杂情况，只要移动堆顶指针，按顺序分配内存即可，实现简单，运行高效。

如图所示：

![复制算法](http://www.ityouknow.com/assets/images/2017/jvm/gc_garbage2.png)

一般来说，JVM肯定不是只用一半内存，否则这对于内存的使用率来说实在过于低下。经研究发现，新生代中的对象每次回收都基本只有10%存活，需要复制的对象较少，所以，通常情况下，JVM会将新生代内存分为Eden区和两个survivor区（[浅谈JVM内存模型](https://qianhangkang.github.io/2018/04/13/%E6%B5%85%E8%B0%88JVM%E5%86%85%E5%AD%98%E5%8C%BA%E5%9F%9F%E4%B8%8E%E5%AF%B9%E8%B1%A1%E5%88%9B%E5%BB%BA%E8%BF%87%E7%A8%8B/#JVM内存模型)），Eden区和两个survivor区的比例为8：1，也就是说新生代的可用内存为整个新生代的90%，新生代内存的利用率达到了90%，这远远超过了之前的50%。

### 标记清理

> “标记-清除”（Mark-Sweep）算法，如它的名字一样，算法分为“标记”和“清除”两个阶段：首先标记出所有需要回收的对象，在标记完成后统一回收掉所有被标记的对象。之所以说它是最基础的收集算法，是因为后续的收集算法都是基于这种思路并对其缺点进行改进而得到的。

主要缺点：

1. 效率问题，标记和清除过程的效率都不高
2. 空间问题，标记清除之后会产生大量不连续的内存碎片，导致之后需要分配较大对象时无法找到足够的连续内存而不得不提前触发GC，而GC的开销是很大的。

标记清楚如下图所示：

![标记清除算法](http://www.ityouknow.com/assets/images/2017/jvm/gc_garbage1.png)



### 标记整理

>  “标记整理”算法与“标记清除”算法非常相似，它也是分为两个阶段：**标记和整理**。首先标记出所有需要回收的对象，然后回收期间，会让所有存活的对象都向一端移动，然后直接清理掉端边界以外的内存

如下图所示

![标记整理算法](http://www.ityouknow.com/assets/images/2017/jvm/gc_garbage3.png)



### 分代收集

> GC分代的基本假设：绝大部分对象的生命周期都非常短暂，存活时间短。
>
> "分代收集"算法，把Java堆分为新生代和老年代，这样就可以根据各个年代的特点采用最适当的收集算法。在新生代中，每次垃圾收集时都发现有大批对象死去，只有少量存活，那就选用复制算法，只需要付出少量存活对象的复制成本就可以完成收集。而老年代中因为对象存活率高、没有额外空间对它进行分配担保，就必须使用“标记-清理”或“标记-整理”算法来进行回收。



## 垃圾收集器

### Serial收集器

> 串行收集器是最古老，最稳定以及效率高的收集器，可能会产生较长的停顿，只使用一个线程去回收。新生代、老年代使用串行回收；新生代复制算法、老年代标记-压缩；垃圾收集的过程中会Stop The World（服务暂停）

![Serial收集器](http://www.ityouknow.com/assets/images/2017/jvm/gc_garbage4.png)

### ParNew收集器

> ParNew收集器其实就是Serial收集器的多线程版本。新生代并行，老年代串行；新生代复制算法、老年代标记-整理

![ParNew收集器](http://www.ityouknow.com/assets/images/2017/jvm/gc_garbage5.png)

### Parallel收集器

> Parallel Scavenge收集器类似ParNew收集器，Parallel收集器更关注系统的吞吐量。可以通过参数来打开自适应调节策略，虚拟机会根据当前系统的运行情况收集性能监控信息，动态调整这些参数以提供最合适的停顿时间或最大的吞吐量；也可以通过参数控制GC的时间不大于多少毫秒或者比例；新生代复制算法、老年代标记-整理

### Parallel Old收集器

> Parallel Old是Parallel Scavenge收集器的老年代版本，使用多线程和“标记－整理”算法。这个收集器是在JDK 1.6中才开始提供

### CMS收集器

> CMS（Concurrent Mark Sweep）收集器是一种以获取最短回收停顿时间为目标的收集器。目前很大一部分的Java应用都集中在互联网站或B/S系统的服务端上，这类应用尤其重视服务的响应速度，希望系统停顿时间最短，以给用户带来较好的体验。
>
> 从名字（包含“Mark Sweep”）上就可以看出CMS收集器是基于“标记-清除”算法实现的，它的运作过程相对于前面几种收集器来说要更复杂一些，整个过程分为4个步骤，包括：
>
> - 初始标记（CMS initial mark）
> - 并发标记（CMS concurrent mark）
> - 重新标记（CMS remark）
> - 并发清除（CMS concurrent sweep）
>
> 其中初始标记、重新标记这两个步骤仍然需要“Stop The World”。初始标记仅仅只是标记一下GC Roots能直接关联到的对象，速度很快，并发标记阶段就是进行GC Roots Tracing的过程，而重新标记阶段则是为了修正并发标记期间，因用户程序继续运作而导致标记产生变动的那一部分对象的标记记录，这个阶段的停顿时间一般会比初始标记阶段稍长一些，但远比并发标记的时间短。
>
> 由于整个过程中耗时最长的并发标记和并发清除过程中，收集器线程都可以与用户线程一起工作，所以总体上来说，CMS收集器的内存回收过程是与用户线程一起并发地执行。

优点：

1. 并发收集
2. 低停顿

缺点：

1. **对CPU资源非常敏感。**面向并发设计的程序都对CPU资源比较敏感。在并发阶段，它虽然不会导致用户线程停顿，但会因为占用了一部分线程（或者说CPU资源）而导致应用程序变慢，总吞吐量会降低。CMS默认启动的回收线程数是（CPU数量+3）/4，也就是当CPU在4个以上时，并发回收时垃圾收集线程不少于25%的CPU资源，并且随着CPU数量的增加而下降。但是当CPU不足4个时（比如2个），CMS对用户程序的影响就可能变得很大，如果本来CPU负载就比较大，还要分出一半的运算能力去执行收集器线程，就可能导致用户程序的执行速度忽然降低了50%，其实也让人无法接受。
2. **无法处理浮动垃圾（Floating Garbage）**。可能出现“Concurrent Mode Failure”失败而导致另一次Full GC的产生。由于CMS并发清理阶段用户线程还在运行着，伴随程序运行自然就还会有新的垃圾不断产生。这一部分垃圾出现在标记过程之后，CMS无法再当次收集中处理掉它们，只好留待下一次GC时再清理掉。这一部分垃圾就被称为“浮动垃圾”。也是由于在垃圾收集阶段用户线程还需要运行，那也就还需要预留有足够的内存空间给用户线程使用，因此CMS收集器不能像其他收集器那样等到老年代几乎完全被填满了再进行收集，需要预留一部分空间提供并发收集时的程序运作使用。
3. **标记-清除算法导致的空间碎片** 。CMS是一款基于“标记-清除”算法实现的收集器，这意味着收集结束时会有大量空间碎片产生。空间碎片过多时，将会给大对象分配带来很大麻烦，往往出现老年代空间剩余，但无法找到足够大连续空间来分配当前对象。因此，为了解决空间浪费问题，CMS回收器不再采用简单的指针指向一块可用堆空间来为下次对象分配使用。而是把一些未分配的空间汇总成一个列表，当JVM分配对象空间的时候，会搜索这个列表找到足够大的空间来存放这个对象。

![CMS收集器](http://dl.iteye.com/upload/attachment/612577/2b525609-ce63-3a42-bf19-b2fbcd42f26c.png)

### G1收集器

> Garbage-First（G1）收集器是一种服务器式垃圾收集器，针对具有**大型内存的多处理器机器**。 它以高概率满足垃圾收集（GC）暂停时间目标，同时实现高吞吐量。垃圾收集器在Oracle JDK 7u4和更高版本中得到完全支持。
>
> G1垃圾收集算法主要应用在多CPU大内存的服务中，在满足高吞吐量的同时，竟可能的满足垃圾回收时的暂停时间，该设计主要针对如下应用场景：
>
> - 垃圾收集线程和应用线程并发执行，和CMS一样
> - 空闲内存压缩时避免冗长的暂停时间
> - 应用需要更多可预测的GC暂停时间
> - 不希望牺牲太多的吞吐性能
> - 不需要很大的Java堆 

具体的回收方式参见[这篇文章](http://www.importnew.com/27793.html)，讲的非常详细，很nice。



# JVM在什么时候垃圾回收？

## System.gc()

如下为System.gc()的源码：

```java
    /**
     * Indicates to the VM that it would be a good time to run the
     * garbage collector. Note that this is a hint only. There is no guarantee
     * that the garbage collector will actually be run.
     */
    public static void gc() {
        boolean shouldRunGC;
        synchronized(lock) {
            shouldRunGC = justRanFinalization;
            if (shouldRunGC) {
                justRanFinalization = false;
            } else {
                runGC = true;
            }
        }
        if (shouldRunGC) {
            Runtime.getRuntime().gc();
        }
    }
```

由源码可见，当justRanFinalization为true时，系统会在调用System.gc()时发生GC。这里只是为了强调System.gc()并不一定会强制发生gc，类似finalize方法。



## 系统自动GC

当运行时某些区域的内存不足时系统自动进行垃圾回收

参考自自[R神的解释](https://www.zhihu.com/question/41922036)：

> 1. young GC：当young gen中的eden区分配满的时候触发。注意young GC中有部分存活对象会晋升到old gen，所以young GC后old gen的占用量通常会有所升高。
> 2. full GC：当准备要触发一次young GC时，如果发现统计数据说之前young GC的平均晋升大小比目前old gen剩余的空间大，则不会触发young GC而是转为触发full GC（因为HotSpot VM的GC里，除了CMS的concurrent collection之外，其它能收集old gen的GC都会同时收集整个GC堆，包括young gen，所以不需要事先触发一次单独的young GC）；或者，如果有perm gen的话，要在perm gen分配空间但已经没有足够空间时，也要触发一次full GC；或者System.gc()、heap dump带GC，默认也是触发full GC。



总结一下就是系统自动回收主要看Eden区和old区的内存使用情况，不够了就触发相应的GC。



这里对GC的种类做一个分类：

> Partial GC：并不收集整个GC堆的模式
>
> - Young GC：只收集young gen的GC
> - Old GC：只收集old gen的GC。只有CMS的concurrent collection是这个模式
> - Mixed GC：收集整个young gen以及部分old gen的GC。只有G1有这个模式
>
> Full GC：收集整个堆，包括young gen、old gen、perm gen（如果存在的话）等所有部分的模式。



**Major GC通常是跟full GC是等价的，收集整个GC堆。但因为HotSpot VM发展了这么多年，外界对各种名词的解读已经完全混乱了，当有人说“major GC”的时候一定要问清楚他想要指的是上面的full GC还是old GC。**



因此更正下之前写的[浅谈JVM内存模型与对象创建](https://qianhangkang.github.io/2018/04/13/%E6%B5%85%E8%B0%88JVM%E5%86%85%E5%AD%98%E5%8C%BA%E5%9F%9F%E4%B8%8E%E5%AF%B9%E8%B1%A1%E5%88%9B%E5%BB%BA%E8%BF%87%E7%A8%8B/#新生代)的错误（已修正）...



# 参考链接

[维基百科（垃圾回收）](https://zh.wikipedia.org/wiki/%E5%9E%83%E5%9C%BE%E5%9B%9E%E6%94%B6_(%E8%A8%88%E7%AE%97%E6%A9%9F%E7%A7%91%E5%AD%B8))

https://blog.csdn.net/justloveyou_/article/details/71216049

https://www.zhihu.com/question/35164211

http://softbeta.iteye.com/blog/1315103

https://crowhawk.github.io/2017/08/15/jvm_3/

http://www.importnew.com/27793.html

https://www.zhihu.com/question/41922036


















> 大部分正义感都是虚伪的 聊胜于无   —————————————————————————沃德彭尤·胡某