# ThreadLocal

## 区别

|                          | 用途               | 原理                                                         | 位置          |
| ------------------------ | ------------------ | ------------------------------------------------------------ | ------------- |
| ThreadLocal              | 线程内共享变量     | ThreadLocal.ThreadLocalMap，<br />Map 结构，key 是当前线程，以此实现同一个线程共享变量 | Thread 的属性 |
| InherbritableThreadLocal | 父子线程间共享变量 | new Tread 时，构造方法 =》init方法：父线程 inheritableThreadLocals不为空，<br />则复制的子线程 | Thread 的属性 |
| TransmittableThreadLocal | 线程池贡献变量     |                                                              |               |

注意：
        * InherbritableThreadLocal 继承 ThreadLocal，实现父子线程间共享变量
        * TransmittableThreadLocal 继承 TransmittableThreadLocal， 实现线程间共享变量



## ThreadLocal 原理

详见：TheadLocal内存泄漏

## InherbritableThreadLocal 继承 ThreadLocal，实现父子线程间共享变量

1. 示例

   ```java
   public class TestThreadLocal {
       public static void main(String[] args) {
           ThreadLocal<String> threadLocal = new ThreadLocal<String>(); //线程内共享变量
           ThreadLocal<String> inheritableThreadLocal = new InheritableThreadLocal<>();//父子线程间共享变量
           threadLocal.set("hello I'm threadLocal");
           inheritableThreadLocal.set("hello I'm inheritableThreadLocal");
   
           Thread thread = new Thread(new Runnable() {
               @Override
               public void run() {
                   System.out.println(Thread.currentThread().getName() + " say:" + threadLocal.get()); //ThreadLocal, 子线程无法获取父线程的变量
                   System.out.println(Thread.currentThread().getName() + " say:" + inheritableThreadLocal.get());//InheritableThreadLocal, 子线程可以获取父线程的变量
               }
           });
           thread.start();
           try {
               thread.join();
           } catch (InterruptedException e) {
               e.printStackTrace();
           }
           System.out.println(Thread.currentThread().getName() + " say:" + threadLocal.get()); //最后证明一下：变量还在，没有丢
           System.out.println(Thread.currentThread().getName() + " say:" + inheritableThreadLocal.get());//最后证明一下：变量还在，没有丢
       }
   }
   
   
   //结果：threadLocal 不能获取父线程的变量，inheritableThreadLocal 可以获取父线程的变量
   Thread-0 say:null
   Thread-0 say:hello I'm inheritableThreadLocal
   main say:hello I'm threadLocal
   main say:hello I'm inheritableThreadLocal
   ```
   
   
   
2. 创建子线程，在初始化的时候，如果inheritableThreadLocals 不为空，则将其复制到子线程里

   ```java
   //构造方法  
   public Thread() {
       init(null, null, "Thread-" + nextThreadNum(), 0);
   }
   
   private void init(ThreadGroup g, Runnable target, String name,
                         long stackSize, AccessControlContext acc,
                         boolean inheritThreadLocals) {
      ...  
           if (inheritThreadLocals && parent.inheritableThreadLocals != null)
               //父线程的 inheritableThreadLocals 不为空时，创建一个 ThreadLocal
               this.inheritableThreadLocals = ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
      ...
   }
   //遍历父线程的 ThreadLocalMap 复制到 子线程	
   private ThreadLocalMap(ThreadLocalMap parentMap) {
               Entry[] parentTable = parentMap.table;
               int len = parentTable.length;
               setThreshold(len);
               table = new Entry[len];
   
               for (int j = 0; j < len; j++) {
                   Entry e = parentTable[j];
                   if (e != null) {
                       @SuppressWarnings("unchecked")
                       ThreadLocal<Object> key = (ThreadLocal<Object>) e.get();
                       if (key != null) {
                           Object value = key.childValue(e.value);
                           Entry c = new Entry(key, value);
                           int h = key.threadLocalHashCode & (len - 1);
                           while (table[h] != null)
                               h = nextIndex(h, len);
                           table[h] = c;
                           size++;
                       }
                   }
               }
           }
   ```

   

3. 那为什么用 InherbritableThreadLocal ，而不是ThreadLocal 呢
   InheritableThreadLocal 继承 ThreadLocal，并重写了三个方法

   ```java
   public class InheritableThreadLocal<T> extends ThreadLocal<T> {	
       protected T childValue(T parentValue) {
           return parentValue; //这个没看懂是什么意思，只是直接返回。父类 ThreadLocal 是抛异常
       }
       ThreadLocalMap getMap(Thread t) {
          //这个很重要，InheritableThreadLocal 返回的是 inheritableThreadLocals，而 ThreadLocal 返回的是 threadLocals
          return t.inheritableThreadLocals; 
       }
       void createMap(Thread t, T firstValue) {
           //创建 ThreadLocalMap，InheritableThreadLocal 赋值给 inheritableThreadLocals
           //创建 ThreadLocalMap，ThreadLocal 赋值给 threadLocals
           t.inheritableThreadLocals = new ThreadLocalMap(this, firstValue);
       }
   }
   ```




## TransmittableThreadLocal 继承 TransmittableThreadLocal， 实现线程间共享变量

1. 示例

   ```java
   public class TestThreadLocal2 {
       public static void main(String[] args) {
           //==========================inheritableThreadLocal 线程池间没办法共享变量============================================
           ExecutorService executorService = Executors.newFixedThreadPool(1);
           ThreadLocal<String> inheritableThreadLocal = new InheritableThreadLocal<>();//父子线程间共享变量
           inheritableThreadLocal.set("hello I'm inheritableThreadLocal ==========  1");
           executorService.execute(() -> {
               //inheritableThreadLocal, 刚开始线程池为空，创建子线程时，有走 init() 方法，所以能获取到父线程的变量
               System.out.println(Thread.currentThread().getName() + " say:" + inheritableThreadLocal.get());
           });
           //参数改变
           inheritableThreadLocal.set("hello I'm inheritableThreadLocal ==========  2");
           executorService.execute(() -> {
               //inheritableThreadLocal, 父线程变量改变，线程池里面的线程感知不到，因为线程池复用线程，不会重新 new 线程（不会走 init() 方法）
               System.out.println(Thread.currentThread().getName() + " say:" + inheritableThreadLocal.get());
           });
   
   
           //==========================transmittableThreadLocal 线程池间可以共享变量============================================
           ExecutorService executorServiceTransmittable = TtlExecutors.getTtlExecutorService(Executors.newFixedThreadPool(1));
           TransmittableThreadLocal<String> transmittableThreadLocal = new TransmittableThreadLocal<>(); //线程池共享变量
           transmittableThreadLocal.set("hello I'm transmittableThreadLocal ==========  1");
           executorServiceTransmittable.execute(() -> {
               System.out.println(Thread.currentThread().getName() + " say:" + transmittableThreadLocal.get());
           });
           transmittableThreadLocal.set("hello I'm transmittableThreadLocal ==========  2");
           executorServiceTransmittable.execute(() -> {
               System.out.println(Thread.currentThread().getName() + " say:" + transmittableThreadLocal.get());
           });
   
       }
   }
   ```

   



参考：
   * [InheritableThreadLocal类原理简介使用 父子线程传递数据详解 多线程中篇（十八）](https://www.cnblogs.com/noteless/p/10448283.html)
        * [ThreadLocal系列（三）-TransmittableThreadLocal的使用及原理解析](https://www.cnblogs.com/hama1993/p/10409740.html)



