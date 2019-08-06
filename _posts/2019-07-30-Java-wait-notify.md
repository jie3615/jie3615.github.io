---
layout: post
title:  "Java并发系列-从消费者生产者模型理解wait/notify"
date:   2019-8-04 20:06:06
categories: 并发
tags: Java 线程 Jvm
---

* content
{:toc}

wait/notify实现生产者消费者

线程的协作机制wait/notify，线程除了有竞争关系，他们还存在着协作，下面我们就用

##  生产者/消费者模型的特性

1. ​    消费者、生产者是通过一个缓冲区进行通信的，这个缓冲区可以是阻塞队列；

2. ​    生产者在队列满的时候停止生产；

3. ​    消费者在队列空的时候停止消费；

4. ​     生产者/消费者模型可以提高程序执行效率，我们可以调节生产者、消费者的个数来达到处理速度的平衡，提高资源利用率；

5. ​    还有一个重要的特性，可以解耦，对于耦合性强的程序，我们可以通过这种方式进行解耦处理，当然现在各种mq很多，也是利用了这个思路来完成解耦；

   

   > 研究wait/notify底层机制也有段时间了，今天我不打算从jvm层面说的太多，因为这这个完全可以开个章节，很多东西可以聊。我们都知道这是Object提供的三个方法，经常被忽视，或者我们很多时候压根用不到。但是了解它的精髓对我们了解并发这一块知识会有很大的帮助。
   >
   >  既然说到Object的方法，那肯定是每个对象都有，为什么？因为它在jvm层面是跟ObjectMonitor息息相关，这个类就是承载我们对象监视器的地方，也就是每个对象都有一把锁。现在不需要知道太多，但是有两点我希望你们可以明白，第一个执行wait的线程在waitSet(等待池)，被唤醒的线程在entrySet(姑且叫锁池吧)；下面开始撸代码.






   ## wait/notify实现生产者消费者

   我们以商品为例，首先定义商品类

   ```java
    class Goods {
           //编号
           private int code;
           //名称
           private String name;
   }
   ```

   下面使用LinkList来实现我们的阻塞队列，也就是缓冲区，在wait调用的时候一定是获得锁的前提下，否则可能会出现Lost Wake-Up现象，暂时不展开，有兴趣自己去查查；

   ```java
    class MyQueue<T> {
           /**
            *默认大小
            */
           final int DEFAULT_SIZE = 10;
           /**
            *实际大小
            */
   
           int realSize;
           LinkedList<T> linkedList = new LinkedList<>();
   
           public MyQueue(int realSize) {
               this.realSize = realSize <= 0 ? DEFAULT_SIZE : realSize;
           }
           public synchronized void put(T t) throws InterruptedException {
               while (linkedList.size() >= realSize) {
                   try {
                       this.wait();//达到最大上限，停止生产
                   } catch (InterruptedException e) {
                       e.printStackTrace();
                   }
               }
               linkedList.add(t);
               Thread.sleep(0);
               System.out.println("生产者："+Thread.currentThread().getName()+t.toString());
               this.notifyAll();
           }
           public synchronized void get() {
   
               while (linkedList.size() <= 0) {
                   try {
                       this.wait();
                   } catch (InterruptedException e) {
                       e.printStackTrace();
                   }
               }
               T t = linkedList.pollFirst();
               System.out.println("消费者：" + Thread.currentThread().getName() + t.toString());
               this.notifyAll();//消费完一个可以唤醒生产者继续生产
           }
       }
   ```

   生产者/消费者

   ```java
    class Producer implements Runnable{
           MyQueue myQueue;
           public Producer(MyQueue myQueue) {
               this.myQueue = myQueue;
           }
           @Override
           public void run() {
               AtomicInteger atomicInteger = new AtomicInteger(0);
               while (true) {
                   Goods goods = new Goods();
                   goods.setCode(atomicInteger.incrementAndGet());
                   goods.setName("商品" + atomicInteger.get());
                   try {
                       myQueue.put(goods);
                   } catch (InterruptedException e) {
                       e.printStackTrace();
                   }
               }
           }
       }
       class Consumer implements Runnable{
           MyQueue myQueue;
           public Consumer(MyQueue myQueue) {
               this.myQueue = myQueue;
           }
           @Override
           public void run() {
               while (true) {
                   myQueue.get();
               }
           }
       }
   ```

   主函数，定义一个生产者，一个消费者，看看结果

   ```java
    public static void main(String[] args) {
           MyQueue<Goods> myQueue = new MyQueue<Goods>(1);
           Thread thread0 = new Thread(new Producer(myQueue), "线程0");
   //        Thread thread1 = new Thread(new Producer(myQueue), "线程1");
   //        Thread thread2 = new Thread(new Consumer(myQueue),"线程2");
   //        Thread thread3 = new Thread(new Consumer(myQueue),"线程3");
           Thread thread4 = new Thread(new Consumer(myQueue),"线程4");
   //        thread1.start();
   //        thread2.start();
   //        thread3.start();
           thread4.start();
           thread0.start();
       }
   ```

   执行结果

   ```java
   生产者：线程0Goods{code=1, name='商品1'}
   消费者：线程4Goods{code=1, name='商品1'}
   生产者：线程0Goods{code=2, name='商品2'}
   消费者：线程4Goods{code=2, name='商品2'}
   生产者：线程0Goods{code=3, name='商品3'}
   消费者：线程4Goods{code=3, name='商品3'}
   ......
   
   ```

   ## 过程分析

   假设我们缓冲池大小设置为10；

   生产者生产第一个商品，发现缓冲池未满，第一个商品入队；此时消费者可能也起来了发现缓冲池为空，执行wait，此时的消费者线程进入了waitSet队列；

   接下来生产者线程执行唤醒操作，加入我们是单线程，那么使用notify没关系，否则可能会造成程序假死，后面说；

   被唤醒的线程进入了entrySet队列进行锁的争夺战；（消费者线程就在里面）

   当生产者释放锁的时候消费者刚好抢到锁，然后执行消费操作；接下来生产者执行，如此交替下去相安无事；

   单遇到生产者队列满的现象，或者消费者队列为空的现象。线程都会主动挂起，等待对方唤醒。

   ## 存在的问题

   1. 上面提到的程序假死，什么情况下会出现假死，也就是互相等待，假如我们的生产者有一个，消费者有两个，采用了notify()来唤醒对方。如果遇到了下面的情况

      

      `缓冲池中只有一个商品，生产者生产完成进入waitSet，两个消费者一个能正常消费，另一个也会进入waitSet`

      `当消费完成之后进行notify操作的时候，由于我们是随机唤醒等待池中的一个线程，假如此时唤醒的是另个消费者，那队列是空的，此时两个消费者全部进入等待池，生产者同样也在，任何一个线程都缺少了被唤醒的可能，从而导致假死的现象；`

      

   2. 第二个问题，也是在多线程环境下出现的，如果读者把我上面主函数中代码逻辑注释放开的话，可以在把get /put方法中的while条件改成if试试，可能会出现数组越界，或者其他异常。

      ```java
      Connected to the target VM, address: '127.0.0.1:16904', transport: 'socket'
      Exception in thread "线程2" java.lang.IllegalMonitorStateException
      	at java.lang.Object.wait(Native Method)
      	at java.lang.Object.wait(Object.java:502)
      	at jie.blog.source.threadtest.state.MyQueue.get(生产者消费者1.java:96)
      	at jie.blog.source.threadtest.state.Consumer.run(生产者消费者1.java:145)
      	at java.lang.Thread.run(Thread.java:748)
      Exception in thread "线程3" java.lang.IllegalMonitorStateException
      ```

      为什么呢?试想，如果我们的队列中没有商品，两个消费者同时执行到了if 判断这里，此时两个消费者线程都执行wait，进入等待池，当生产者生产了一个唤醒等待池中所有的消费者的时候，此时两个消费这不会重新进入条件判断，而是进行消费，因为只有一个商品，必定会抛出异常。

      

      其实对于生产者消费者我们还有其他更简单的方式，比如采用jdk提供的阻塞队列，给个简单的例子，就不一一解释了，后面讲到线程池肯定会细细说来

      ```java
       static class Producer implements Runnable {
              LinkedBlockingQueue<Integer> linkedBlockingQueue ;
      
              public Producer(LinkedBlockingQueue<Integer> linkedBlockingQueue) {
                  this.linkedBlockingQueue = linkedBlockingQueue;
              }
              @Override
              public void run() {
                  int i=0;
                  try {
                      while (true) {
                          linkedBlockingQueue.put(++i);
      //                    Thread.sleep(100);
                          System.out.println("生产者线程"+Thread.currentThread().getName()+"#####"+i);
                      }
      
                  } catch (InterruptedException e) {
                      e.printStackTrace();
                  }
      
              }
          }
      
          static class Consumer implements Runnable{
              LinkedBlockingQueue<Integer> linkedBlockingQueue ;
              public Consumer(LinkedBlockingQueue<Integer> linkedBlockingQueue) {
                  this.linkedBlockingQueue = linkedBlockingQueue;
              }
              @Override
              public void run() {
                  int i = 0;
                  while (true) {
      
                      try {
      
                          i = linkedBlockingQueue.take().intValue();
      //                    Thread.sleep(100);
                      } catch (InterruptedException e) {
                          e.printStackTrace();
                      }
                      System.out.println("消费者线程"+Thread.currentThread().getName()+"#####"+i);
                  }
      
              }
          }
      
          public static void main(String[] args){
             LinkedBlockingQueue<Integer> linkedBlockingQueue = new LinkedBlockingQueue<>(10);
              Producer producer = new Producer(linkedBlockingQueue);
              Consumer consumer = new Consumer(linkedBlockingQueue);
              Thread t1 = new Thread(producer, "1");
              Thread t2 = new Thread(consumer, "2");
              t1.start();
              t2.start();
          }
      
      ```

      

---
  本文版权归作者本人拥有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。










