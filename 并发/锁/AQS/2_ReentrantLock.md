# ReentrantLock

## 源码

![ReentrantLock](https://raw.githubusercontent.com/huan415/JavaYang/master/assets/ReentrantLock.png)



## 子类

 如下图所示：
       ReentrantLock有三个子类：Sync、NofairSync、FairSync
                                                         其中 Sync 继承AbstractQueuedSynchronizer
                                                          NofairSync 和 FairSync 继承 Sync
       ReentrantLock 有一个属性：Sync （实则为NofairSync或FairSync的实现类）
       当调用ReentrantLock.lock 或 tryLock 时，实际调用Sync 里的lock 或 tryLock 

![image-20211202103613834](https://raw.githubusercontent.com/huan415/JavaYang/master/assets/ReentrantLock_2.png)





```java
package java.util.concurrent.locks;
import java.util.concurrent.TimeUnit;
import java.util.Collection;
public class ReentrantLock implements Lock, java.io.Serializable {
    private static final long serialVersionUID = 7373984872572414699L;
    //yangyc 内部调用AQS的动作，都基于该成员属性实现
    private final Sync sync;

     //===================================子类：Sync 继承 AbstractQueuedSynchronizer==========================================
    abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = -5179523762034025860L;
        //yangyc 加锁的具体行为由子类实现
        abstract void lock();
        //yangyc 尝试获取非公平锁
        final boolean nonfairTryAcquire(int acquires) {...}
        //yangyc 释放锁
        protected final boolean tryRelease(int releases) {...}
    }

     //===================================子类：非公平锁==========================================
    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = 7316153563782823691L;
        // 加锁行为
        final void lock() {...}
        // 加锁行为
        protected final boolean tryAcquire(int acquires) {...}
    }

    //===================================子类：公平锁==========================================
    static final class FairSync extends Sync {
        private static final long serialVersionUID = -3000897897090466540L;
        // 加锁行为
        final void lock() {...}
         // 加锁行为
         protected final boolean tryAcquire(int acquires) { }
    }

     //=================================== 默认构造函数，创建非公平锁==========================================
    public ReentrantLock() {
        sync = new NonfairSync();
    }
    
    //===================================ReentrantLock 的方法：实则调用子类==========================================
    
    //yangyc 加锁-----可能是公平锁、非公平锁
    public void lock() {
        sync.lock();
    }
    //yangyc 尝试加锁----已非公平锁的形式获取锁
    public boolean tryLock() {
        return sync.nonfairTryAcquire(1);
    }
  
}

```





## 公平锁与非公平锁

1. lock 方法的区别
   最大的区别在于，非公平锁一上来就去加锁，不会去判断同步队列(CLH队列)中 *--- hasQueuedPredecessors()*，是否有排队等待加锁的节点，上来直接加锁（判断state是否为0,CAS修改state为1） 

   ```java
   // FairSync   hasQueuedPredecessors()
    final void lock() {
          acquire(1);
   }
   protected final boolean tryAcquire(int acquires) {
          ...
          if (!hasQueuedPredecessors() && compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
          }
         ...
   }
   
   // NonfairSync
    final void lock() {
         if (compareAndSetState(0, 1))
             setExclusiveOwnerThread(Thread.currentThread());
         else
             acquire(1);
   }
   final boolean nonfairTryAcquire(int acquires) {
      ...
       if (compareAndSetState(0, acquires)) {
              setExclusiveOwnerThread(current);
              return true;
      }
      ...
   }
   ```

   