# BIO_NIO_AIO

## BIO

1. Server 流程
   * new ServerSocket() 并绑定端口
   * 死循环，等待客户端连接：Socket socket = serverSocket.accept()
   * 拿到连接后，从连接(socket) 中获取数据
2. Client 流程
   * Socket socket = new Socket() 指定服务端的 ip 和 端口
   * 从 socket 中获取输出流并写出数据
3. 特点：
   1. ServerSocket.accept() 等待连接时阻塞
   2. Socket.read() 阻塞等待客户端数据
      如果客户端一直不发数据，这个线程会一直在那等着
      * 多个线程处理 Socket.read()：太浪费，一个个线程在那儿傻傻等着客户端发消息，而且还有线程间竞争 CPU
      * 一个线程处理 Socket.read()：其他线程无法连接或发送数据

```java
package com.javayang.test.test1.netty.bio;

import lombok.SneakyThrows;
import java.io.IOException;
import java.net.ServerSocket;
import java.net.Socket;

public class TestSocketServcer {
    public static void main(String[] args) throws IOException {
        ServerSocket serverSocket = new ServerSocket(8088);
        while (true){
            System.out.println("等待客户端连接...");
            Socket socket = serverSocket.accept();
            System.out.println("客户端已连接");
            //new 新线程：线程太多，可能有的客户端一直不发数据，占着茅坑不拉屎
            //不 new 新线程：阻塞，只有一个客户端能连接
            new Thread(new Runnable() {
                @SneakyThrows
                @Override
                public void run() {
                    System.out.println("等待客户端发数据...");
                    byte[] bytes = new byte[1024];
                    //输入流：接收数据
                    int readSize = socket.getInputStream().read(bytes);
                    if(readSize >= 0){
                        System.out.println("收到数据："+new String(bytes,0,readSize));
                    }
                }
            }).start();
        }
    }
}
```

```java
package com.javayang.test.test1.netty.bio;

import java.io.IOException;
import java.io.OutputStream;
import java.net.Socket;

public class TestSocketClient {
    public static void main(String[] args) throws IOException {
        Socket socket = new Socket("127.0.0.1",8088);
        //输出流：发送数据
        OutputStream outputStream = socket.getOutputStream();
        outputStream.write("我是客户端1".getBytes("UTF-8"));
        outputStream.flush();
    }
}
```



## NIO（多路复用器 selector）

**selector 底层有可能 select、poll、epoll，取决于操作系统的实现**

### NIO的两种形式

1. 方式1：将客户端Channel加入集合，每次有响应就遍历所有的channel

   * 缺点：
     * 每次都遍历所有的 Channel，比如集合里有1w个channel，但只有一个channel有事件，遍历1w个channel太浪费
     * 一个事件处理慢，影响到其他事件

2. 方式2：把Channel绑定到Selector，有事件时，selector.select()只响应有事件的Channel

   * 利用Selector，事件驱动，只返回有事件的Channel

     ```java
     主要流程
     Selector.open() //创建多路复用器
     socketChannel.register() //将channel注册到多路复用器上
     selector.select() //阻塞等待需要处理的事件发
     ```

   * Epoll

     ```java
     epoll三个主要函数：
     1. int epoll_create(int size)  创建epoll实例，返回文件描述符
     2. int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)   注册新的fd到epfd中，并关联事件event
     3. int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout)   等待文件描述符epfd上的事件
         epfd是Epoll对应的文件描述符，events表示调用者所有可用事件的集合，maxevents表示最多等到多少个事件就返回，
     timeout是超时时间。
     ```

     |              | **select**                               | **poll**                                 | **epoll(jdk 1.5及以上)**                                     |
     | ------------ | ---------------------------------------- | ---------------------------------------- | ------------------------------------------------------------ |
     | **操作方式** | 遍历                                     | 遍历                                     | 回调                                                         |
     | **底层实现** | 数组                                     | 链表                                     | 哈希表                                                       |
     | **IO效率**   | 每次调用都进行线性遍历，时间复杂度为O(n) | 每次调用都进行线性遍历，时间复杂度为O(n) | 事件通知方式，每当有IO事件就绪，系统注册的回调函数就会被调用，时间复杂度O(1) |
     | **最大连接** | 有上限                                   | 无上限                                   | 无上限                                                       |

1. NIO  Server 流程
   * 创建服务端 Channel 阶段
     * 创建 ServerSocketChannel 并绑定端口，设置成非阻塞**。ServerSocketChannel.open().**   
     * 创建 Selector。**selector = Selector.open()**
     * Channel 和 Selector 绑定并指定事件。**serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT)**
   * 阻塞等待消息，遍历SelectionKey     selector.select()
     * 接收到连接事件（创建客户端 Channel）
       * 从 SelectionKey 中得到服务端的 ServerSocketChannel     **key.channel()**
       * 新建客户端的 SocketChannel  并设置成非阻塞  **server.accept()**
       * 客户端 Channel 和 Selector 绑定并指定事件**socketChannel.register(selector,SelectionKey.OP_READ);**
     * 接收到消息
       * 从 SelectionKey 中得到客户端的 SocketChannel   **key.channel()**
       * 开辟直接内存并读取消息 socketChannel.read(ByteBuffer.allocateDirect(128))


```java
//方式1：将客户端Channel加入集合，每次有响应就遍历所有的channel
public class TestNIOSocketServer1 {
    public static void main(String[] args) throws IOException {
        List<SocketChannel> channelList = new ArrayList<>();
        //创建 NIO 服务端
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
        serverSocketChannel.socket().bind(new InetSocketAddress(8088));
        //设置成 非阻塞
        serverSocketChannel.configureBlocking(false);
        System.out.println("NIOSocketServer 启动成功");
        while (true){
            SocketChannel socketChannel = serverSocketChannel.accept();
            if(socketChannel != null){
                System.out.println("有一个连接成功了");
                //客户端连接设置成非阻塞的
                socketChannel.configureBlocking(true);
                //客户端 channel 保存到 list
                channelList.add(socketChannel);
            }
            //遍历连接
            Iterator<SocketChannel> iterator = channelList.iterator();
            while (iterator.hasNext()){
                SocketChannel channel = iterator.next();
                //开辟内存
                ByteBuffer byteBuffer = ByteBuffer.allocate(124);
                //从 channel 读取数据
                int len = channel.read(byteBuffer);
                if(len > 0){
                    System.out.println("收到消息:" + new String(byteBuffer.array()));
                }else if(len == -1){
                    iterator.remove();
                }
            }
        }
    }
}

```

```java
//方式2：把Channel注册到Selector，有事件时，selector.select()只响应有事件的Channel
public class TestNIOSocketServer2 {
    public static void main(String[] args) throws IOException {
        //创建 NIO 服务端
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
        serverSocketChannel.socket().bind(new InetSocketAddress(8088));
        //设置成非阻塞
        serverSocketChannel.configureBlocking(false);
        //创建 Selector(取决于系统支持：linux:epoll; window:selctor)
        Selector selector = Selector.open();
        //ServerSocketChannel注册到selector上,绑定accept事件
        serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);

        while (true){
            //阻塞等待事件发送
            System.out.println("==============阻塞等待事件==============");
            selector.select();
            System.out.println("哎呀，有事件了...");
            //获取 selector 中有事件的 SelectionKey
            Set<SelectionKey> selectionKeySet = selector.selectedKeys();
            //迭代遍历 selectionKeySet
            Iterator<SelectionKey> iterator = selectionKeySet.iterator();
            while (iterator.hasNext()){
                SelectionKey key = iterator.next();
                //连接事件
                if(key.isAcceptable()){
                    System.out.println("连接事件");
                    //从 SelectionKey 中得到服务端的 ServerSocketChannel
                    ServerSocketChannel server = (ServerSocketChannel)key.channel();
                    //新建客户端的 SocketChannel
                    SocketChannel socketChannel = server.accept();
                    // 客户端的 SocketChannel 设置成非阻塞
                    socketChannel.configureBlocking(false);
                    //将客户端 SocketChannel 绑定到Selector，并绑定响应事件
                    socketChannel.register(selector,SelectionKey.OP_READ);
                    System.out.println("客户端连接成功了");
                }else if(key.isReadable()){
                    //读事件
                    System.out.println("收到消息事件");
                    //从 SelectionKey 中得到客户端的 SocketChannel
                    SocketChannel socketChannel = (SocketChannel) key.channel();
                    //开辟直接内存
                    ByteBuffer byteBuffer = ByteBuffer.allocateDirect(128);
                    //从 Channel 中读取数据
                    int len = socketChannel.read(byteBuffer);
                    if(len > 0){
                        byte[] bytes = new byte[128];
                        ByteBuffer.wrap(bytes,0,len);
                        System.out.println("收到客户端数据："+new String(bytes));
                    }
                }
                //删除已处理的 SelectionKey
                iterator.remove();
            }
        }
    }
}
```



## AIO

不阻塞，利用系统底层，有事件时，回调我们的钩子函数。
底层实际是对 NIO 的封装。

```java
public class TestAIOSocketServer {
    public static void main(String[] args) throws IOException, InterruptedException {
        AsynchronousServerSocketChannel serverChannel = AsynchronousServerSocketChannel.open().bind(new InetSocketAddress(8088));
        //异步：有连接事件时回调 CompletionHandler#completed
        serverChannel.accept(null, new CompletionHandler<AsynchronousSocketChannel, Object>() {
            @Override
            public void completed(AsynchronousSocketChannel socketChannel, Object attachment) {
                System.out.println("客户端开始连接");
                //接收客户端连接
                serverChannel.accept(attachment,this);
                ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
                //异步：有读取数据事件时回调 CompletionHandler#completed
                socketChannel.read(byteBuffer, byteBuffer, new CompletionHandler<Integer, ByteBuffer>() {
                    @Override
                    public void completed(Integer result, ByteBuffer buffer) {
                        System.out.println("接收到客户端数据");
                        buffer.flip();
                        //接收到客户端数据
                        System.out.println(new String(buffer.array()));
                        //响应数据
                        socketChannel.write(ByteBuffer.wrap("服务器收到了请求".getBytes()));
                    }

                    @Override
                    public void failed(Throwable exc, ByteBuffer attachment) {
                        exc.printStackTrace();
                    }
                });
            }

            @Override
            public void failed(Throwable exc, Object attachment) {

            }
        });
        Thread.sleep(Integer.MAX_VALUE);
    }
}
```

