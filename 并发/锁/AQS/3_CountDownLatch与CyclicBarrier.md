# CountDownLatch与CyclicBarrier

## 倒计时器CountDownLatch

## 概念

1. 场景：协作时，需要等到其他多个线程完成任务之后，主线程才能继续往下执行。
   即：每个任务执行完，CountDownLatch#countDown 使得 state-1，直到 state == 0
2. 常见方法：
   1. countDown()：初始值 state 减1
   2. getCount()：获取 state 的值
   3. await()：线程阻塞，直到 state==0 才能继续往下执行
   4. await(long timeout, TimeUnit unit)：线程阻塞，直到 state==0 才能继续往下执行。
      但是如果超时，不论 state 释放等于0，都会继续往下执行

### 源码

![CountDownLatch](https://raw.githubusercontent.com/huan415/JavaYang/master/assets/CountDownLatch.jpg)









## 循环栅栏CyclicBarrier

### 概念

1. 场景：多个线程都达到了指定点后，才能继续往下继续执行

### 源码

![CyclicBarrier](https://raw.githubusercontent.com/huan415/JavaYang/master/assets/CyclicBarrier.jpg)













## CountDownLatch与CyclicBarrier的比较

两个都用于控制并发（计数器）

1. CountDownLatch 某个线程等到其他若干个线程执行完之后，它才能执行 **(强调的是一个线程等其他多个线程完成某件事情)**
   CyclicBarrier 一组线程相互等待至某个状态时，一组线程一起执行 **(强调的是多个线程相互等待，大家都完成，再携手共进)**
2. CountDownLatch#countDown()，当前线程不会阻塞，会继续往下执行
   CyclicBarrier#await()，当前线程会阻塞，直到所有的线程都到达指定点，才会继续往下执行
3. CountDownLatch 不能复用
   CyclicLatch 可以复用