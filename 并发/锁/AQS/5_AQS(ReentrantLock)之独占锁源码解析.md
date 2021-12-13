# AQS(ReentrantLock)之独占锁源码解析

## 源码流程

###  各个方法概览

![AQS_AbstractQueuedSynchronizer](https://raw.githubusercontent.com/huan415/JavaYang/master/assets/AQS_AbstractQueuedSynchronizer.jpg)
![ReentrantLock](https://raw.githubusercontent.com/huan415/JavaYang/master/assets/ReentrantLock-16392027039361.png)

## ReentrantLock#lock() 流程(以非公平锁为例)

### 流程图

<iframe id="embed_dom" name="embed_dom" frameborder="0" style="display:block;width:525px; height:245px;" src="https://www.processon.com/embed/61b2a73863768956b5f56f01"></iframe>

### 要步骤

1. 加锁
2. 如果加锁失败，则进行锁重入
3. 如果锁重入失败，addWaiter 和 enq 加入同步等待队列
4. 加入队列后，acquireQueued 死循环去获取锁（循环期间有挂起线程，等待前驱节点唤醒）
5. 第四步线程挂起期间如果有中断产生，则标记中断状态 （parkAndCheckInterrupt 检查中断状态）
6. 获取锁之后，如果有中断产生，要进行自我中断

### 代码解析

#### ReentrantLock#lock()

功能：加锁，可以是 NonfairSync 或 FairSync，默认是 NonfairSync 
说明：加锁，不拿到锁誓不罢休，即使等待锁的过程中有中断，也不响应，等到获取锁之后再响应中断

```java
    public void lock() {
        sync.lock(); // sync 子类NonfairSync或FairSync
    }
```

#### NonfairSync#lock()

非公平锁加锁，分为两种情况：

1. 加锁成功，调用 AQS 的父类 AbstractOwnableSynchronizer 设置当前线程
2. 加锁失败，调用 AQS 的 acquire(1)，进行加锁失败的处理逻辑

```java
final void lock() {
            if (compareAndSetState(0, 1)) //CAS 加锁,非公平锁--一进来，不管三七二十一，先进行加锁
                setExclusiveOwnerThread(Thread.currentThread()); //加锁成功，调用AQS的父类AbstractOwnableSynchronizer设置当前线程
            else
                acquire(1); //加锁失败，调用AQS的acquire(1)，进行加锁失败的处理逻辑
        }
```

#### AQS#acquire(int arg)

功能：获取锁--独占模式（有中断时不响应，等到获取锁后再响应）
说明：获取锁失败有3个处理逻辑：

1. tryAcquire 此为 ReentrantLock 实现的方法，前面已经加锁失败，这时尝试锁重入
2. addWaiteraddWaiter 加入同步等待队列
3. acquireQueued 死循环获取锁--独占模式

```java
//功能：获取锁--独占模式（有中断时不响应，等到获取锁后再响应）
//tryAcquire 尝试获取锁--独占模式
//addWaiteraddWaiter 加入同步等待队列，队列为空或 CAS 失败，则调用 enq()
//acquireQueued 死循环获取锁--独占模式（有中断时不响应，等到获取锁后再响应）
public final void acquire(int arg) {
    if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
          selfInterrupt(); //获取锁之后进行自我中断，
}
```

#### NonfairSync#tryAcquire

功能：尝试获取锁--独占模式
说明：直接调用父类 Sync 类的方法

```java
protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
}
```

#### Sync#nonfairTryAcquire

功能：尝试获取锁--独占模式
说明：虽然在 ReentrantLock#lock 加锁失败了，是此时可能已经被释放了，所以此处在校验一次 state

    1. state == 0，说明锁已被释放， 进行加锁，返回加锁成功
    2. 锁的持有者是当前线程，锁重入，返回加锁成功
    3. 锁被持有、且不是当前线程，则返回加锁失败

```java
final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread(); // 当前线程
            int c = getState(); //AQS 获取state
            if (c == 0) { //校验 state，看锁是否被释放
                if (compareAndSetState(0, acquires)) { //AQS,加锁--CAS 修改 state
                    setExclusiveOwnerThread(current); //AQS, 设置独占锁的线程为当前线程
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) { //AQS的父类，判断持有锁的线程是不是当前线程，如果是进行锁重入
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);//AQS,锁重入--CAS 修改 state
                return true;
            }
            return false; //锁被持有、且不是当前线程，则返回加锁失败
        }
```

#### AQS#addWaiter

功能：加锁失败，加入同步等待队列
说明：两种情况：

1. 队列里面有节点正在排队，当前节点加进队列末尾
2. 队列为空或CAS失败，则调用enq()

```java
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode); //将当前线程构建成Node节点
        Node pred = tail;
        if (pred != null) { //尾节点!=null,说明队列里面有节点正在排队，当前节点加进队列末尾
            node.prev = pred;
            if (compareAndSetTail(pred, node)) { //CAS 修改尾节点
                pred.next = node;
                return node;
            }
        }
        enq(node); //走到这一步，说明队列为空，或者 CAS 加入尾节点失败。enq 进行死循环加锁
        return node;
    }
```

#### AQS#enq

功能：则利用 enq 死循环加入队列末尾
说明：队列为空或CAS加入队列末尾失败，分为两种情况：

1. 队列为空，先走 if 初始化队列（不退出循环），再走 else 加入队列末尾
2.  //队列里面有节点正在排队，CAS 将当前节点加进队列末尾

```java
    private Node enq(final Node node) {
        for (;;) { //死循环
            Node t = tail;
            if (t == null) { // Must initialize   队列初始化, 当队列为空先走if进行初始化(不退出循环)，再走else死循环CAS加入队列末尾
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else { //队列里面有节点正在排队，当前节点加进队列末尾
                node.prev = t;
                if (compareAndSetTail(t, node)) { //CAS加入队列末尾
                    t.next = node;
                    return t;
                }
            }
        }
    }
```

#### AQS#acquireQueued

功能：死循环获取锁--独占模式（有中断时不响应，等到获取锁后再响应）
说明：死循环去获取锁，有3种情况：

1. 获取锁成功，返回中断状态（修改 failed，避免走cancelAcquire()）
2. 线程挂起期间，如果有中断，只修改中断状态interrupted（不响应中断，等待后续处理）-----这一步类比一下doAcquireInterruptibly()
3. 可能出现了异常，退出了循环，此时节点取消等待，队列中删除节点

```java
    // acquireQueued：不断自旋尝试获取同步状态(锁)，获取不成功，则找安全点休息
    // shouldParkAfterFailedAcquire：判断当前线程是否应该被阻塞
    // parkAndCheckInterrupt：休眠线程并返回中断状态,返回中断状态
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true; //获取锁是否是失败
        try {
            boolean interrupted = false; //中断状态
            for (;;) {
                final Node p = node.predecessor();  //找到前驱结点
                if (p == head && tryAcquire(arg)) { //前驱节点是头节点，且加锁成功（）
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted; // 返回中断状态，无需中断
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                        parkAndCheckInterrupt())
                    interrupted = true; //需要中断，且parkAndCheckInterrupt返回的状态是中断，则修改中断状态
            }
        } finally {
            if (failed)
                cancelAcquire(node); //上面是死循环获取锁，正常不会走到这一步，如果走到这一步且failed=true,可能出现了异常，则节点取消等待，队列中删除节点
        }
    }
```

#### AQS#shouldParkAfterFailedAcquire

功能：判断当前线程是否应该被阻塞
说明：有3种情况：

1. SIGNAL，可以被唤醒，则返回 true，进行阻塞
2. 异常无用节点（取消状态）
3. 无用节点被移除之后，剩下的就是有用的节点，要CAS将节点状态改为SIGNAL，因为只有节点状态为SIGNAL才能被唤醒

```java
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL) //可以被唤醒,则应该进行阻塞
            return true;
        if (ws > 0) { // 前驱节点是取消状态(等待超时或者被中断)，则移除同步队列
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0); //健康检查，踢除不监控的（pred.waitStatus > 0）的节点
            pred.next = node;
        } else { //前驱节点waitStatus为 0 or PROPAGATE状态时. 将其设置为SIGNAL状态，然后当前结点才可以可以被安全地park
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
```

#### AQS#parkAndCheckInterrupt

功能：休眠线程并返回中断状态,返回中断状态
说明：挂起线程，当被唤醒之后才会 return，唤醒情况有两种

1. 前驱节点释放同步状态时，唤醒了该线程
2. 当前线程被打断导致唤醒

```java
    private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this); //阻塞该线程，至此该线程进入等待状态，等着unpark和interrupt叫醒他
        //上面一步线程已经中断了，接下来的唤醒之后，返回该线程是否在中断状态， 并会清除中断记号。
        return Thread.interrupted();
    }
```

#### AQS

功能：节点移除队列，取消等待
说明：死循环获取锁时抛异常，则cancelAcquire

```java
   private void cancelAcquire(Node node) {
        if (node == null)
            return;
        node.thread = null; //将当前节点封装的线程设置为NULL
        Node pred = node.prev;
        // 跳过前面已经移除的节点
        while (pred.waitStatus > 0)
            node.prev = pred = pred.prev;
        Node predNext = pred.next; //获取前驱节点的后继节点 (因为上一步有跳过CANCELLED的节点，前驱节点的后继节点也可能不是自己)
        node.waitStatus = Node.CANCELLED; //将节点设置为CANCELLED
        if (node == tail && compareAndSetTail(node, pred)) { //如果当前节点是队尾，则直接移除
            compareAndSetNext(pred, predNext, null); //CAS 设置 pred 的下一个节点为空( null )，表示是队尾。
        } else { //说明node不是队尾 或者 CAS 移除失败
            int ws;
            //前驱节点不是头节点 + 前驱节点的状态为SIGNAL(不是SIGNAL就设置为SIGNAL) + 前驱节点的线程不为NULL
            if (pred != head &&
                    ((ws = pred.waitStatus) == Node.SIGNAL ||
                            (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
                    pred.thread != null) {
                Node next = node.next; //后继节点
                if (next != null && next.waitStatus <= 0) //如果后继节点不是CANCELLED
                    compareAndSetNext(pred, predNext, next); //将前驱节点的后继指针指向当前节点的后继节点
            } else {
                unparkSuccessor(node);//如果是头节点，就唤醒线程
            }
            node.next = node; // help GC
        }
    }
```

### FairSync#lock() 与 NonfairSync#lock() 的区别

1. lock() 方法

   * NonfairSync 不管三七二十一，先 CAS 加锁再 acquire(1)

   * FairSync 直接  acquire(1)

     ```java
     //NonfairSync 
     final void lock() {
         if (compareAndSetState(0, 1)) //CAS 加锁
              setExclusiveOwnerThread(Thread.currentThread()); //加锁成功，调用AQS的父类AbstractOwnableSynchronizer设置当前线程
         else
              acquire(1); //加锁失败，调用AQS的acquire(1)，进行加锁失败的处理逻辑
     }
     //FairSync 
     final void lock() {
         acquire(1);
     }
     ```

2. tryAcquire() 方法

   1. NonfairSync 直接 CAS 加锁

   2. 先 hasQueuedPredecessors() 看看有没有节点在排队，如果没有再加锁

      ```java
      // FairSync   hasQueuedPredecessors()
      protected final boolean tryAcquire(int acquires) {
             ...
             if (!hasQueuedPredecessors() && compareAndSetState(0, acquires)) {
                       setExclusiveOwnerThread(current);
                       return true;
             }
            ...
      }
      
      // NonfairSync
      final boolean nonfairTryAcquire(int acquires) {
         ...
          if (compareAndSetState(0, acquires)) {
                 setExclusiveOwnerThread(current);
                 return true;
         }
         ...
      }
      ```

      

### ReentrantLock#lockInterruptibly()

#### 源码

功能：加锁--未获取锁也可以响应中断
说明：未获取锁也可以响应中断

```java
    public final void acquireInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted()) //前线程是否被中断
            throw new InterruptedException(); //如果已经被中断了，那么直接抛出异常
        if (!tryAcquire(arg)) // 尝试加锁
            doAcquireInterruptibly(arg); //死循环获取锁，与 acquireQueued() 区别在于是否响应中断
    }
```

#### 与 ReentrantLock#lock() 区别

主要是未获取到锁时（死循环等待锁的过程中），对于中断的处理逻辑不同

1. acquireQueued                         当 parkAndCheckInterrupt() 返回中断时，只修改中断状态 (等获取到锁再处理)

2. doAcquireInterruptibly(arg)    当 parkAndCheckInterrupt() 返回中断时，立即抛异常 (不用等到获取锁)

   ```java
   //acquireQueued 
   final boolean acquireQueued(final Node node, int arg) {
      ...
      if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt())
          interrupted = true;
      }
      ...
   }
   
   //doAcquireInterruptibly(arg) 
   private void doAcquireInterruptibly(int arg)
               throws InterruptedException {
      ...
      if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt())
          throw new InterruptedException();
      }
      ...
   }
   ```

   ### ReentrantLock#tryLock()

   #### 源码

   功能：尝试加锁，无论成功与失败，都会立即返回
   说明：立即返回，不会进入同步等待队列

   #### 与 ReentrantLock#lock() 区别

   主要区别是加锁失败之后的处理逻辑不同

   1. ReentrantLock#lock()：NonfairSync#lock() ==》AQS#acquire() （里面除了tryAcquire还有addWaiter和acquireQueued） ==》NonfairSync#tryAcquire() ==》Sync#nonfairTryAcquire()
      即：nonfairTryAcquire 失败之后，还会进行addWaiter和acquireQueued-----进入同步等待队列

   2. ReentrantLock#tryLock()：ReentrantLock#tryLock() == 》Sync#nonfairTryAcquire()
      tryLock 没有调用AQS#acquire()，即加锁失败即结束，不会进入同步等待队列

      ```java
      // ReentrantLock#lock()  tryAcquire(nonfairTryAcquire)失败 ==》 addWaiter和acquireQueued
      public final void acquire(int arg) {
                  if (!tryAcquire(arg) &&
                          acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
                      selfInterrupt(); //获取锁之后进行自我中断，
      }
      //ReentrantLock#tryLock()
      public boolean tryLock() {
              return sync.nonfairTryAcquire(1);
      }
      ```

   

   ### ReentrantLock#tryLock(long timeout, TimeUnit unit)

   #### 源码

   功能：尝试加锁，有超时时间，timeout时间内仍未得到锁，则返回 false
   说明：

   ```java
       public boolean tryLock(long timeout, TimeUnit unit)
               throws InterruptedException {
           return sync.tryAcquireNanos(1, unit.toNanos(timeout));
       }
       public final boolean tryAcquireNanos(int arg, long nanosTimeout)
               throws InterruptedException {
           if (Thread.interrupted())
               throw new InterruptedException();
           return tryAcquire(arg) ||
                   doAcquireNanos(arg, nanosTimeout);
       }
       private boolean doAcquireNanos(int arg, long nanosTimeout)
               throws InterruptedException {
           if (nanosTimeout <= 0L)
               return false;
           final long deadline = System.nanoTime() + nanosTimeout;
           final Node node = addWaiter(Node.EXCLUSIVE);  //加入队列
           boolean failed = true;
           try {
               for (;;) {
                   final Node p = node.predecessor();
                   if (p == head && tryAcquire(arg)) {
                       setHead(node);
                       p.next = null; // help GC
                       failed = false;
                       return true;
                   }
                   nanosTimeout = deadline - System.nanoTime();
                   if (nanosTimeout <= 0L)
                       return false;//超时直接返回获取失败
                   if (shouldParkAfterFailedAcquire(p, node) &&
                           nanosTimeout > spinForTimeoutThreshold)
                       LockSupport.parkNanos(this, nanosTimeout);//阻塞指定时长，超时则线程自动被唤醒
                   if (Thread.interrupted())//当前线程中断状态
                       throw new InterruptedException();
               }
           } finally {
               if (failed)
                   cancelAcquire(node);
           }
       }
   ```

   #### 与 ReentrantLock#lock() 区别

   主要区别是加锁失败，死循环获取锁中的逻辑不同（是否有超时时间）：

   1. ReentrantLock#lock()：tryAcquire 失败 ==》addWaiter 加入队列 ==》acquireQueued 死循环获取锁
   2. ReentrantLock#tryLock(long timeout, TimeUnit unit)：tryAcquire 失败 ==》addWaiter 加入队列 ==》doAcquireNanos 死循环获取锁（每次循环都会判断超时时间 + LockSupport.parkNanos() 指定阻塞时长）

   

   

