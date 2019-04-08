---
layout: post
title:  "Java线程生命周期"
categories: 多线程
tags: 并发
author: supernova
description: Java线程生命周期
---
## 通用线程的生命周期
* 初始状态  
在编程语言层面，线程已经创建(new Thread)，但是在操作系统层面，线程还未被创建。  
* 可运行状态   
线程在编程语言层面被启动(start())，在操作系统层面被创建。  
* 运行状态  
被分配到CPU的线程的状态  
* 休眠状态  
运行状态的线程调用阻塞的API或者等待某个事件，线程就会进入休眠状态并且交出CPU使用权。    
* 终止状态  
线程执行完毕，或者出现异常，就会进入终止状态。

## java线程的生命周期
* NEW(初始化)
* RUNABLE(运行)
* BLOCKED(阻塞)
遇到synchronized时，会进入对象的阻塞队列，等现阶段持有锁的线程释放锁后争抢锁。一直获得了互斥锁后再次回到运行状态。
* WAITING(无时限等待)
    * 获得synchronized的线程调用wait方法，直到被notify
    * A.join()方法,直到A线程结束
    * lockSupport.park()->lockSupport.upPack()
* TIMED_WAITING(有时限等待)
    * sleep(millis)
    * join(millis)
    * wait(millis)
    * lockSupport.parkNanos( Object blocker,long)
    * lockSupport.parkUntil(long)
* TERMINATED(终止)
    * run执行完
    * stop()  
    已被标记为不建议使用。此方法会真的立即杀死线程，即使该线程正拥有synchronized锁。造成其他线程无法再获得此锁。  
    * interrupt
    仅仅是通知线程interrupt，线程有机会执行后续操作，也可以无视这个操作。线程收到通知的方式：
        * 异常  
        处于WAITING和TIMED_WAITING的线程，在被执行interrupt方法是，触发InterruptExcetion
        处于RUNABLE的线程，并阻塞在java.nio.channels.InterruptibleChannel，触发ClosedByInterruptException异常  
        阻塞在java.nio.channels.Selector上时，ava.nio.channels.Selector会立即返回。
        * 主动检查  
        通过isInterrrupt方法检测
    * run异常
    
    ## 创建多少线程合适
    ### CPU密集型
    大部分场景下，都是纯CPU计算，很少的IO操作。
    CPU密集型的策略是提升CPU的利用率。对于一个4核的cpu一般创建4个线程，再多线程就增加了线程切换的成本。所以理论上是：线程数=CPU核数。  
    但实际工程上采用线程数=CPU数+1,在偶尔的内存页失效或者其他原因导致的阻塞时，多的一个线程能够顶上，最大的提升CPU利用率。
    ###IO密集型
    CPU计算和IO操作交叉执行，由于IO的设备和CPU操作比较起来速度非常的慢。所以大部分情况下IO操作的时间相较于CPU操作来说都比较长，这种场景一般叫做IO密集型。
    单线程：最佳线程数 =1 +（I/O 耗时 / CPU 耗时），多线程：最佳线程数 =CPU核数*[1 +（I/O 耗时 / CPU 耗时）]