---
layout: post
title: JMM & synchronized概述
category: cheatsheet
description: JAVA多线程知识点
---

参考[JMM & synchronized概述][1] 


根据Java语言规范中的说明，JVM系统中存在一个主内存（Main Memory），Java中所有的变量存储在主内存中，对于所有的线程是共享的（相当于黑板，其他人都可以看到的）。每个线程都有自己的工作内存（Working Memory），工作内存中保存的是主存中变量的拷贝，（相当于自己笔记本，只能自己看到），工作内存由缓存和堆栈组成，其中缓存保存的是主存中的变量的copy，堆栈保存的是线程局部变量。线程对所有变量的操作都是在工作内存中进行的，线程之间无法直接互相访问工作内存，变量的值得变化的传递需要主存来完成。在JMM中通过并发线程修改的变量值，必须通过线程变量同步到主存后，其他线程才能访问到。  

大体上来讲，线程对某个变量的操作可以简化成下面的步骤：
 
-  1.从主内存中复制数据到工作内存 (read and load)
-  2.执行代码，对数据进行各种操作和计算 (use and assign)
-  3.把操作后的变量值重新写回主内存中 (store and write)


当然这样的运行顺序也是我们所期望的！但是，JVM并不保证第1步和第3步会严格按照上述次序立即执行。因为根据java语言规范的规定，线程的工作内存和主存间的数据交换是松耦合的，什么时候需要刷新工作内存或者什么时候更新主存的内容，可以由具体的虚拟机实现自行决定。由于JVM可以对特征代码进行调优，也就改变了某些运行步骤的次序的颠倒，那么每次线程调用变量时是直接取自己的工作存储器中的值还是先从主存储器copy再取是没有保证的，任何一种情况都可能发生。同样的，线程改变变量的值之后，是否马上写回到主存储器上也是不可保证的，也许马上写，也许过一段时间再写。那么，在多线程的应用场景下就会出现问题了，多个线程同时访问同一个代码块，很有可能某个线程已经改变了某变量的值，当然现在的改变仅仅是局限于工作内存中的改变，此时JVM并不能保证将改变后的值立马写到主内存中去，也就意味着有可能其他线程不能立马得到改变后的值，依然在旧的变量上进行各种操作和运算，最终导致不可预料的结果。    

这样的情况是不是就不能避免了呢？   

Java程序员都知道synchronized关键字强制实施一个互斥锁，使得被保护的代码块在同一时间只能有一个线程进入并执行。当然synchronized还有另外一个方面的作用：在线程进入synchronized块之前，会把工作存内存中的所有内容映射到主内存上，然后把工作内存清空再从主存储器上拷贝最新的值。而在线程退出synchronized块时，同样会把工作内存中的值映射到主内存，但此时并不会清空工作内存。这样一来就可以强制其按照上面的顺序运行，以保证线程在执行完代码块后，工作内存中的值和主内存中的值是一致的，保证了数据的一致性！ 


[1]:http://www.iteye.com/topic/438068