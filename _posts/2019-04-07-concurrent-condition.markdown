---
layout: post
title:  "Lock和Condition"
categories: 多线程
tags: 并发
author: supernova
description: Lock和Condition
---
## 再造管程的理由
Java SDK 并发包通过 Lock 和 Condition两个接口来实现管程，其中 Lock 用于解决互斥问题，Condition 用于解决同步问题。  
原因是 synchronized 申请资源的时候，如果申请不到，线程直接进入阻塞状态了，  
而线程进入阻塞状态，啥都干不了，也释放不了线程已经占有的资源。但我们希望的是：对于“不可抢占”这个条件，占用部分资源的线程进一步申请其他资  
源时，如果申请不到，可以主动释放它占有的资源，这样不可抢占这个条件就破坏掉了。  
* 能够响应中断  
void lockInterruptibly() throws InterruptedException;
* 支持超时
boolean tryLock(long time,TimeUnit unit) throws InterruptedException;
* 非阻塞地获取锁
boolean tryLock();
## 可见性
lock用一个volatile变量state，在加锁和解锁的时候都修改state，根据happens-before的传递性，第一个线程的lock before  第二个线程的lock。

```
class SampleLock {
  volatile int state;
  // 加锁
  lock() {
    // 省略代码无数
    state = 1;
  }
  // 解锁
  unlock() {
    // 省略代码无数
    state = 0;
  }
}

```

## 可重入锁
reentrant指的是线程可以重复获取同一把锁。指的是多个线程可以同时调用该函数，每个线程都能得到正确结果；  
同时在一个线程内支持线程切换，无论被切换多少次，结果都是正确的。
```
class X {
  private final Lock rtl =
、  new ReentrantLock();
  int value;
  public int get() {
    // 获取锁
    rtl.lock();         ②
    try {
      return value;
    } finally {
      // 保证锁能释放
      rtl.unlock();
    }
  }
  public void addOne() {
    // 获取锁
    rtl.lock();  
    try {
      value = 1 + get(); ①
    } finally {
      // 保证锁能释放
      rtl.unlock();
    }
  }
}

```

## 公平锁与非公平锁

```
// 无参构造函数：默认非公平锁
public ReentrantLock() {
    sync = new NonfairSync();
}
// 根据公平策略参数创建锁
public ReentrantLock(boolean fair){
    sync = fair ? new FairSync() 
                : new NonfairSync();
}

```

如果一个线程没有获得锁，就会进入等待队列，当有线程释放锁的时候，就需要从等待队列中唤醒一个等待的线程。  
如果是公平锁，唤醒的策略就是谁等待的时间长，就唤醒谁，很公平； 
如果是非公平锁，则不提供这个公平保证，有可能等待时间等待时间短的线程反而先被唤醒。

## 用锁的最佳实践
* 永远只在更新对象的成员变量时加锁
* 永远只在访问可变的成员变量时加锁
* 永远不在调用其他对象的方法时加锁

## Condition实现条件变量
Java 语言内置的管程里只有一个条件变量，而 Lock&Condition 实现的管程是支持多个条件变量。

```
public class BlockedQueue<T>{
  final Lock lock =
    new ReentrantLock();
  // 条件变量：队列不满  
  final Condition notFull =
    lock.newCondition();
  // 条件变量：队列不空  
  final Condition notEmpty =
    lock.newCondition();

  // 入队
  void enq(T x) {
    lock.lock();
    try {
      while (队列已满){
        // 等待队列不满
        notFull.await();
      }  
      // 省略入队操作...
      // 入队后, 通知可出队
      notEmpty.signal();
    }finally {
      lock.unlock();
    }
  }
  // 出队
  void deq(){
    lock.lock();
    try {
      while (队列已空){
        // 等待队列不空
        notEmpty.await();
      }  
      // 省略出队操作...
      // 出队后，通知可入队
      notFull.signal();
    }finally {
      lock.unlock();
    }  
  }
}

```

## Dubbo异步转同步
