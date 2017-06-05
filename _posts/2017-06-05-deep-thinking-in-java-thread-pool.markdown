---
layout:     post
title:      "Deep thinking in Java thread pool"
subtitle:   ""
date:       2017-06-05 21:15:35
author:     "Echo Yuan"
#header-img: "img/in-post/hello-world/hello-world.gif"
tags:
    - Java
    - 多线程
    - 线程池
    - ThreadPool
---
在深入思考Java线程池之前，我觉得确实应该静下心来好好读一读并认真理解下[ThreadPoolExecutor](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/ThreadPoolExecutor.html)的类和方法的注释，要清楚的知道在传入不同参数的情况下ThreadPool会产生怎样的行为。

#### corePoolSize & maximumPoolSize
这两个值可不能乱设，不然可能会与你期望的行为完全不一样。

* corePoolSize用于设定thread pool需要时刻保持的最小core threads的数量，即便这些core threads处于空闲状态啥事都不做也不会将它们回收掉，当然前提是你没有设置allowCoreThreadTimeOut为true。至于pool是如何做到保持这些个threads不死的，我们稍后再说。
* maximumPoolSize用于限定pool中线程数的最大值。

#### On-demand construction
按需构建。

如果让我们来设计如何构建thread pool中的thread的话，大部分人的选择可能都是thread pool一开，马上create core thread到corePoolSize指定的数量；当core threads不够用时，再一个个递增到maximumPoolSize。

然而，Sun's way并非如此。Sun的做法是
1. 当有新的task submitted时，如果当前pool中core thread的数量<corePoolSize，那么就创建一个新的core thread来处理此次请求并把这个task作为core thread的first task，即使当前正有其他core thread处于idle状态也是如此；
2. 当core thread的数量>=corePoolSize时，将task放入由整个pool所共享的一个queue中；
3. 当queue满了后，如果当前thread的数量<maximumPoolSize，那就创建一个新的thread来处理这个task；
4. 当queue满了且thread的数量也>=maximumPoolSize时，pool会reject掉这个task。有以下几个reject policy可选，当然也可以定义自己的policy。
    * AbortPolicy
      pool默认的policy，直接抛出一个RuntimeException。
    * DiscardPolicy
      啥都不做直接忽略reject行为的一个policy。
    * DiscardOldestPolicy
      从queue的头部弹出一个最老的task，然后把当前要处理的这个task塞进去。
    * CallerRunsPolicy
      由调用者线程来run这个task，pool撒手不管了。


#### Core thread & Non-core thread
其实Core thread和Non-core thread没有任何区别，都是普通的thread而已，生生区分出来core thread只是为了说明它们是常驻在pool中的，pool需要保持corePoolSize个core thread来随时响应请求。Non-core thread就没这么好命了，在keepAliveTime为0的情况下，run方法执行完这个线程就这么结束了。

那Core thread是如何能一直保持alive的呢？Core thread除了执行它的first task之外，还会不断地从workQueue中取task来执行，原因就在Core thread从queue中取元素用的是take()方法，这是一个block方法，当queue中没有元素时，当前线程会一直阻塞直到有task可用。

下面就是重点了。到底哪些thread会成为Core thread呢？其实并不一定是最开始创建的那些threads，也并不一定是最后创建的那些threads，而是执行时间最长或者说活的时间最久的那些threads。在thread pool运作的过程中，它会不停地检测当前线程数是否>corePoolSize，如果条件成立，threads会不断地消亡；直到<=corePoolSize，pool中的thread会稳定在corePoolSize个。这里的前提是allowCoreThreadTimeOut为false。具体的逻辑还是请看代码吧。

![get-task-in-threadpoolexecutor](/img/in-post/deep-thinking-in-java-thread-pool/get-task-from-thread-pool.png)

#### keepAliveTime
默认情况下，keepAliveTime是只针对Non-core threads来说的哦，core thread不受它的影响，除非你设置了allowCoreThreadTimeOut为true。


