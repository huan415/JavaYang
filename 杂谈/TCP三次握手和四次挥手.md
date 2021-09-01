# TCP三次握手和四次挥手

## 三次握手

1. Service看到SYN=1, 就会知道Client要发起请求。
2. 为什么要有三次握手？两次不行？
   答: 为了保证连接安全可靠，Serivce和Client都要保证能收到消息和发送出去的消息能被收到。
   如果只有两次握手：Client保证能发送和接收
   Service只保证了接收(Serivce不知道发出去的消息有没有被接收)

## 四次挥手

1. 为什么要四次？第2步和第三步可以合并？

   答：不能，因为tcp为了保证连接安全可靠，Service收到消息后只能回复收到，不能给本地发起断开连接的请求，因为这时候Service有可能还在往本地发送请求，要等消发送完再发起断开连接请求



![](https://raw.githubusercontent.com/huan415/JavaYang/master/assets/TCP%E4%B8%89%E6%AC%A1%E6%8F%A1%E6%89%8B%E5%92%8C%E5%9B%9B%E6%AC%A1%E6%8C%A5%E6%89%8B.jpg)