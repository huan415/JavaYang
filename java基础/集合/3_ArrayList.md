# ArrayList

## 数据结构

1. ArrayList：见名知意，Array，底层是数组（动态数组）。
   ArrayList 与 java 中 Array 的区别是：ArrayList 的数组是动态数组，可以动态扩容。
2. ArrayList 底层是数组，与之相对应的是LinkedList，底层是双向链表。
3. ArrayList 是线程不安全的，与之对应的是 Vector

## 特点

1. 基于数组实现，动态数组，动态扩容
2. 对比 LinkedList，查询快，增删慢，且不需要维护链表结构
3. 对比 Vector，非线程安全，效率高
4. 有序，且可以为null



## 实现的接口

三个标识型接口，没有任何方法

1. Randomccess：支持快速随机访问
2. Cloneable：支持克隆，实现了 clone 方法
3. Serializable：支持序列化

```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
{}
```



## 扩容机制

1. 扩容是1.5倍，新建一个空数组elementData，然后把元素copy过去



## 零散的知识点

1. 数组初始化时机
   无参构造方法可以看出，new Arraylist() 初始化的是一个空数组，当真正添加元素的时候才分配容量。
   且数组大小为10