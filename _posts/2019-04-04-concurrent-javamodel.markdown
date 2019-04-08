---
layout: post
title:  "java内存模型"
categories: 多线程
tags: 并发
author: supernova
description: java内存模型解决线程安全性问题
---
## java内存模型
我们知道造成多线程问题的主要原因主要有3个：缓存导致的可见性问题、线程切换导致的原子性问题、编译优化导致的顺序性问题。
java是一门高级语言，自然会针对并发问题提供了一系列的解决方案。  
java内存模型就是用来解决可见性问题和顺序性问题的。  
导致这两个问题的缘由是因为缓存和编译优化，我们是否可以禁止它们。答案是否定的，之所以会提出这2种技术，就是为了提高程序的执行效率，如果禁止了就像从信息时代回到了石器时代。  
所以合理的办法就是按需禁止编译优化和缓存。何时禁用只有程序员知道，所以按需禁用就是按照程序员的要求禁用。  
所以java内存模型的作用就是规范了程序员禁用缓存和编译优化的方法。    

## volatile
volatile只能修饰成员变量，它的作用是任何线程对这个变量操作时禁止cpu缓存读写转而实时从内存种读写。很明显的它可以解决可见性的问题。  
其实volatile也解决了顺序性的问题，具体会在下面happens-before中介绍。    

## happens-before 
happens-before约束了编译优化的顺序性问题，它规定了happens-before之前的操作结果对happens-before之后的操作一定是可见的。  
* 程序的顺序性规则  
在一个线程中，前面操作对后面操作是可见的。
```
class VolatileExample {
  int x = 0;
  public void writer() {
    x = 42;
    v = x + 3;
  }
}
在单线程中第6行代码对于第7行是可见的
```
* 传递性规则  
如果 A Happens-Before B，且 B Happens-Before C，那么 A Happens-Before C。  
* volatile规则  
对volatile变量的写操作一定happens-before 对volatile变量的读操作。结合传递性可得出结论：   
当A线程执行writer，B线程执行reader方法时 x一定=42  
 
```
// 以下代码来源于【参考 1】
class VolatileExample {
  int x = 0;
  volatile boolean v = false;
  public void writer() { A线程
    x = 42;
    v = true;
  }
  public void reader() {B线程
    if (v == true) {
      // 这里 x 一定是42  由于v是volatile的，所以在对v=tr后，Bread时一定时42true。 又因为 x=42 before v = ture，v == true before v=true，根据传递性x一定=42
      
    }
  }
}

```

* 管程中锁的规则  
主要是针对synchronized的操作。    
它指的是A线程在synchronizad代码块中对共享变量进行操作然后释放锁，B线程获得锁进入代码块是可以看见之前A的操作结果。  

```
synchronized (this) { // 此处自动加锁
  // x 是共享变量, 初始值 =10
  if (this.x < 12) {
    this.x = 12; 
  }  
} // 此处自动解锁

```

* 线程start规则  
如果A线程调用了B线程的start方法，那么B线程能够看到A线程在启动B线程之前的操作。

```
Thread B = new Thread(()->{
  // 主线程调用 B.start() 之前
  // 所有对共享变量的修改，此处皆可见
  // 此例中，var==77
});
// 此处对共享变量 var 修改
var = 77;
// 主线程启动子线程
B.start();

```

* 线程中断规则  
一个线程调用另一个线程的interrupt happens-before于被中断的线程发现中断  
* 线程join规则  
A线程调用了B线程的join方法，那么在B线程结束之后，A是能看到B对共享变量的修改的。  

```
Thread B = new Thread(()->{
  // 此处对共享变量 var 修改
  var = 66;
});
// 例如此处对共享变量修改，
// 则这个修改结果对线程 B 可见
// 主线程启动子线程
B.start();
B.join()
// 子线程所有对共享变量的修改
// 在主线程调用 B.join() 之后皆可见
// 此例中，var==66

```

* 对象终结法则  
一个对象的创建一定happens-before对象的finalizer  

## final
final是正向激励，它告诉编译器和cpu，final修饰的变量是稳定不变的，可以尽情的优化哦。  
但是要避免引用的逸出例如：  

```
// 以下代码来源于【参考 1】
final int x;
// 错误的构造函数
public FinalFieldExample() { 
  x = 3;
  y = 4;
  // 此处就是讲 this 逸出，
  global.obj = this;
}

```

我们之前介绍过由于重排序的原因，x=3可能在global.obj = this之后执行，此时通过this.x=0