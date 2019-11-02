---
title: Java注解【未完】
date: 2018-04-17 22:58:08
tags: ['Java','注解']
---

> Annotation其实是一种接口。通过Java的反射机制相关的API来访问Annotation信息。相关类（框架或工具中的类即使用注解的类）根据这些信息来决定如何使用该程序元素或改变它们的行为。

<!--more-->

# 注解

## 元注解

### @Documented
> 被该注解修饰的类会生成到Javadoc中

### @Retention

> @Retention定义了该Annotation被保留的时间长短。
>
> - RetentionPolicy.SOURCE 
>   源码级别，注解只存在源码中。主要是与编译器交互，用于代码检测。如@Override，@SuppressWarnings。额外效率损耗发生在编译时。
> - RetentionPolicy.CLASS
>   字节码级别，注解存在于源码与字节码文件中。主要用于编译时生成额外的文件，如XML，Java文件等，但运行时无法获得。这个级别需要添加JVM加载时候的代理，使用代理来动态修改字节码文件。
> - RetentionPolicy.RUNTIME 
>   运行时级别，注解存在于源码，字节码与Java虚拟机中。主要用于运行时反射相关信息

### @Target

> 说明了该Annotation所修饰的对象范围

#### ElementType[] value();

```java
public enum ElementType {
        /** Class, interface (including annotation type), or enum declaration */
        TYPE,

        /** Field declaration (includes enum constants) */
        FIELD,

        /** Method declaration */
        METHOD,

        /** Formal parameter declaration */
        PARAMETER,

        /** Constructor declaration */
        CONSTRUCTOR,

        /** Local variable declaration */
        LOCAL_VARIABLE,

        /** Annotation type declaration */
        ANNOTATION_TYPE,

        /** Package declaration */
        PACKAGE,

        /**
         * Type parameter declaration
         *
         * @since 1.8
         */
        TYPE_PARAMETER,

        /**
         * Use of a type
         *
         * @since 1.8
         */
        TYPE_USE
}
```
​	jdk1.8中共有10个取值
​	- TYPE：用来修饰类、接口（包括注解类型）和枚举类型
​	- FIELD：用来修饰域（包含枚举常量），域相当于类里的成员变量
​	- METHOD：用来修饰方法
​	- PARAMETER：用来修饰形式参数
​	- CONSTRUCTOR：用来修饰构造器
​	- LOCAL_VARIABLE：用来修饰局部变量
​	- ANNOTATION_TYPE：用来修饰注解类型
​	- PACKAGE：用来修饰包
​	- TYPE_PARAMETER（@since 1.8）：用来修饰参数类型
​	- TYPE_USE（@since 1.8）：可以用于标注任意类型(不包括class)

### @Inherited
> 阐述了某个被标注的类型是被继承的。如果一个使用了@Inherited修饰的annotation类型被用于一个class，则这个annotation将被用于该class的子类

## 标准注解
 - @Override：表示被修饰的方法覆盖了父类的方法
 - @Deprecated：表示被修饰的类或类成员已经在当前jdk版本被遗弃了，通常来说使用这些是危险的，也可能是因为出现了其他更好的解决方案。
 - @SuppressWarnings：表示告诉Java编译器关闭对类、方法及成员变量的警告。






# 参考链接
https://juejin.im/entry/57e496fd7db2a20063a24a3a
http://josh-persistence.iteye.com/blog/2226493
































> 大部分正义感都是虚伪的 聊胜于无   —————————————————————————沃德彭尤·胡某