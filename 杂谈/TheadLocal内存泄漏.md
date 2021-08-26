# TheadLocal内存泄漏

## 类关系

1. Thread里有一个ThreadLocalMap属性
   一个线程可能存储了多个共享变量，即多个TheadLocal实例，所以threadLocals的key是TheadLocal实例

   ```java
   public
   class Thread implements Runnable {
      ThreadLocal.ThreadLocalMap threadLocals = null;
   }
   ```

2. TheadLocal#set流程

   ```java
   //示例
   public ThreadLocal<String> threadLocal = new ThreadLocal<>();
   threadLocal.set("huan415");
   threadLocal.get();
   
   //ThreadLocal#set：获取当前线程，从当前线程种获ThreadLocalMap
   public void set(T value) {
           Thread t = Thread.currentThread();
           ThreadLocalMap map = getMap(t);
           if (map != null)
               map.set(this, value);
           else
               createMap(t, value);
       }
   
   //Thread#getMap: 获取当前线程的ThreadLocalMap属性
   ThreadLocalMap getMap(Thread t) {
       return t.threadLocals;
   }
   ```

   

3. TheadLocal#get流程

   ```java
   //ThreadLocal#set：获取当前线程，从当前线程种获ThreadLocalMap
   public T get() {
           Thread t = Thread.currentThread();
           ThreadLocalMap map = getMap(t);
           if (map != null) {
               ThreadLocalMap.Entry e = map.getEntry(this);
               if (e != null) {
                   @SuppressWarnings("unchecked")
                   T result = (T)e.value;
                   return result;
               }
           }
           return setInitialValue();
       }
   ```

   

4. 这样的涉及很巧妙

   ```java
   这样的涉及很巧秒，对使用者很方便，学习一下
   map结构：key.set(value) ==>  实际是有一个第三方保存一个map对象, set时，获取第三方map,当前对象当key，值当value   ==>   使用时比较方便，不用 put(key,value)
           key.get()       ==>  实际是有一个第三方保存一个map对象, get时，获取第三方map,当前对象当key，直接get(key)   ==>   使用时比较方便，不用 get(key)
   ```



## 内存溢出

* 由上可知，ThreadLocalMap是存储在Thread中的，线程不死，ThreadLocalMap就不灭。其中ThreadLocalMap中的key是TheadLocal实例，弱引用，gc时可以被回收，但是value无法被回收。



## 内存泄漏

1. ThreadLocal内存泄漏根源：ThreadLocal的生命周期和Thread一样长，线程不死，ThreadLocalMap就不灭。
2. key指向ThreadLocal实例时为什么是弱引用
   * key为强引用
     key为强引用，没办法解决内存泄漏的问题（看上一点ThreadLocal内存泄漏根源）。反而会加快内存溢出，因为如果key是强引用，threadLocal被ThreadLocalMap强引用。
   * key为弱引用
     key为弱引用有一个好处，当threadLocal不被栈引用，在垃圾回收时可以被回收，不会因为ThreadLocalMap的强引用而不被回收。



## 四大引用

1. 强引用
   永远不会被回收
2. 弱引用（WeakReference）
   无论内存是否足够，内存回收时必被回收
3. 软引用（SoftReference）
   内存不够时，才会被回收
4. 虚引用



## 解决办法

每次使用完ThreadLocal，建议remove()方法，清除数据

