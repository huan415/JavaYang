# 并发工具之Semaphore与Exchanger

## 并发工具之Semaphore与Exchanger

### 概念

1. Semaphore：信号量、许可证，多个线程能同时并发获取到锁
   例如：new Semaphore(10)。相当于有10个许可证，允许有10个线程并发执行
2. 场景：适合用于流量控制



## 线程间交换数据的工具Exchanger

1. 线程间协作工具，俩个线程间交换数据
2. 一个线程执行了exchange方法，它会阻塞等待另一个线程也执行exchange方法，这个时候两个线程都到达了同步点，这个时候就可以交换数据。