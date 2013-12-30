---
layout: post
title: 多线程同步
category: cheatsheet
description: 多线程同步机制
---

##互斥锁（mutex）  

mutex是一个特殊的变量，它有锁上(lock)和打开(unlock)两个状态。mutex一般被设置成全局变量。打开的mutex可以由某个线程获得。一旦获得，这个mutex会锁上，此后只有该线程有权打开。其它想要获得mutex的线程，会等待直到mutex再次打开的时候。我们可以将mutex想像成为一个只能容纳一个人的洗手间，当某个人进入洗手间的时候，可以从里面将洗手间锁上。其它人只能在mutex外面等待那个人出来，才能进去。在外面等候的人并没有排队，谁先看到洗手间空了，就可以首先冲进去。  

##条件变量（condition variable）  

条件变量(condition variable)是利用线程间共享的全局变量进行同步的一种机制，主要包括两个动作：一个线程等待某个条件为真，而将自己挂起；另一个线程使的条件成立，并通知等待的线程继续。为了防止竞争，条件变量的使用总是和一个互斥锁结合在一起。  

使用条件等待有如下的场景：  

多线程访问一个互斥区域内的资源，如果获取资源的条件不够时，则线程需要等待，直到其他线程释放资源后，唤醒等待条件，使得线程得以继续。例如：  


```
Thread1：

Lock (mutex)

while (condition is false) {

//为什么要在这里用while而不是if呢? 
Cond_wait(cond, mutex, timeout)

}

DoSomething()

Unlock (mutex)

 

Thread2:

Lock (mutex)

…

condition is true

Cond_signal(cond)

Unlock (mutex)
```

`为什么用while而不用if`Linux中帮助中提到的：
在多核处理器下，pthread_cond_signal可能会激活多于一个线程（阻塞在条件变量上的线程）。 结果是，当一个线程调用pthread_cond_signal()后，多个调用pthread_cond_wait()或pthread_cond_timedwait()的线程返回。这种效应成为”虚假唤醒”。虽然虚假唤醒在pthread_cond_wait函数中可以解决，为了发生概率很低的情况而降低边缘条件（fringe condition）效率是不值得的，纠正这个问题会降低对所有基于它的所有更高级的同步操作的并发度。所以pthread_cond_wait的实现上没有去解决它。
所以通常的标准解决办法是这样的：将条件的判断从if 改为while 。  

##信号量（semaphore）  

信号量（Semaphore）是一个非负的整数计数器，被用于进程或线程间的同步与互斥。
通过信号量可以实现 “PV操作”这种进程或线程间的同步机制。
P操作是获得资源，将信号量的值减1，如果结果不为负则继续执行，线程获得资源，否则线程被阻塞，处于睡眠状态，直到等待的资源被别的线程释放；
V操作则是释放资源，给信号量的值加1，释放一个因执行P操作而等待的线程。  

可以将信号量Semaphore和互斥锁(mutex)来实现一个来实现对一个池的同步和保护。使用mutex来实现同步，使用semaphore用于实现对资源记数。
获得资源的线程：

```
sem_wait (semaphore1)
Lock (mutex)
…
Unlock (mutex)
sem_post (semaphore2)
 
释放资源的线程:
sem_wait (semaphore2)
Lock (mutex)
…
Unlock (mutex)
sem_post (semaphore1)
```

这个模型很像多线程的生产者与消费者模型，这里的semaphore2是为了防止过度释放。
比起信号量来说，条件变量可以实现更为复杂的等待条件。当然，条件变量和互斥锁也可以实现信号量的功能（window下的条件变量只能实现线程同步不能实现进程同步）。  
 
在Posix.1基本原理一文声称，有了互斥锁和条件变量还提供信号量的原因是：“本标准提供信号量的而主要目的是提供一种进程间同步的方式；这些进程可能共享也可能不共享内存区。互斥锁和条件变量是作为线程间的同步机制说明的；这些线程总是共享(某个)内存区。这两者都是已广泛使用了多年的同步方式。每组原语都特别适合于特定的问题”。尽管信号量的意图在于进程间同步，互斥锁和条件变量的意图在于线程间同步，但是信号量也可用于线程间，互斥锁和条件变量也可用于进程间。应当根据实际的情况进行决定。信号量最有用的场景是用以指明可用资源的数量。

##读写锁（reader-writer lock）  

Reader-writer lock与mutex非常相似，有三种状态: 共享读取锁(shared-read)，互斥写入锁(exclusive-write lock)，打开(unlock)。后两种状态与之前的mutex两种状态完全相同。

一个unlock的RW lock可以被某个线程获取R锁或者W锁。如果被一个线程获得R锁，RW lock可以被其它线程继续获得R锁，而不必等待该线程释放R锁。但是，如果此时有其它线程想要获得W锁，它必须等到所有持有共享读取锁的线程释放掉各自的R锁。
如果一个锁被一个线程获得W锁，那么其它线程，无论是想要获取R锁还是W锁，都必须等待该线程释放W锁。
这样，多个线程就可以同时读取共享资源。而具有危险性的写入操作则得到了互斥锁的保护。  

参考:  
【1】[Linux多线程与同步][1]  
【2】[对条件变量(condition variable)的讨论][2]

[1]: http://www.cnblogs.com/vamei/archive/2012/10/09/2715393.html
[2]: http://blog.csdn.net/nhn_devlab/article/details/6117239#t1


       
