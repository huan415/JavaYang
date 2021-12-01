# HashMap

## HashMap 数据结构

1. 数组+链表 （对比数组和链表的优缺点）
2. 数组+链表+红黑 （长度 > 8 && 数组总容量 >=64 转换成红黑树，长度<6退化成链表）



## put() 流程

![java_basis_hashmap_put_1](https://raw.githubusercontent.com/huan415/JavaYang/master/assets/java_basis_hashmap_put_1.png)

1. Hashmap 的 table 是懒加载，new 的时候并没有加载table，直到 put 之后才进行加载
2. hash 值与 （n-1）取 & 运算，得出 hash 桶位置 （table 数组下标位置）
3. 如果 hash 桶的位置没有值，则直接 newNode()
4. 如果 hash 桶的位置有值，即发生 Hash 碰撞，Hash 碰撞有如下几种情况
5. 情况一：判空  key 是否已存在（equals），如果相等，则覆盖 value
6. 情况二：判断哈希桶的首节点是否是红黑树节点，如果是，则在红黑树插入新节点
7. 情况三：哈希桶的首节点不是红黑树节点，则为链表节点
8. 开始遍历链表节点，判断 key 是否已存在（equals）
9. 如果 key 已存在，则替换value
10. 如果不存在则插入新节点(1.8尾插法，1.7为头插法)
11. 插入新节点后，判断链表长度是否 >= 8，如果大于则转换成红黑树
12. 最后，实际长度自加1，如果长度 > 临界值，则进行扩容

有注解的代码，详见：https://github.com/huan415/jdk/blob/master/src/java.base/share/classes/java/util/HashMap.java

## 怎么计算 hash ? 为什么要这样子算？

低16位 + 高 16位 进行 ^ 运算

原因：当数组长度很短时，只有低位的hashcode能参与运算，这样子hash碰撞几率高。而让低16位 ^ 高16位（高16位也参与运算），可以减少碰撞。

为什么是 ^，而不是 & 或者 | ？
因为：& 运算二进制更向1靠拢
            | 运算二进制更向0靠拢
            而 ^ 

```java
static final int hash(Object key) {
        int h;
        //@yangyc 先获取到key的hashCode，然后进行移位再进行异或运算，为什么这么复杂，不用想肯定是为了减少hash冲突
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```

## table 的容量如何确定？loadFactor 是什么？该容量如何变化？

1. capacity： table 大小，默认是16，最大 1<<30。
   刚开始new HashMap 时不会初始化table，在第一次 put 的时候才会初始化table
2. loadFactor：装载因子，默认0.75。用来确定是否需要动态扩容。
                        例如：table数组大小16，负载因子0.75，那么threshold=16*0.75=12，即：数组大小超过12时就要动态扩容（注意：不是等到整个数组满了才扩容）
3. 扩容：resize()；table 变为原来的2倍

## resize()

resize 分两步: 

1. 扩容： 新建一个Entry空数组，长度是原来的2倍
2. ReHash：遍历原来的 Entry 数组，将所有的 Entry 重新 hash 到新数组里面

### 扩容

1. 为什么是先插入元素在扩容
   因为：先比较是否是新增的数据，再判断增加数据后是否要扩容，这样比较会浪费时间，而先插入后扩容，就有可能在中途直接通过return返回了（本次put是覆盖操作，size不变不需要扩容），这样可以提高效率的。

### ReHash

- **[2.1]-索引定位**：1.7需要对原数组中的元素进行**重新hash定位**在新数组的位置，1.8采用更简单的判断逻辑，**位置不变或索引+旧容量大小**；

- **[2.2]-判断顺序**：**1.7先判断**是否需要扩容，再插入；**1.8先插入**，再判断是否需要扩容。


  下面是1.8的逻辑

   1. 由于table数组大小为2的n次幂，使得元素的位置要么是在原位置，要么是在原位置再移动2次幂的位置（即原位置+oldCap）

   2. 流程
      n为table的长度，图中上半部分表示扩容前的key1和key2两个Node的索引位置，图中下半部分表示扩容后key1和key2两个Node新的索引位置。
      ![java_basis_hashmap_rehash_1](https://raw.githubusercontent.com/huan415/JavaYang/master/assets/java_basis_hashmap_rehash_1.png)

      元素在重新计算hash之后，因为n变为2倍，那么n-1在高位多1bit，因此新的index就会发生这样的变化：
      ![java_basis_hashmap_rehash_2](https://raw.githubusercontent.com/huan415/JavaYang/master/assets/java_basis_hashmap_rehash_2.jpg)


## 为什么初始化大小是16？为什么是2的 n 次幂

```java
int index = (n - 1) & hash;
```

因为当数组大小等于 n 次幂时，n - 1的二进制低位全是1，高位全是0。

这样有两个好处：

1. hash & (n-1) 等价于 hash%n。
2. 使key分布更加均匀，低位&1，值和原来的hash一样。高位&0，值为0
3. 扩容时，元素的位置要么是在原位置，要么是在原位置再移动2次幂的位置

## JDK7和JDK8中HashMap不同之处

1. JDK7 HashMap接口：数组+链表
   JDK8 HashMap接口：数组+链表+红黑 （长度 > 8 && 数组总容量 >=64 转换成红黑树，长度<6退化成链表）
   因链表过长，导致查询效率变低（遍历链表），因此引入了红黑树

2. JDK7 hash碰撞时：头插法（链表的头部插入）

   ​                                  作者认为后来的值被查找的可能性更大一点，提升查找的效率。
   JDK8 hash碰撞时：尾插法（链表的尾部插入）
   ​                                   JDK7 头插法会出现死循环，优化成尾插法

   头插法会改变链表的顺序，即在插入数据的同时，其他线程在进行扩容，存在一定的记录出现环形链表----死循环。
   采用尾插法：扩容时顺序不会改变，保持原有的顺序，不会出现链表成环

 

| HashMap          | JDK1.7                             | JDK1.8                     |
| ---------------- | ---------------------------------- | -------------------------- |
| 底层结构         | 数组+链表                          | 数组+链表/红黑树           |
| 插扩顺序         | 先扩容再插值                       | 先插值再扩容               |
| 插入顺序         | 表头插入法                         | 表尾插入法                 |
| 扩容后的索引计算 | 扩容时需要重新计算哈希值和索引位置 | 不需要重新计算             |
| 并发问题         | 改变原有顺序，并发时引起链表闭环   | 保持原有顺序，不会出现闭环 |

## HashMap为什么引入红黑树而不是平衡树（AVL）

1. 为什么不使用二叉查找树
   因为二叉树在极端情况下像一条链表（只有左子树或只有右子树），跟链表一样，结构深，遍历查找慢

2. 为什么不使用平衡树（AVL）
   平衡树要求：每个节点的左子树和右子树的高度差<=1;
   这样子的要求太过苛刻，导致每次插入或者删除节点时，都会破坏平衡性，会触发左旋或者右旋；使性能变差

3. 红黑树为节点增加颜色，任何的不平衡性在三次旋转之内解决，维持平衡性的开销比AVL小得多。

   注意：

   1. 插入效率：红黑树 > AVL 树
   2. 查找效率：AVL 树 > 红黑树 （因为AVL树高度平衡）

## 负载因子为什么是0.75

0.75是对时间和空间的平衡

1. 加载因子越大，空间利用越充分，但是链表长度太长，查找效率低
2. 加载因子越小，空间利用不充分，虽然查找效率高，但是空间浪费严重

所以：内存利用要求高的，可以适当增加加载因子
            查询频繁的，可以适当减少加载因子

## 影响HashMap性能的因为

1. 负载因子：时间和空间的平衡
2. 边界值：设计resize()
3. hash值：均匀散列

## HashMap 的key

1. key要求条件
   必须重新hashcode 和equals方法
2. 如果只重写equals，不重写hashcode() 会怎样？
   默认的`equals()`与`hashCode()`比较的是内存地址，不重写会认为`hashCode()`不一样，就认为是不同对象，不需要再进行`equals()`比较，效率和正确性都会有问题。
3. HashMap 运行key/value 为null,  但是最多只有一个，为什么？
   因为 **key 为 null 会放在第一个桶**，而且链表的第一个节点





## 快速失败与安全失败

### 快速失败（fail-fast）

1. 通俗理解：集合在遍历的过程中，发现被修改过了，马上终止遍历并抛异常，没必要继续遍历，即快速响应失败
2. 它是 Java 的一种机制，迭代器在遍历一个集合的过程中，不允许修改（增、删、改），否则抛异常Concurrent Modification Exception
3. 原理：增删改的时候会修改modCount变量的值，而迭代器在hashNext()/next()遍历下一个元素之前，会校验modCount的值时候被修改

### 安全失败（fail-safe）

1. 多线程下并发使用，并发修改
2. java.util.concurrent包下的容器都是安全失败





参考：

1. [Java中hashMap容器的原理（为什么无符号移动16位，为什么要异或运算）](https://blog.csdn.net/weixin_43842753/article/details/105927912)
2. [这21 个刁钻的HashMap 面试题，我把阿里面试官吊打了](https://zhuanlan.zhihu.com/p/164531596)
3. [面试官：请说下对理解HashMap及LinkedHashMap的理解（八股文）](https://www.toutiao.com/i7024720360893334020/?tt_from=weixin&utm_campaign=client_share&wxshare_count=1&timestamp=1638256248&app=news_article&utm_source=weixin&utm_medium=toutiao_android&use_new_style=1&req_id=20211130151047010131136036053DCAB7&share_token=02fbaba1-6513-4d01-9cf7-c4f32da20734&group_id=7024720360893334020)
4. [【面试】HashMap常见面试题](https://blog.csdn.net/heavendan/article/details/121094540?share_token=cfce05c8-fce4-42eb-8127-2c7ba02ddd1c)