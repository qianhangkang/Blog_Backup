---
title: IDEA DEBUG
date: 2019-11-02 14:41:10
tags: ['idea']
---

IDEA断点的一些设置

<!--more-->

![idea debug](https://i.loli.net/2019/03/01/5c78c9496b585.png)

# 属性

## Enabled

> Deselect to temporarily disable a breakpoint without removing it from the project. Disabled breakpoints will be skipped during the debugging process.

启用禁用。一般不动

## Suspend

> Select to pause the program execution when a breakpoint is hit. Suspending an application is useful if you need to obtain logging information or calculate an expression at a certain point without interrupting the program. If you need to create a *master* breakpoint that will trigger dependent breakpoints when hit, choose not to suspend the program at that breakpoint.

命中断点后暂停程序

### All

> all threads will be suspended

DEFAULT

### Thread

> only the thread containing this breakpoint will be suspended. If you want the Thread policy to be used as the default one, click the Make default button.

如果想要在多线程中debug每个线程，可以设置该属性。每个包含了该断点的线程都会在此暂停，你可以选择相应的Frames来执行相应的线程，以此控制线程执行的顺序

## Condition

> Select to specify a condition for hitting a breakpoint. A condition is a Java Boolean expression including a method returning `true` or `false`, for example, `str1.equals(str2)`.This expression must be valid at the line where the breakpoint is set, and it is evaluated each time the breakpoint is hit.If the evaluation result is `true`, the selected actions are performed.

给命中断点带上特定的条件。传入的条件需要时Java的布尔表达式

## Log

> Select if you want to log the following events to console:

### Breakpoint hit

命中断点后打日志

### Stack trace

命中断点后打堆栈日志

## Pass count

> Select if you need a breakpoint to be triggered only after it has been hit a certain number of times. This is useful for debugging loops or methods called several times.
>
> NOTE: If both the Pass Count and a Condition are set, IntelliJ IDEA first satisfies the condition, and then checks for pass count to avoid conflicts between the two settings.

命中指定次数的断点后才会暂停程序。

如果同时设置了Condition，IDEA 优先匹配Condition，然后再判断Pass count

## Evaluate and Log

> Select to evaluate an expression when the breakpoint is hit, and show the result in the console output.

## Remove once hit

> Select to remove the breakpoint from your project right after it has been hit.

## Disable until breakpoint is hit

> Select the breakpoint that will trigger the current breakpoint. Until that breakpoint is hit, the current breakpoint will be disabled. You can also select if you wish to disable it again or leave it enabled once it has been hit.

## Instance filters

> Select to limit breakpoint hits with particular object instances. Enter instance IDs separated by spaces, or click ![browse](https://www.jetbrains.com/help/img/idea/2018.3/icons.general.openDisk.svg@2x.png) and add them in the Instance Filters dialog.

选择命中特定对象实例的断点。

## Class filters

> Select to filter classes where the breakpoint must be hit. Either type in the filters, or click ![browse](https://www.jetbrains.com/help/img/idea/2018.3/icons.general.openDisk.svg@2x.png) and configure filters in the Class Filters dialog.You can specify class names or class patterns (strings with the * wildcard). If a filter is specified through a class name, it points at the class itself and all its subclasses. A filter specified through a class pattern points at classes whose fully qualified names match this pattern.

选择命中特定的类。可以使用*匹配模式

## Caller filters

> Select if you need to stop at a breakpoint only when it is called (or NOT called) from a certain method. Either type in the caller methods, or click and configure filters in the
>
> Caller Filters dialog.
>
> You can specify method names or method patterns (strings with the * wildcard).

选择是否只有在某个方法调用（或未调用）时才需要在断点处停止。

# Watches

可以在watches里动态的改变变量或表达式的值















> *大部分正义感都是虚伪的 聊胜于无   —————————————————————————沃德彭尤·胡某*

