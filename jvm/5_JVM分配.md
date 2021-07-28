# JVM内存分配

# 常见知识点

1. Minor GC/Young GC：新生代的垃圾回收。频繁、速度快。

2. major GC/Full GC：老年代、年轻代、方法区的垃圾回收。速度比Young慢10被以上。

3. 大对象直接进入老年代。通过-XX:PretenureSizeThreshold=10000设置大小。

   只有Serial和ParNew两种垃圾回收器有效

   目的：年轻代内存比较小，容易触发Young GC，然后复制大对象到老年代，浪费性能

4.  每个对象都有一个年龄计数器，每经历一次Young GC年龄加1，达到年龄阈值之后复制到老年代

   年龄阈值: -XX:MaxTenuringThreshold     默认：15

5. Minor GC之后Survivor区依然放不下，会将存活下来的对象移到老年区

6. Eden比Survivor==8:1:1

   大部分对象都在Eden，Eden满了触发minor GC，回收Eden(99%以上的对象被收获)和Survivor,  未被回收的对象年龄加1，移到另一个Survivor

   -XX:+UseAdaptiveSizePolicy 修改比例，Eden尽可能的大



### 怎么判断一个对象可以被回收

1. 引用计数法、

   每个对象维护一个引用计数器，有一个对象引用就加1，失去一个引用就减1。0代表没有被引用。

   优点：简单、高效

   缺点：无法解决循环依赖问题

2. 可达性分析算法
   起点(GC Roots)向下搜索，收得到的是非垃圾对象，收不到的就是垃圾对象。

   GC Roots根节点包含：线程栈本地变量、静态变量、本地方法栈变量

3. 常见引用类型

   * 强引用：普通变量引用

   * 软引用：SoftReference包裹的泛型

     ​                GC回收后，内存还是不够，才会回收软引用

     ```java
                   SoftReference<Person> p1 = new SoftReference<Person>(new
                   Person());
     ```

     

   * 弱引用: WeakReference包裹的泛型
                  GC直接回收

     ```java
     WeakReference<Person> p1 = new WeakReference<Person>(new
     Person())
     ```

     

   * 虚引用：最弱的一种引用，基本不用

4. finalize（）
   相当于被判定为垃圾对象，仍有一个免死金牌的机会
   第一次：没有实现finalize方法，直接回收。
                  有实现finalize方法, 执行该方法（如果不想回收直接跟GC Roots引用，例如成员变量）
   第二次：垃圾回收

5. 方法区回收无用的类
   无用类判断条件：

   * 该类所有实例都被回收
   * 加载该类的ClassLoader已经被回收
   * 对应的java.lang.Class对应没有被引用，即无法反射到该类