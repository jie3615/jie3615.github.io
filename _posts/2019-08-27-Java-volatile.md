---
layout: post
title:  "Java并发系列-深入理解volatile"
date:   2019-08-27 20:06:06
categories: 并发
tags: Java 线程 volatile
---

* content
{:toc}
> ***在并发场景中我们经常会看到volatile的身影，它到底能解决哪些问题*？**



## volatile关键字的语义：

一旦一个共享变量被其修饰之后：

可见性：一个线程对这个变量的修改对其他线程可见，会立马得到最新值

有序性：被volatile修饰的变量会加入内存屏障



## 首先我们先验证可见性：

执行如下代码：

```
public static void main(String[] args){
      MyData myData = new MyData();
      new Thread(()->{
          try {
              sleep(3000);
          } catch (InterruptedException e) {
              e.printStackTrace();
          }
          myData.addData();

      },"线程一").start();

      while (myData.i == 0) {
      }
      
      //资源类
  class MyData{
    int i =0;
    void addData() {
        this.i=i+19;
    }
}
```



结果分析：主线程一直在做死循环，实际上线程一已经对data中的数据做了更改，由于在线程执行过程中i被拷贝到每个线程的变量副本当中，主线程感知不到data.i的变化，所以一直在循环；

解决方案：使用 volatile 修饰i；此时主函数会正常结束。

底层原理：首先明白cpu是多级缓存的架构模式，也是导致数据不一致的主要原因；这个时候cpu制造商规定一个缓存一致性协议来从硬件指令层面来解决这个问题；

## 如何解决缓存一致性问题？

使用总线锁：具体执行方式是当一个cpu对其缓存中数据做修改操作时，会向总线发出一个Lock信号，收到Lock信号的cpu会停止操作自己缓存中的数据，直到Lock被解除；这种方式有个很大的弊端就是性能问题，会导致cpu性能下降；

第二种方式使用MESI协议，是对cpu缓存加标志位的方式，共有四种状态

M:Modify，修改，cpu缓存中的数据已经合主存中不一致了

E:Exclusive，独占，当前cpu缓存和主存数据一致，并且其他cpu没有使用此缓存数据

S:Share，共享，与主存中的数据一致，并且可以被多组缓存共享

I:Invalid，失效，此缓存已经失效，不能使用



cpu对缓存的操作流程遵循MESI协议

如果缓存的状态是I，缓存无效，直接读取主存；

如果是M或者E，也就是说缓存已经被修改或者自己独占，这个时候有其他cpu读取操作的时候就把自己的缓存写入主存，并且将自己缓存状态设置成共享S。

只有缓存状态是M,E的时候，cpu才可以修改缓存中的数据，修改完成把状态改成M，区别于上一条，上一条是在其他cpu读取前刷到主存，变成共享状态；

使用上面的这种工作模式，每个cpu中的缓存都可以保证一致性，同时又不会导致性能下降太多；

在使用volatile的变量上，底层汇编会多出一条lock add $0x0,(%esp) 在以前的cpu架构中是采用锁总线的模式，现在基本上都是采用MESI协议，处理器之间使用嗅探技术来保证，cpu内部缓存，系统内存，以及和其他cpu缓存中的数据保持一致。



## 验证有序性：

```java
for (int i = 0; i < 100000; i++) {
        ResortSeqDemo resortSeqDemo = new ResortSeqDemo();
        Thread thread1 = new Thread(() -> {
            a = 1;
            x = b;
        }, String.valueOf(i));
        Thread thread2 = new Thread(() -> {
            b = 2;
            y = a;
        }, String.valueOf(i));
        x = 0;
        y = 0;
        a = 0;
        b = 0;
        thread1.start();
        thread2.start();
        try {
            thread1.join();
            thread2.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        if (x == 0 && y == 0) {
            System.out.println("指令重排");
        }
    }
}
```

结果分析：执行上面一段代码，会出现打印出指令重排的情况，如果出现不了，可以加大循环次数；

加上 volatile会解决以上问题：

底层原理分析：这是happens-before原则中的其中一条，其他的暂且不说；volatile修饰的变量会在写之后插入一个写屏障指令，在读之前插入一个读屏障指令；内存屏障可以保证指令的顺序执行；

---
  本文版权归作者本人拥有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。










