

# AQS#Condotion源码解析

## 注意事项

1. Condotion 前提是独占锁
2. await 或 signal 之前必须持有独占锁
3. 条件队列是单向链表（同步等待队列是双向链表）
4. 条件队列头节点有线程，同步队列头节点没有线程
5. 公平的，在条件队列里先进先出--signal 只会唤醒头节点
   (同步等待队列有公平和非公平，非公平时直接插队，不进入队列)

## Condotion#await() 流程

功能：等待，支持中断，当 interrupt() 时会报错

<iframe id="embed_dom" name="embed_dom" frameborder="0" style="display:block;width:525px; height:245px;" src="https://www.processon.com/embed/61b4c8d47d9c086a676818a0"></iframe>

### 代码解析

#### Condotion#await()

功能：等待，支持中断，当 interrupt() 时会报错
说明：步骤：

1. 加入条件队列
2. 释放锁
3. 挂起当前线程
4. 被signal() 唤醒并进入同步等待队列
5. 在同步等待队列里自旋获取锁

```java
public final void await() throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();//如果线程被中断，抛出中断异常
            Node node = addConditionWaiter(); //新建节点并加入条件队列
            int savedState = fullyRelease(node);//AQS,释放锁，并唤醒后继节点
            int interruptMode = 0;
            while (!isOnSyncQueue(node)) {//AQS,循环一直判断,是否又被加到同步队列中，signal或signalAll--唤醒线程并条件队列转移到同步队列
                LockSupport.park(this); //如果不在，阻塞线程直至被中断或者被其他线程唤醒
                //AQS，检查wait期间，是否有中断，0:没有中断；-1(THROW_IE):signal之前; 1(REINTERRUPT):signal之后
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)//0:没有中断；!=0: 有中断
                    break;  //如果被中断过跳出当前循环
            }
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)//acquireQueued 死循环自旋获取锁
                //如果线程被中断，并且中断的方式不是抛出异常，则设置中断后续的处理方式设置为REINTERRUPT(可重新中断)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters(); //遍历条件队列,删除条件队列中无效节点(非CONDITION状态)
            if (interruptMode != 0)
                //中断处理： 如果线程已经被中断，则根据之前获取的interruptMode的值来判断是继续中断还是抛出异常
                reportInterruptAfterWait(interruptMode);
        }
```

#### Condotion#addConditionWaiter()

功能：加入条件队列
说明：分为下面几个步骤：

1. 判断 waitStatus 是不是 CONDITION，如果不是，则遍历链表，删除状态不是CONDITION的节点
2. 将线程封装成节点
3. 加入条件队列末尾
   * 尾节点，头节点 = 当前节点
   * 非尾节点，在队列末尾加入当前节点（单向链表）

```java
        private Node addConditionWaiter() {
            Node t = lastWaiter;//对尾
            if (t != null && t.waitStatus != Node.CONDITION) { //状态不是CONDITION
                unlinkCancelledWaiters();//遍历链表，删除状态不是CONDITION的节点
                t = lastWaiter;
            }
            Node node = new Node(Thread.currentThread(), Node.CONDITION);
            if (t == null)//尾节点为 null,说明条件队列为空
                firstWaiter = node;
            else
                t.nextWaiter = node;//加到条件队列末尾
            lastWaiter = node;//重置尾节点
            return node;
        }
```

#### AQS#fullyRelease()

功能：释放锁，并唤醒后继节点
说明：

1. 支持锁的重入，savedState 为锁重入次数，要全部都释放
2. 释放失败，则抛异常
3. finally，释放失败或没有锁（没有锁却调用 await()），将节点标记为取消

```java
    final int fullyRelease(Node node) {
        boolean failed = true;
        try {
            int savedState = getState();
            if (release(savedState)) {//释放锁并唤醒后继节点
                failed = false;
                return savedState;
            } else {
                throw new IllegalMonitorStateException();//释放锁失败，则抛异常
            }
        } finally {
            if (failed)
                node.waitStatus = Node.CANCELLED;//没有持有锁(能进入条件队列)，就调用condition.await()
        }
    }

    public final boolean release(int arg) {
        if (tryRelease(arg)) {//yangyc 释放一次锁
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);//yangyc 唤醒后继结点
            return true;
        }
        return false;
    }
```

#### AQS#isOnSyncQueue()

功能：释放锁，并唤醒后继节点
说明：

1. 为了性能，在 isOnSyncQueue() 里两个 if 快速判断
2. 第一步没有结果，再遍历队列，看节点有没有在条件队列里面

```java
    final boolean isOnSyncQueue(Node node) {
        if (node.waitStatus == Node.CONDITION || node.prev == null)//状态是CONDITION或前驱节点为null,说明在条件队列中
            return false;
        if (node.next != null) //后继节点不为空，说明在同步对等队列中 // If has successor, it must be on queue
            return true;
        // 上面两个是快速判断，避免遍历链表，当快速判断失败时，再从尾开始遍历链表
        return findNodeFromTail(node);//从尾玩前找node，是不是在条件队列中
    }

    private boolean findNodeFromTail(Node node) {
        Node t = tail;
        for (;;) {
            if (t == node)
                return true;
            if (t == null)
                return false;
            t = t.prev;
        }
    }

```

#### Condotion#checkInterruptWhileWaiting()

功能：检查wait期间，是否有中断

说明：
背景：
         先判断节点有没有在同步队列里面
          如果没有在同步队列里面，即：在条件队列里面
          阻塞线程
          唤醒线程后，要检查阻塞期间，有没有发生中断 checkInterruptWhileWaiting()

interruptMode 状态说明：

* 0：没有中断，正常 signal 或者 signalAll，这时候节点在同步等待队列中，while 条件不成立，退出循环

* -1（THROW_IE）：signal之前，这时候要抛异常 THROW_IE

* 1（REINTERRUPT）：signal之后，这时候要进入同步等待队列中继续中断 REINTERRUPT

  

  下面几个步骤：

1.  判断有没有中断状态，没有返回 0
2. 有中断状态，判断中断时机（transferAfterCancelledWait）
   1. CAS waitStatus 成功，说明 signal 之前中断的，返回 true （因为signal方法会将waitStatus 为 0）
   2. 否则就是signal 之后中断的，返回 false
      * 返回 false 之前要保证节点在同步队列中，否则给其他线程让步，直到被加入到同步队列

```java
        private int checkInterruptWhileWaiting(Node node) {
            //检查wait期间，是否有中断，0:没有中断；-1(THROW_IE):signal之前; 1(REINTERRUPT):signal之后
            return Thread.interrupted() ?
                    (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) :
                    0;
        }

    final boolean transferAfterCancelledWait(Node node) {
        //背景： signal 方法会将状态设置为0
        if (compareAndSetWaitStatus(node, Node.CONDITION, 0)) {
            enq(node);//CAS成功，说明没有signal的调用，即：signal之前，加入同步队列
            return true;
        }
        //到这一步，说明signal之后
        while (!isOnSyncQueue(node))// 如果不在同步队列中，则给其他线程让步，直到被加入到同步队列
            Thread.yield();
        return false;
    }
```



#### Condotion#acquireQueued()

功能：死循环获取锁--独占模式
说明：在前面已经加入同步等待队列了（signal 或 signalAll 或 transferAfterCancelledWait）
            这边要 acquireQueued()，死循环去获取锁

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



#### Condotion#unlinkCancelledWaiters() 

功能：删除条件队列中被取消的节点
说明：截止到这一步，节点已经从条件队列转移到同步等待队列中，去排队等锁了
            当前节点如果不是尾节点，就清理一下条件队列----遍历条件队列,删除条件队列中无效节点(非CONDITION状态)

```java
        private void unlinkCancelledWaiters() {
            Node t = firstWaiter;
            Node trail = null;
            while (t != null) {
                Node next = t.nextWaiter;
                if (t.waitStatus != Node.CONDITION) {
                    t.nextWaiter = null;
                    if (trail == null)
                        firstWaiter = next;
                    else
                        trail.nextWaiter = next;
                    if (next == null)
                        lastWaiter = trail;
                }
                else
                    trail = t;
                t = next;
            }
        }
```



#### Condotion#reportInterruptAfterWait()

功能：处理中断
说明：中断状态有三种情况

* 0：不用处理
* 1（REINTERRUPT）：自我中断 selfInterrupt()
* -1（THROW_IE）：抛异常

```java
        private void reportInterruptAfterWait(int interruptMode)
                throws InterruptedException {
            if (interruptMode == THROW_IE)
                throw new InterruptedException();
            else if (interruptMode == REINTERRUPT)
                selfInterrupt();
        }
```



## Condotion#awaitUninterruptibly() 流程

#### 源码

功能：等待--不支持中断，interrupt() 时候不会报错
说明：挂起期间，如果发生中断，不响应，只记录状态，等待 signal 和 signalAll 将线程唤醒后，再将节点从等待队列转移到同步队列，获取锁只会再自我中断

```java
        //awaitUninterruptibly()        
        public final void awaitUninterruptibly() {
            Node node = addConditionWaiter();
            int savedState = fullyRelease(node);
            boolean interrupted = false;
            while (!isOnSyncQueue(node)) {
                LockSupport.park(this);
                if (Thread.interrupted()) //忽略中断，中断时只记录中断状态，后继续挂起
                    interrupted = true;
            }
            if (acquireQueued(node, savedState) || interrupted) //获取到锁返回时，再自我中断一下
                selfInterrupt();
        }
```



#### 与 Condotion#await 区别

1. Condotion##await：signal (signalAll) 或 中断，都可以唤醒，都可以退出阻塞
2. Condotion#awaitUninterruptibly：只有 signal (signalAll) ，才会退出阻塞

```java
        //awaitUninterruptibly()        
        public final void awaitUninterruptibly() {
           ...
            while (!isOnSyncQueue(node)) {
                LockSupport.park(this);
                if (Thread.interrupted()) //忽略中断，中断时只记录中断状态，后继续挂起
                    interrupted = true;
            }
            if (acquireQueued(node, savedState) || interrupted) //获取到锁返回时，再自我中断一下
                selfInterrupt();
        }

        //await()
        public final void await() throws InterruptedException {
            ...
            while (!isOnSyncQueue(node)) {
                LockSupport.park(this); 
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0) //挂起期间发生中断，会 break 退出
                    break;
            }
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }
```



### Condotion#awaitNanos(long nanosTimeout)

#### 源码

功能：等待--设置超时时间
说明：每次挂起之前，先判断有没有超时，如果超时要 break 退出，且 park 时要指定时间

```java
        public final long awaitNanos(long nanosTimeout)
                throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();
            Node node = addConditionWaiter();
            int savedState = fullyRelease(node);
            final long deadline = System.nanoTime() + nanosTimeout;
            int interruptMode = 0;
            while (!isOnSyncQueue(node)) {
                if (nanosTimeout <= 0L) { // 判断有没有超时
                    transferAfterCancelledWait(node);
                    break;
                }
                if (nanosTimeout >= spinForTimeoutThreshold)
                    LockSupport.parkNanos(this, nanosTimeout);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
                    nanosTimeout = deadline - System.nanoTime(); //计算剩余时间
            }
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null)
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
            return deadline - System.nanoTime();
        }
```

#### 与 Condotion#await 区别

1. Condotion##await：while 循环挂起线程，如果没有 signal (signalAll) 或 中断，不会退出循环，一直阻塞

2. Condotion#awaitNanos：为了防止一直阻塞，加了个超时时间，一段时间没有signal (signalAll) 或 中断，仍然要退出循环

   ```java
           //Condotion#awaitNanos
           public final long awaitNanos(long nanosTimeout)
                   throws InterruptedException {
               ...
               while (!isOnSyncQueue(node)) {
                   if (nanosTimeout <= 0L) { // 判断有没有超时
                       transferAfterCancelledWait(node);
                       break;
                   }
                   if (nanosTimeout >= spinForTimeoutThreshold)
                       LockSupport.parkNanos(this, nanosTimeout);
                   if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                       break;
                   nanosTimeout = deadline - System.nanoTime();//计算剩余时间
               }
               ...
           }
   
           //Condotion##await
           public final void await() throws InterruptedException {
               ...
               while (!isOnSyncQueue(node)) {
                   LockSupport.park(this); 
                   if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                       break;
               }
              ...
           }
   ```



## Condotion#await(long time, TimeUnit unit) 流程

与上面的 Condotion#awaitNanos 一样，只是超时时间多了一个单位



## Condotion#awaitUntil(long time, TimeUnit unit) 流程

#### 源码

功能：等待--设置截止日期
说明：每次挂起之前，先判断有没有超时(当前时间 - 截止时间)，如果超时要 break 退出，且 park 时要指定时间

```java
        public final boolean awaitUntil(Date deadline)
                throws InterruptedException {
            long abstime = deadline.getTime();
            if (Thread.interrupted())
                throw new InterruptedException();
            Node node = addConditionWaiter();
            int savedState = fullyRelease(node);
            boolean timedout = false;
            int interruptMode = 0;
            while (!isOnSyncQueue(node)) {
                if (System.currentTimeMillis() > abstime) {
                    timedout = transferAfterCancelledWait(node);
                    break;
                }
                LockSupport.parkUntil(this, abstime);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null)
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
            return !timedout;
        }
```

#### 与 Condotion#await 区别

1. Condotion##await：while 循环挂起线程，如果没有 signal (signalAll) 或 中断，不会退出循环，一直阻塞

2. Condotion#awaitNanos：为了防止一直阻塞，加了个超时时间，一段时间没有signal (signalAll) 或 中断，仍然要退出循环

3. Condotion#awaitUntil：同 awaitNanos，区别只是这边的超时时间判断：

   * awaitUntil：当前时间 - 截止时间
   * awaitNanos：while 每次循环都算出剩余时间

   ```java
           //Condotion#awaitUntil
           public final long awaitNanos(long nanosTimeout)
                   throws InterruptedException {
               ...
               while (!isOnSyncQueue(node)) {
                   if (System.currentTimeMillis() > abstime) {
                       timedout = transferAfterCancelledWait(node);
                       break;
                   }
                   LockSupport.parkUntil(this, abstime);
                   if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                       break;
               }
               ...
           }
   
           //Condotion##await
           public final void await() throws InterruptedException {
               ...
               while (!isOnSyncQueue(node)) {
                   LockSupport.park(this); 
                   if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                       break;
               }
              ...
           }
   ```

   



## Condotion#signal() 流程

说明：条件队列转移到同步等待队列

### 源码解析

#### Condotion#signal()

功能：条件队列转移到同步等待队列
说明：某个线程 await()，需要 signal 将其唤醒
            *条件队列是公平，所以只能唤醒第一个节点（同步等待队列可以公平和非公平，表诉不是很准确，不是队列公平，是不排队直接竞争来实现非公平）*,

```java
        public final void signal() {
            if (!isHeldExclusively())//yangyc 必须持有当前独占锁
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;
            if (first != null)
                doSignal(first);//唤醒条件队列中第一个节点
        }

        private void doSignal(Node first) {
            //while循环，如果当前节点迁移不成功，迁移下一个节点
            do {
                if ( (firstWaiter = first.nextWaiter) == null)
                    lastWaiter = null; //后面没有等待的节点，将尾节点置空
                first.nextWaiter = null;
            } while (!transferForSignal(first) && //将节点从条件队列转移到同步等待队列
                    (first = firstWaiter) != null);
        }
```



#### Condotion#transferForSignal()

功能：将节点从条件队列转移到同步等待队列
说明：非为下面几个步骤

1. enq()，死循环自旋加入同步阻塞队列
2. 唤醒线程

```java
    final boolean transferForSignal(Node node) {
        if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
            return false;//可能是被其他线程转移了
        Node p = enq(node);//死循环自旋加入同步阻塞队列
        int ws = p.waitStatus;
        if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
            LockSupport.unpark(node.thread);//唤醒线程
        return true;
    }
```



