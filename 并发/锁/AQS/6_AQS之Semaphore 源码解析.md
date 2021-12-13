# AQS 之 Semaphore 源码解析

## 各个方法概览

![AQS_Semaphore](https://raw.githubusercontent.com/huan415/JavaYang/master/assets/AQS_Semaphore.jpg)

## 类比独占锁

共享锁与独占锁，整个流程特别相似，我在这边就不一 一详细解释，而只是对比其中的不同点

| 独占锁                                      | 共享锁                                            |
| ------------------------------------------- | ------------------------------------------------- |
| tryAcquire(int arg)                         | tryAcquireShared(int arg)                         |
| tryAcquireNanos(int arg, long nanosTimeout) | tryAcquireSharedNanos(int arg, long nanosTimeout) |
| acquire(int arg)                            | acquireShared(int arg)                            |
| acquireQueued(final Node node, int arg)     | doAcquireShared(int arg)                          |
| acquireInterruptibly(int arg)               | acquireSharedInterruptibly(int arg)               |
| doAcquireInterruptibly(int arg)             | doAcquireSharedInterruptibly(int arg)             |
| doAcquireNanos(int arg, long nanosTimeout)  | doAcquireSharedNanos(int arg, long nanosTimeout)  |
| release(int arg)                            | releaseShared(int arg)                            |
| tryRelease(int arg)                         | tryReleaseShared(int arg)                         |
| -                                           | doReleaseShared()                                 |



## 获取锁

以独占锁 ReentrantLock#lock 和 Semaphore#acquireUninterruptibly 为例

### 差异

#### 相同之处

1. 不拿到锁誓不罢休
2. 不响应中断，等拿到做之后才处理中断

#### 差异之处

1. ReentrantLock#lock ==》NonfairSync#lock ==》AQS#acquire ==》 NonfairSync#tryAcquire ==》AQS#addWaiter ==》AQS#acquireQueued

2. Semaphore#acquireUninterruptibly ==》AQS#acquireShared ==》NonfairSync#tryAcquireShared ==》AQS#doAcquireShared（里面有 addWaiter 和acquireQueued）

   说明：

   1. 共享锁的 doAcquireShared 包含了独占锁的 addWaiter、acquireQueued、selfInterrupt 三步
   2. 自旋等待锁的时候，共享锁 trAcquireShared 返回剩余资源的个数，独占锁 tryAcquire 返回 true/false
   3. 设置头节点不同，共享锁 setHeadAndPropagate（共享：拿到锁之后，也要唤醒后继节点来拿锁），独占锁 setHead

```java
    //独占锁 acquire 
    public final void acquire(int arg) {
         if (!tryAcquire(arg) &&
                   acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
           selfInterrupt();
    }
    //acquireShared 
    public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg); //里面包含了 addWaiter、acquireQueued、selfInterrupt
    }
    private void doAcquireShared(int arg) {
        final Node node = addWaiter(Node.SHARED);//这一步跟独占锁的 addWaiter 一样
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) { //这一步跟独占锁的 acquireQueued 一样
                final Node p = node.predecessor();
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        if (interrupted)
                            selfInterrupt(); //这一步跟独占锁的 selfInterrupt 一样
                        failed = false;
                        return;
                    }
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                        parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```





## 释放锁

### 差异

#### 相同之处

1. 先释放锁，再唤醒后继节点

#### 差异之处

1. 独占锁，某一节点释放锁的时候，才去唤醒后继节点
2. 共享锁，两种情况回去唤醒节点
   1. 释放共享锁，
   2. 成功获取锁之后

#### 注意事项

1. doReleaseShared 什么时候退出循环？
   答：h == head 的时候，按照 if (h != null && h != tail)  期间，head 有没有被改变分为两种情况
           1. 头节点没有被改变，这是正常情况，唤醒节点后退出
              2. 头节点已经被改变，这时候循环，继续释放新的头节点，如此反复，总有结束的时候（h == head）
                 此期间可能会有 doReleaseShared 的调用风暴，这样加快节点的唤醒

```java
    //独占锁 release
    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }

    //共享锁 releaseShared
    public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) { //释放锁，类比独占锁的 tryRelease
            doReleaseShared(); //唤醒后继节点（有传播性），类比独占锁的 unparkSuccessor
            return true;
        }
        return false;
    }
    private void doReleaseShared() {
        for (;;) {
            Node h = head;
            if (h != null && h != tail) { //节点数 >= 2
                int ws = h.waitStatus;
                if (ws == Node.SIGNAL) { //说明后继节点需要唤醒
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0)) //doReleaseShared 调用风暴,保证只有一个能成功
                        continue;            // loop to recheck cases 设置失败，重新循环
                    unparkSuccessor(h);//CAS 保证只有一个能调用 unparkSuccessor
                }
                else if (ws == 0 && //队列的最后一个节点成为了头节点(ws == 0),因为队列加入末尾加入新节点，会将前驱节点改为 SIGNAL
                        !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            if (h == head)                   // loop if head changed  如果head变了，重新循环
                break;
        }
    }
```

