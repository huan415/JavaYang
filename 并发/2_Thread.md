# Thread

## 线程状态

|                           线程工作                           |            状态            | 备注 |
| :----------------------------------------------------------: | :------------------------: | :--: |
|                           new 线程                           |      **NEW（新建）**       |      |
|                           start()                            |    **READY（可运行）**     |      |
|                        获得CPU 时间片                        |    **RUNNING（运行）**     |      |
|                            wait()                            |    **WAITING（等待）**     |      |
| 等待超时<br />sleep（long millis）`方法或 `wait（long millis） | **TIME_WAITING(超时等待)** |      |
|                  第一次获取锁，没有获取到锁                  |    **BLOCKED（阻塞）**     |      |
|                      run()方法执行完了                       |   **TERMINATED（终止）**   |      |



## 线程间操作

1. 中断（interrupted） 实现线程间交互 ----  终止一个线程的运行

   |               方法名                |         详细解释         |                           **备注**                           |
   | :---------------------------------: | :----------------------: | :----------------------------------------------------------: |
   |       public void interrupt()       |      中断该线程对象      | 如果该线程被调用了Object wait/Object wait(long)，或者被调用sleep(long)，join()/join(long)方法时会抛出interruptedException并且中断标志位将会被清除 |
   |   public boolean isinterrupted()    | 测试该线程对象是否被中断 |                     中断标志位不会被清除                     |
   | public static boolean interrupted() |  测试当前线程是否被中断  |                      中断标志位会被清除                      |

   

2. join    线程间协作的一种方式
   线程实例A执行了threadB.join()，其含义是：当前线程A会等待threadB线程终止后threadA才会继续执行

3. |                            方法名                            |                           详细解释                           |                           **备注**                           |
   | :----------------------------------------------------------: | :----------------------------------------------------------: | :----------------------------------------------------------: |
   |     public final void join() throws InterruptedException     |                      等待这个线程死亡。                      | 如果任何线程中断当前线程，如果抛出InterruptedException异常时，当前线程的中断状态将被清除 |
   | public final void join(long millis) throws InterruptedException | 等待这个线程死亡的时间最多为`millis`毫秒。 `0`的超时意味着永远等待。 |      如果millis为负数，抛出IllegalArgumentException异常      |
   | public final void join(long millis, int nanos) throws InterruptedException |     等待最多`millis`毫秒加上这个线程死亡的`nanos`纳秒。      | 如果millis为负数或者nanos不在0-999999范围抛出IllegalArgumentException异常 |

4.  sleep(long millis)     指定的时间休眠 **(不会去获取锁)**

5. yield
   当前线程让出CPU （**给当前线程相同优先级**）

## sleep() VS wait()

1. sleep()方法是Thread的静态方法，而wait是Object实例方法
2. sleep()方法在休眠时间达到后，如果再次获得CPU时间片就会继续执行
   wait()方法必须等待Object.notift/Object.notifyAll通知后，才会离开等待池，并且再次获得CPU时间片才会继续执行
3. sleep()方法只是会让出CPU并不会释放掉对象锁
   wait()方法会释放占有的对象锁，使得该线程进入等待池中，等待下一次获取资源
4. sleep()方法在任何地方使用
   wait()方法必须要在同步方法或者同步块中调用，也就是必须已经获得对象锁



