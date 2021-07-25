# Volatile

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