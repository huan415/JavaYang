# ConcurrentHashMap

## HashMap 的线性安全

1. Hashtable
   synchronized 关键字加锁。锁住了整个 table，效率低

   ```java
   #Hashtable
   public synchronized V put(K key, V value) {}
   public synchronized V remove(Object key) {}
   public synchronized V get(Object key) {}
   ```

   

2. Collections
   和 Hashtable 差不多，同样的原因，锁住整张表，效率低下

   ```java
    Collections.synchronizedMap(new HashMap<>());
   ```

3. ConcurrentHashMap
   性能最好。JDK7 和 JDK8 在实现上不同。看下面对比



# JDK7 和 JDK8 的区别



| ConcurrentHashMap       | JDK1.7                                                       | JDK1.8                                                       |
| ----------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 锁的结构不同            | 基于分段锁实现Segment+HashEntry数组                          | synchronized+CAS+红黑树                                      |
| put()的执行流程有所不同 | 两次定位<br />1. 先对Segment进行定位，<br />2. 再对其内部的数组下标进行定位 | 只需要一次定位                                               |
| size的方法不一样        | 采用类似于乐观锁的机制，先是不加锁直接进行统计，最多执行三次，如果前后两次计算的结果一样，则直接返回。若超过了三次，则对每一个Segment进行加锁后再统计 | 会维护一个baseCount属性用来记录结点数量，每次进行put操作之后都会CAS自增baseCount |
| 结构                    |                                                              | 引入了红黑树                                                 |





## 存储结构

### JDK7

`ConcurrnetHashMap` 由很多个 `Segment`(继承ReentrantLock) 组合，而每一个 `Segment` 是一个类似于 HashMap 的结构，所以每一个 `HashMap` 的内部可以进行扩容。但是 `Segment` 的个数一旦**初始化就不能改变**，默认 `Segment` 的个数是 16 个，你也可以认为 `ConcurrentHashMap` 默认支持最多 16 个线程并发。

![java_basis_concurrentHashMap_structure_1](https://raw.githubusercontent.com/huan415/JavaYang/master/assets/java_basis_concurrentHashMap_structure_1.png)



### JDK8

**Node 数组 + 链表 / 红黑树**。当冲突链表达到一定长度时，链表会转换成红黑树。

![java_basis_concurrentHashMap_structure_2](https://raw.githubusercontent.com/huan415/JavaYang/master/assets/java_basis_concurrentHashMap_structure_2.png)







## put流程

### JDK7

1. 计算 key 的位置，获取指定位置的 Segment
2. 如果 Segment 不存在，则初始化这个 Segment
3. Segment.put 插入 key/value

为什么是线性安全的？
  由于Segment 继承ReenrantLock, 所以 Segment 内部可以很方便的获取锁，put流程

1. tryLock() 获取锁，获取不到使用 **`scanAndLockForPut`** 方法继续获取。
2. 计算 put 的数据要放入的 index 位置，然后获取这个位置上的 HashEntry 

### JDK8

1. 根据 key 计算出 hashcode 。
2. 判断是否需要进行初始化。
3. 即为当前 key 定位出的 Node，如果为空表示当前位置可以写入数据，利用 CAS 尝试写入，失败则自旋保证成功。
4. 如果当前位置的 `hashcode == MOVED == -1`,则需要进行扩容。
5. 如果都不满足，则利用 synchronized 锁写入数据。
6. 如果数量大于 `TREEIFY_THRESHOLD` 则要转换为红黑树。



## get流程

### JDK 7

两次hash，第一次hash查找segment，第二次hash查找元素

### JDK8

一次hash





## **ConcurrentHashMap 在 JDK 1.8 中，为什么要使用内置锁 synchronized 来代替重入锁 ReentrantLock？**

①、粒度降低了；

②、JVM 开发团队没有放弃 synchronized，而且基于 JVM 的 synchronized 优化空间更大，更加自然。

③、在大量的数据操作下，对于 JVM 的内存压力，基于 API 的 ReentrantLock 会开销更多的内存。

























参考：

1. [ConcurrentHashMap 1.7与1.8的区别总结](https://blog.csdn.net/SCUTJAY/article/details/104359923?share_token=ca53fac3-bfbc-4756-897a-ec6232b58ac3)
2. [ConcurrentHashMap 源码+底层数据结构分析](https://snailclimb.gitee.io/javaguide/#/docs/java/collection/concurrent-hash-map-source-code)