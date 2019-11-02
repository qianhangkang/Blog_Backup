---
title: IO模型及同步异步阻塞非阻塞
date: 2019-11-02 14:43:53
tags: ['操作系统']
---

概念混淆，重新理解

<!--more-->

# 同步、异步、阻塞、非阻塞

> In many circumstances they are different names for the same thing, but in some contexts they are quite different. So it depends. Terminology is not applied in a totally consistent way across the whole software industry.
>
> For example in the classic sockets API, a non-blocking socket is one that simply returns immediately with a special “would block” error message, whereas a blocking socket would have blocked. You have to use a separate function such as `select` or `poll` to find out when is a good time to retry.
>
> But asynchronous sockets (as supported by Windows sockets), or the asynchronous IO pattern used in .NET, are more convenient. You call a method to start an operation, and the framework calls you back when it’s done. Even here, there are basic differences. Asynchronous Win32 sockets “marshal” their results onto a specific GUI thread by passing Window messages, whereas .NET asynchronous IO is free-threaded (you don’t know what thread your callback will be called on).
>
> So they don’t always mean the same thing. To distil the socket example, we could say:
>
> - Blocking and synchronous mean the same thing: you call the API, it hangs up the thread until it has some kind of answer and returns it to you.
>
>   阻塞与同步意味着同一种意思：一个线程调用API，线程被挂起直到它收到一些回复或返回值
>
>   （阻塞更倾向于等待数据输入，同步倾向等待结果）
>
> - Non-blocking means that if an answer can’t be returned rapidly, the API returns immediately with an error and does nothing else. So there must be some related way to query whether the API is ready to be called (that is, to simulate a wait in an efficient way, to avoid manual polling in a tight loop).
>
>   非阻塞意味着如果返回值不能立即返回，那么被调用的API将会返回一个错误并且什么都不做。因此必须存在一些相关的方法来查询API是否准备好被调用（就是用有效的方式模拟一个等待来避免手动的轮询）
>
> - Asynchronous means that the API always returns immediately, having started a “background” effort to fulfil your request, so there must be some related way to obtain the result.
>
>   异步意味着API调用会立马返回，但是会在“后台”来处理你的调用，因此必须存在一些相关的方法来获取结果

# IO模型

## IO阻塞

![屏幕快照 2019-01-11 下午4.18.36.png](https://i.loli.net/2019/01/11/5c38516df0249.png)

进程阻塞于内核等待数据与之后将数据从内核复制到用户空间的这两个时间段

## 非阻塞IO

![pic2](https://i.loli.net/2019/01/11/5c38516e2103b.png)

进程一直轮询等待内核是否准备好数据，并不会被挂起或沉睡，直到内核通知进程数据准备完毕

> 这里所谓的“非阻塞”，也是针对调用recvfrom的进程来说的。对于用户来说，如果每次recvfrom发现没准备好，都继续等待，给用户的感受上其实也要算是一种“阻塞”。

## IO复用

![屏幕快照 2019-01-11 下午4.18.13.png](https://i.loli.net/2019/01/11/5c38516e26b74.png)

## 信号驱动式IO模型

![屏幕快照 2019-01-11 下午4.18.07.png](https://i.loli.net/2019/01/11/5c38516e1f2fc.png)

## 异步IO模型

![屏幕快照 2019-01-11 下午4.18.01.png](https://i.loli.net/2019/01/11/5c38516e1fccd.png)

## 比较

![屏幕快照 2019-01-11 下午4.17.55.png](https://i.loli.net/2019/01/11/5c38516e2896d.png)

1. 同步IO操作导致请求进程阻塞，直到IO完成
2. 异步IO操作不导致请求阻塞

# 参考链接

https://stackoverflow.com/questions/2625493/asynchronous-vs-non-blocking

http://www.programmr.com/blogs/difference-between-asynchronous-and-non-blocking

《Unix网络编程》

















> *大部分正义感都是虚伪的 聊胜于无   —————————————————————————沃德彭尤·胡某*

