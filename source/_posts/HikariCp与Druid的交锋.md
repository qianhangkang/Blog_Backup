---
title: HikariCP与Druid的交锋
date: 2018-06-02 15:22:26
tags: ['闲谈','数据库']
---

看到GitHub上一个issue，关于HikariCP与Druid的性能，此间引出了两位大佬和众多网友的口水战。

余以为甚是有趣，特此翻译记录。

<!--more-->

![roit](https://i.loli.net/2018/06/02/5b1252d8e626a.jpg)

# 先介绍下两位

## Druid

> Druid是Java语言中最好的数据库连接池。Druid能够提供强大的监控和扩展功能。

## HikariCP

> Fast, simple, reliable. HikariCP is a "zero-overhead" production ready JDBC connection pool. At roughly 130Kb, the library is very light. 

极速、极简、可靠。HikariCP是一个“零开销”的数据库连接池。大约130kb，这个库非常轻量。



# 一位印度小哥引起的口水战

PS：一下翻译几乎来自谷歌，我只是润色了下。。。之前翻译的全丢了，TMD :(



首先，一位印度小哥在HikariCP的issue上提了一个问题

> Hi, I find your analysis on Java DB pools very informative. I happen to come across this 'druid' pool from Alibaba: <https://github.com/alibaba/druid/wiki/FAQ> (claims as the fastest DB pool in Java!).
>
> From my very quick glance, seems it has some cool features. Any thoughts on this. Thanks.

嗨，我发现你对Java DB池的分析非常丰富。 我碰巧发现了一个来自阿里巴巴的连接池Druid（声称是Java中速度最快的数据库池）。从我的快速浏览，似乎它有一些很酷的功能。 你对此有什么想法，请分享下，谢谢。



HikariCP作者**brettwooldridge**回答了这位印度小哥的问题

> A quick run of the benchmark on my desktop PC yielded the following:
>
> ```
> Benchmark                                 (pool)   Mode  Samples       Score  Units
> c.z.h.b.ConnectionBench.cycleCnnection    hikari  thrpt       16   21206.330  ops/ms
> c.z.h.b.ConnectionBench.cycleCnnection    bone    thrpt       16   10389.139  ops/ms
> c.z.h.b.ConnectionBench.cycleCnnection    vibur   thrpt       16    6764.233  ops/ms
> c.z.h.b.ConnectionBench.cycleCnnection    tomcat  thrpt       16    2117.792  ops/ms
> c.z.h.b.ConnectionBench.cycleCnnection    c3p0    thrpt       16     128.447  ops/ms
> c.z.h.b.ConnectionBench.cycleCnnection    druid   thrpt       16     110.370  ops/ms
> 
> c.z.h.b.StatementBench.cycleStatement     hikari  thrpt       16  130426.003  ops/ms
> c.z.h.b.StatementBench.cycleStatement     tomcat  thrpt       16   62071.180  ops/ms
> c.z.h.b.StatementBench.cycleStatement     druid   thrpt       16   54845.335  ops/ms
> c.z.h.b.StatementBench.cycleStatement     vibur   thrpt       16   33414.774  ops/ms
> c.z.h.b.StatementBench.cycleStatement     bone    thrpt       16   26551.919  ops/ms
> c.z.h.b.StatementBench.cycleStatement     c3p0    thrpt       16   15956.412  ops/ms
> ```
>
> So, at least in our benchmark, Druid was the *slowest* for obtaining and returning connections, and the third fastest for creating and closing statements.Druid was configured with similar settings to other pools in the benchmark:
>
> ```
> druid.setInitialSize(0);
> druid.setMinIdle(0);
> druid.setMaxActive(32);
> druid.setMaxWait(5000);
> druid.setValidationQuery("SELECT 1");
> druid.setTestOnBorrow(true);
> druid.setDefaultAutoCommit(false);
> druid.setDefaultTransactionIsolation(Connection.TRANSACTION_READ_COMMITTED);
> ```
>
> The benchmark page in their wiki does not show what settings they ran with, but I suspect that they disabled test-on-borrow. While I won't say that is "cheating", it is not how I would run a production pool. They also do not provide the source code to their test, as far as I can tell.

作者给出了一个测试数据，说明了Druid在获得和归还连接上的速度最慢，创建和关闭statements的速度第三。同时还说，他们wiki中的页面没有显示他们运行的设置，但我怀疑他们禁用了借用测试。 虽然我不会说这是“作弊”，但我不会运行生产池。 据我所知，他们也没有提供源代码给他们的测试。



**bobwenx**网友接着回复了

> druid was designed to focus on monitor and data access behavior enhance(like automatically database slice). it provides an SQL parser to analyze user's SQL query and delegate most JDBC class to collect metrics. so if what you need is a JDBC monitor solution like newrelic, you may try Druid.

Druid的目的是专注于监控和数据访问行为的增强（如自动数据库切片）。 它提供了一个SQL分析器来分析用户的SQL查询并委派大多数JDBC类来收集度量标准。
所以如果你需要的是像newrelic这样的JDBC监视器解决方案，你可以试试Druid。



作者**brettwooldridge**回复这位网友

> [@bobwenx](https://github.com/bobwenx) It is a valid point. I will point out that New Relic is also supported by HikariCP through the [DropWizard Metrics integration](https://github.com/brettwooldridge/HikariCP/wiki/Codahale-Metrics), but the metrics provided are "pool-level" metrics and not specific to query execution time, etc.

这是一个有效的观点。 我会指出newrelic也受到HikariCP通过DropWizard指标集成的支持，但提供的度量标准是“池级”度量标准，并且不针对查询执行时间等。



此时天空一声巨响，Druid负责人温少出来找场子了

>If you configure maxWait property, druid pool use "fair mode ReentrantLock", it's bad performance.
>
><https://github.com/alibaba/druid/blob/master/src/main/java/com/alibaba/druid/pool/DruidAbstractDataSource.java>
>
>```java
> public void setMaxWait(long maxWaitMillis) {
>        if (maxWaitMillis == this.maxWait) {
>            return;
>        }
>
>        if (maxWaitMillis > 0 && useUnfairLock == null && !this.inited) {
>            final ReentrantLock lock = this.lock;
>            lock.lock();
>            try {
>                if ((!this.inited) && (!lock.isFair())) {
>                    this.lock = new ReentrantLock(true);
>                    this.notEmpty = this.lock.newCondition();
>                    this.empty = this.lock.newCondition();
>                }
>            } finally {
>                lock.unlock();
>            }
>        }
>
>        if (inited) {
>            LOG.error("maxWait changed : " + this.maxWait + " -> " + maxWaitMillis);
>        }
>
>        this.maxWait = maxWaitMillis;
>    }
>
>```
>
>This is by design, because we had some problems in the production environment. Here are the relevant [documents](<https://github.com/alibaba/druid/wiki/Druid%E9%94%81%E7%9A%84%E5%85%AC%E5%B9%B3%E6%A8%A1%E5%BC%8F%E9%97%AE%E9%A2%98>), but I'm sorry, it is in Chinese.
>
>Druid of PreparedStatementCache optimized, this is very important to enhance the mysql 5.5 & oracle & sqlserver & db2 performance.
>
>Druid has very stable ExceptionSorter, including Oracle / MySql / Alibaba Oceanbase etc.
>
>In [Taobao large-scale high concurrency environment](http://www.forbes.com/sites/jlim/2015/11/10/alibaba-group-sells-5bn-in-first-90-minutes-of-11-11-sale/#3de3850f1449), only two connection pools to work very well, [druid](https://github.com/alibaba/druid) and jboss connection pool.
>
>Druid is not just a connection pool, he can do a very good extension, similar to Filter-Chain extension. Built-in Filter include [WallFilter](https://github.com/alibaba/druid/wiki/%E7%AE%80%E4%BB%8B_WallFilter), [StatFilter](https://github.com/alibaba/druid/wiki/%E9%85%8D%E7%BD%AE_StatFilter), [LogFilter](https://github.com/alibaba/druid/wiki/%E9%85%8D%E7%BD%AE_LogFilter). Which can WallFilter defense SQL injection, StatFilter can do performance monitoring, LogFilter can output SQL logs.
>
>Druid built very strong monitoring support.

（你说都是自己人，回答的时候就中英双语并用呗。。。）

大意就是Druid启用了公平锁模式，这会降低性能。

至于为什么这么做，是因为他们为了解决一些生产上的问题，同时给了一个[中文文档](<https://github.com/alibaba/druid/wiki/Druid%E9%94%81%E7%9A%84%E5%85%AC%E5%B9%B3%E6%A8%A1%E5%BC%8F%E9%97%AE%E9%A2%98>)。

Druid对PreparedStatementCache进行了优化，这对于增强mysql 5.5＆oracle＆sqlserver＆db2性能非常重要。

Druid拥有非常稳定的ExceptionSorter，包括Oracle / MySql /阿里巴巴Oceanbase等。

在淘宝大型高并发环境下，只有两个连接池工作得很好，druid和jboss连接池。

Druid不仅仅是一个连接池，他可以做一个很好的扩展，类似于Filter-Chain扩展。 内置过滤器包括WallFilter，StatFilter，LogFilter。 其中WallFilter可以防御SQL注入，StatFilter可以做性能监视，LogFilter可以输出SQL日志。

Druid建立了非常强大的监控支持。



**hrchu**网友指出了提问者的一个小错误

> Hmmm I only see it claims as "the best" DB pool in Java, instead of the fast... 😕
>
> Anyway I am glad to see more positive comparisons between HikariCP and Druid...

提问者在修饰Druid上使用了“the fastest”，实际上Druid的wiki上用的是“the best”。同时这位网友很乐意看到大家对于两者之间更多积极的讨论



省略一些无意义的讨论...



一位中国网友说

> druid有阿里大数据量的验证，而HikariCP没有



作者**brettwooldridge**回复中国网友

> HikariCP is one of the most widely used connection pools in the world, used by some of the largest companies, serving literally billions of users per day. Druid is rarely seen outside of China.

HikariCP是世界上使用最广泛的连接池之一，由一些最大的公司使用，每天为数十亿用户提供服务。 Druid在中国以外很少见到。



紧接着这位中国网友对作者回复的“最大的公司”、“每天为数十亿用户服务”表示怀疑，希望作者能给出例子



作者**brettwooldridge**再次回复中国网友

> Wix.com by itself, for example hosts over 109 million websites and serves over 1 Billion requests per day. Atlassian has millions of customers for its products -- no published numbers exists for daily usage.
>
> HikariCP is the default pool for every application built with the Play framework, Slick, JOOS, and is now the default pool for Spring Boot.
>
> HikariCP is resolved from the central maven repository over 300,000 times per month.
>
> [Companies known to be using HikariCP](https://github.com/brettwooldridge/HikariCP/blob/dev/documents/Wall-of-Fame.md).

意思就是我的东西就是这么牛13，你要的例子我都给你了



最后，另外一位中国网友做了如下中肯的评价

> 我认为2个组件专注的方向不一样，各有特色和优点，不是组件如何，要看使用组件的人如何考虑，综合自己的需求选用，druid非常大一部分是为了监控和运维



# 综合网上的资料总结下

## Druid偏大数据

- 需要交互式聚合和快速探究大量数据时；
- 需要实时查询分析时；
- 具有大量数据时，如每天数亿事件的新增、每天数10T数据的增加；
- 对数据尤其是大数据进行实时分析时；
- 需要一个高可用、高容错、高性能数据库时



## HikariCP偏优化

- **字节码精简** ：优化代码，直到编译后的字节码最少，这样，CPU缓存可以加载更多的程序代码；
- **优化代理和拦截器**：减少代码，例如HikariCP的Statement proxy只有100行代码，只有BoneCP的十分之一；
- **自定义数组类型（FastStatementList）代替ArrayList**：避免每次get()调用都要进行range check，避免调用remove()时的从头到尾的扫描；
- **自定义集合类型（ConcurrentBag**：提高并发读写的效率；
- **其他针对BoneCP缺陷的优化**，比如对于耗时超过一个CPU时间片的方法调用的研究（但没说具体怎么优化）。



客官你怎么看。。。



# 参考链接

https://github.com/brettwooldridge/HikariCP/issues/232

http://blog.didispace.com/Springboot-2-0-HikariCP-default-reason/





> *大部分正义感都是虚伪的 聊胜于无   —————————————沃德彭尤·胡某*