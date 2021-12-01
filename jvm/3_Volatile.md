# 并发编程的三大特性

## 原子性问题

32位操作系统，基本数据类型是原子的;   long、double不是原子的

解决方案：Synchronized、lock 保证每时每刻只有一个线程

## 可见性问题

主内存和工作内存的延迟现象

解决方案：Volatile: 在工作内存中修改完之后更新到主存 + 嗅探机制

​                   Synchronized、lock  保证每时每刻只有一个线程

## 有序性问题

指令重排 （时机：编译器优化重排序、指令级并行重排序、内存系统重排序）

解决方案：Volatile: 禁止指令重排

​                   Synchronized、lock  只有一个线程顺序执行



## as-if-serial与happens-before

1. as-if-serial:
   不管怎么重排，单线程执行的结果不会被改变

2. happens-before:

   两个线程间顺序执行，

   跨线程内存可见性保证：尽管A线程写操作au 与 B线程写操作b 在happens-before，尽管a,b操作不走同一个线程，但是a操作对b操作可见



## happens-before 8项规则

================================常规===========================================

1. 程序顺序规则：一个线程中的每个操作，happens-before于该线程中的任意后续操作。
2. 监视器锁规则：对一个锁的解锁，happens-before于随后对这个锁的加锁。
3. volatile变量规则：对一个volatile域的写，happens-before于任意后续对这个volatile域的读。
4. 传递性：如果A happens-before B，且B happens-before C，那么A happens-before C。
   ================================线程的三个动作===========================================
5. start()规则：如果线程A执行操作ThreadB.start()（启动线程B），那么A线程的ThreadB.start()操作happens-before于线程B中的任意操作。
6. join()规则：如果线程A执行操作ThreadB.join()并成功返回，那么线程B中的任意操作happens-before于线程A从ThreadB.join()操作成功返回。
7. 程序中断规则：对线程interrupted()方法的调用先行于被中断线程的代码检测到中断时间的发生。
   ================================对象的生命周期===========================================
8. 对象finalize规则：一个对象的初始化完成（构造函数执行结束）先行于发生它的finalize()方法的开始。
   



## Volatile

1. volatile 保证了可见性（MESI）、原子性（内存屏障），但是不保证原子性



### DCL问题

因为new 对象需要三个步骤，而其中2、3可能存在指令重排。但发生重排时，对象的引用指向一个未被初始化的地址（空对象）

1. 分配空间
2. 初始化对象
3. 指向刚分配的空间



### volatile

1. volatile写之前的操作不会被编译重排序到volatile写之后
2. volatile读之后的操作不会被编译重排序到volatile读之前
3. 第一个是volatile写，第二个是volatile读，不能重排序