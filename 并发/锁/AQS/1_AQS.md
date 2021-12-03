# 抽象队列同步器 AbstractQueuedSyncronizer （AQS）

## AQS 介绍

1. 抽象队列同步器，JUC并发包中的大部分的并发工具类，都是基于AQS实现的
2. AQS 定义了一套多线程访问共享资源的同步框架
3. 主要依赖一个状态state的同步。锁的重入也是依赖于这个state

## AQS 特性

1. 公平锁和非公平锁，依赖于等待队列和同步队列
2. 共享锁和独占锁，依赖于state
   1. 可重入，state==0 就获取锁，否则再判断是不是当前线程，是的话state+1，否则将线程封装成Node，加入等待队列
3. 允许中断

## Java.concurrent.util里并发类实现逻辑

1. 定义内部类： Sync 继承 AQS，调用同步器里方法实则调用Sync里的方法

2. 定义属性：state (volatitle 修饰)
                      AQS 里面已经实现了 getState()、setState()、compareAndSetState() 三个方法

3. 主要实现的方法：

   1. isHeldExclusively()：该线程是否正在独占资源

   2. 独占方式
      boolean tryAcquire(int arg)：尝试获取资源
      boolean tryRelease(int arg)：尝试释放资源

   3. 共享方式

      int tryAcquireShared(int arg)：尝试获取资源。返回剩余资源个数。负数则表示获取失败。
      boolean tryReleaseShared(int arg)：尝试释放资源



## 独占锁于共享锁

1. 独占锁：只有一个线程可以执行，例：ReentrantLock
2. 共享锁：多个线程可以同时执行，例：Semaphore、CountDownLatch

## AQS 的两种队列

1. 同步等待队列 （CLH 队列）
   1. **作用是保存等待在这个锁上的线程(由于lock()操作引起的等待）**
   2. 一种双向链表的队列，先入先出的等待队列
      1. 在 AbstractQueuedSynchronizer 维护 Node head 和 Node tail
2. 条件等待队列
   1. **Condition.await()引起阻塞的线程**
   2. 在子类 ConditionObject 维护 Node firstWaiter 和 Node lastWaiter

## ConditionObject

1. 用于线程间的通信，能够把锁粒度减小。重点是**await()和signal()**。

### await()

当前线程处于阻塞状态，直到调用signal()或中断才能被唤醒。

1. 将当前线程封装成node且等待状态为CONDITION。
2. 释放当前线程持有的所有资源，让下一个线程能获取资源。
3. 加入到条件队列后，则阻塞当前线程，等待被唤醒。
4. 如果是因signal被唤醒，则节点会从条件队列转移到等待队列；
   如果是因中断被唤醒，则记录中断状态。
   两种情况都会跳出循环。
5. 若是因signal被唤醒，就自旋获取资源；否则处理中断异常。