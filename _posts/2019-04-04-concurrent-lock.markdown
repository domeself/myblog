---
layout: post
title:  "互斥锁"
categories: 多线程
tags: 并发
author: supernova
description: 互斥锁解决原子性问题
---
## 原子性问题分析
一个或者多个操作在 CPU 执行的过程中不被中断的特性，称为“原子性”。  
原子性问题的源头是线程切换，而操作系统做线程切换是依赖 CPU 中断的，所以禁止 CPU 发生中断就能够禁止线程切换。  
同一时刻只有一个线程执行”这个条件非常重要，我们称之为互斥。如果我们能够保证对共享变量的修改是互斥的，  
那么，无论是单核 CPU 还是多核 CPU，就都能保证原子性。  

## synchronized
Java 语言提供的 synchronized 关键字，就是锁的一种方案。它既可以修饰方法，也可以修饰代码块。  
* 修饰方法时，锁的对象是什么？  
当修饰静态方法的时候，锁定的是当前类的 Class 对象。  
当修饰非静态方法时，锁定的是this。

## 锁和被保护资源之间的关系  
正常的关系是一个锁保护多个资源。  
那么一个资源可不可以用多个锁来保护呢？ 
 
### 多个锁保护一个资源

```
class SafeCalc {
  static long value = 0L;
  synchronized long get() {
    return value;
  }
  synchronized static void addOne() {
    value += 1;
  }
}

```

get方法使用this的锁来保护value。
addOne方法使用SafeCalc.class来保护value。
可以看见，2个方法是2个邻界资源，没有互斥关系，会导致并发问题。
  
### 保护无关联的多个资源      
假如有2个资源AB，我们可以用一个锁同时保护AB。这样一来对这两个资源的操作我们都要排队来执行，即使对A的操作根本不会对B产生影响。  
所以结果就是效率非常的低。那我们试试用2个不同的锁，来分别保护一个资源。  
这样一来对A的操作只会和对A的操作互斥。这种锁还有个名字，叫细粒度锁。

### 保护有关联的多个资源  
转账业务，账户A减少100元，账户B增加100元。这两个账户就是有关联关系的。  

```
class Account {
  private int balance;
  // 转账
  void transfer(
      Account target, int amt){
    if (this.balance > amt) {
      this.balance -= amt;
      target.balance += amt;
    }
  } 
}

```
* 方法一 方法加锁
方法上面加锁只能保护this，即本人的余额被保护了，别人的却没有。
* 方法二 让不同账户共享一把锁  
    * 创建一把锁lock，在创建不同账户时，将lock作为构造参数让每个账户都持有。用lock将转账方法保护起来。  
    这个方法虽然可以解决问题，但是在实际开发中并不可行，因为账户的创建往往在分散在多处，  
    甚至是多个工程，传入共享的lock不现实。  
    * 将Accout.class作为锁，Account一定是唯一的，不需要创建对象时传入。咋一看这个方法完美的解决的问题。  
    但是仔细想想Acount.class锁定的是所有的账户，在同一时间全银行只能有一个账户能进行转账，这个结果不可接受。  
* 方法三 既然一把锁行不通那就使用2把锁分别保护  

```
class Account {
  private int balance;
  // 转账
  void transfer(Account target, int amt){
    // 锁定转出账户
    synchronized(this) {              
      // 锁定转入账户
      synchronized(target) {           
        if (this.balance > amt) {
          this.balance -= amt;
          target.balance += amt;
        }
      }
    }
  } 
}

```

这个实现看上去也很美，但是还是会有问题，什么问题呢？就是非常著名的死锁问题。  

### 死锁问题怎么解决
* 找一个管理员将锁管理起来，必须要同时获得2把锁，才有资格访问同步代码块  

```
class Allocator {
  private List<Object> als =
    new ArrayList<>();
  // 一次性申请所有资源
  synchronized boolean apply(
    Object from, Object to){
    if(als.contains(from) ||
         als.contains(to)){
      return false;  
    } else {
      als.add(from);
      als.add(to);  
    }
    return true;
  }
  // 归还资源
  synchronized void free(
    Object from, Object to){
    als.remove(from);
    als.remove(to);
  }
}

class Account {
  // actr 应该为单例
  private Allocator actr;
  private int balance;
  // 转账
  void transfer(Account target, int amt){
    // 一次性申请转出账户和转入账户，直到成功
    while(!actr.apply(this, target))
      ；
    try{
      // 锁定转出账户
      synchronized(this){              
        // 锁定转入账户
        synchronized(target){           
          if (this.balance > amt){
            this.balance -= amt;
            target.balance += amt;
          }
        }
      }
    } finally {
      actr.free(this, target)
    }
  } 
}

```
   
* 让两把锁排序，无论多少个线程近来，都得按同一个顺序请求所资源，只有申请到了第一个锁的线程才有资格申请第二个锁。  

```
class Account {
  private int id;
  private int balance;
  // 转账
  void transfer(Account target, int amt){
    Account left = this        ①
    Account right = target;    ②
    if (this.id > target.id) { ③
      left = target;           ④
      right = this;            ⑤
    }                          ⑥
    // 锁定序号小的账户
    synchronized(left){
      // 锁定序号大的账户
      synchronized(right){ 
        if (this.balance > amt){
          this.balance -= amt;
          target.balance += amt;
        }
      }
    }
  } 
}

```

## 等待-通知
实现方式：synchronized+ wait()+ notify() or notifyAll()  
同一时刻，只允许一个线程进入 synchronized 保护的临界区，当有一个线程进入临界区后，其他线程就只能进入图中左边的等待队列里等待。  
这个等待队列和互斥锁是一对一的关系，每个互斥锁都有自己独立的等待队列。
在并发程序中，当一个线程进入临界区后，由于某些条件不满足，  
需要进入等待状态，Java 对象的 wait() 方法就能够满足这种需求。当调用 wait() 方法后，当前线程就会被阻塞，  
并且进入到另外一个等待队列中，这个等待队列也是互斥锁的等待队列。 线程在进入等待队列的同时，会释放持有的互斥锁，  
线程释放锁后，其他线程就有机会获得锁，并进入临界区了。
线程要求的条件满足时，调用 notify()，会通知等待队列（互斥锁的等待队列）中的线程，  
告诉它条件曾经满足过。为什么说是曾经满足过呢？因为notify() 只能保证在通知时间点，条件是满足的。  
而被通知线程的执行时间点和通知的时间点基本上不会重合，所以当线程执行的时候，很可能条件已经不满足了（保不齐有其他线程插队）。  
被通知的线程要想重新执行，仍然需要获取到互斥锁因为曾经获取的锁在调用 wait() 时已经释放了。
这些方法必须在synchronized{}内部被调用的。实际上这些方法是通过synchronized锁定的对象调用的。
最终的实现方式：

```
class Allocator {
  private List<Object> als;
  // 一次性申请所有资源
  synchronized void apply(
    Object from, Object to){
    // 经典写法
    while(als.contains(from) ||
         als.contains(to)){
      try{
        wait();
      }catch(Exception e){
      }   
    } 
    als.add(from);
    als.add(to);  
  }
  // 归还资源
  synchronized void free(
    Object from, Object to){
    als.remove(from);
    als.remove(to);
    notifyAll();
  }
}

```

