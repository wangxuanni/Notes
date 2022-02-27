

socket是一个通用概念，c++的sicket和java的socket也可以通信

连接对应的线程数量：bio每一个请求对应一个线程，多少个连接就多少个线程，阻塞等待读写事件。消耗主要花费在线程上下文切换。

nio把请求注册在多路复用器上，一个多路复用器线程就可以维持很多个链接，连接和读写对多路复用器来说只是事件。消耗：虽然事情还是一样要做，但多路复用器做了连接注册、轮询等工作。

阻塞：bio在内核准备好数据复制到用户空间这个过程是线程是阻塞，nio只有多路复用器是阻塞

bio在内核准备好数据复制到用户空间这个过程是线程是阻塞，nio只有多路复用器是阻塞

**一对一，连接维持，阻塞等内核复制到用户空间，消耗在上下文切换**          

nio适合连接数多且连接短的操作

其他：nio基于块，sc只能在bb里读

epoll用红黑树存放socket达到操作O（1），链表存储就绪事件

# bio，阻塞并同步

特点：IO两个阶段被阻塞，基于流模型。

客户端一个请求服务端就启动一个线程，线序发生请求给内核，由内核去通信，在内核准备好数据之前，线程是被挂起的，知道数据从内核复制到用户空间。调用是可靠的线性顺序。

缺点：每次请求创建一个线程在销毁开销比较大，操作系统对线程的总数有限制，太多服务器可能瘫痪。可以用线程池改进。



# nio，非阻塞并同步

可构建多路复用、同步非阻塞的io操作。

特点：程序去不断询问内核是否准备好，基于buffers。

客户端请求会注册到多路复用器上，单线程轮询到有io请求时，才启动一个线程进行处理，仅仅selector是阻塞的。

核心：Channels（类似流，全双工，可以读写。socketChannel、serverSocketChannel）

buffers8种基本类型都有，数据从channel读到buffer中，也可以从buffer写到channel。本质是一块方便读写数据的内存

selector多路复用器，以监听多个Channel通道感兴趣的事情。允许单线程处理多个channel。 

OP_ACCEPT: 接收就绪，OP_READ: 读取就绪，OP_WRITE: 写入就绪，OP_CONNECT: 连接就绪，其中只有OP_ACCEPT: 接收就绪是serviceSocketChannel使用的，其他三个都是socketChannel使用。

## select、poll、epoll

IO多路复用优点在于**单线程能处理多个网络IO**,共有select、poll、epoll三种l方式。nio会根据操作系统的不同而选择不同的多路复用IO，linux下用的是epoll，其他poll。

|                        | select                       | poll         | epoll                           |
| ---------------------- | ---------------------------- | ------------ | ------------------------------- |
| 一个进程能打开的连接数 | 基于数组，大小为1040         | 基于链表无限 | 有限，但很大，1G内存能打开1万个 |
| 随FD增加后的效率       | O(N)每次调用都会进行线性遍历 | 同select     | O（1）基于回调函数              |
| 消息传递方式           | 需要内核拷贝动作             | 同select     | 内核和用户空间共享一块内存      |

## epoll

epoll的高效就在于，当我们调用epoll_ctl往里塞入百万个句柄时，epoll_wait仍然可以飞快的返回，并有效的将发生事件的句柄给我们用户。这是由于我们在调用epoll_create时，内核除了帮我们在epoll文件系统里建了个file结点，**在内核cache里建了个红黑树用于存储以后epoll_ctl传来的socket外，还会再建立一个list链表，用于存储准备就绪的事件，当epoll_wait调用时，仅仅观察这个list链表里有没有数据即可。**有数据就返回，没有数据就sleep，等到timeout时间到后即使链表没数据也返回。所以，epoll_wait非常高效。

epoll_create：在epoll文件系统建立、file结点红黑树、list链表

epoll_wait：与select函数类似，等待文件描述符发生变化。

epoll_ctl：控制某个 Epoll 文件描述符上的事件：注册、修改、删除。

### 两种模式

LT(level triggered，**水平触发模式**)是缺省的工作方式，并且同时支持 block 和 non-block socket。在这种做法中，内核告诉你一个文件描述符是否就绪了，然后你可以对这个就绪的fd进行IO操作。如果你不作任何操作，内核还是会继续通知你的，所以，这种模式编程出错误可能性要小一点。

ET(edge-triggered，**边缘触发模式**)是高速工作方式，只支持no-block socket。在这种模式下，当描述符从未就绪变为就绪时，内核通过epoll告诉你。然后它会假设你知道文件描述符已经就绪，并且不会再为那个文件描述符发送更多的就绪通知，等到下次有新的数据进来的时候才会再次出发就绪事件。

**epoll的LT和ET的区别**

LT：水平触发，效率会低于ET触发，尤其在大并发，大流量的情况下。但是LT对代码编写要求比较低，不容易出现问题。LT模式服务编写上的表现是：只要有数据没有被获取，内核就不断通知你，因此不用担心事件丢失的情况。
ET：边缘触发，效率非常高，在并发，大流量的情况下，会比LT少很多epoll的系统调用，因此效率高。但是对编程要求高，需要细致的处理每个请求，否则容易发生丢失事件的情况。



> [IO复用_epoll函数](https://www.cnblogs.com/orange1438/p/4637810.html)
>
> [epoll的两种模式详解](https://www.xuebuyuan.com/1604600.html?mobile=0)



### 惊群

当多个进程/线程在同时阻塞等待同一个事件时，如果这个事件发生会唤醒所有的进程，但最终只可能有一个进程/线程对该事件进行处理，其他进程/线程会在失败后重新休眠，**这种性能浪费就是惊群。**

#### accept()的解决

Linux2.6内核解决了accept()函数的“惊群”问题，大概的处理方式就是，当内核接收到一个客户连接后，**只会唤醒等待队列上的第一个进程或线程**。

#### epoll_wait的部分解决

但对于实际工程中常见的服务器程序，大多服务器不是阻塞在accept，而是阻塞在select、poll或epoll_wait。

epoll_wait进程的解决方案也是**只会唤醒等待队列上的第一个进程或线程**，但只能说解决了**部分的**epoll的惊群问题。epoll还存在惊群的场景如下：在worker保持工作的状态下，都会被唤醒，例如在epoll_wait后调用sleep一次。

#### Nginx的解决

Nginx使用**全局mutex互斥锁**解决这个问题。每个子进程在epoll_wait()之前先去申请锁，申请到则继续处理，获取不到则等待，并设置了一个负载均衡的[算法](http://www.haodaima.net/tag/算法)（当某一个子进程的任务量达到总设置量的7/8时，则不会再尝试去申请锁）来均衡各个进程的任务量。



[Linux网络编程“惊群”问题总结](https://www.cnblogs.com/Anker/p/7071849.html)



# aio，非阻塞且异步

基于**事件和回调机制**，注册事件完不会阻塞，立即返回，程序准备好操作系统会通知响应线程一基于回调函数，返回futureselectepollpoll

[epoll博客](https://blog.51cto.com/yaocoder/888374 ) 
项目每一个请求分给一个线程——>线程池executorService——>nio

```java
public class NioServer {
    public void start() throws IOException {
        /**
         * 1. 创建Selector
         */
        Selector selector = Selector.open();
        /**
         * 2. 通过ServerSocketChannel创建channel通道
         */
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
        /**
         * 3. 为channel通道绑定监听端口
         */
        serverSocketChannel.bind(new InetSocketAddress(8000));
        /**
         * 4. **设置channel为非阻塞模式**
         */
        serverSocketChannel.configureBlocking(false);
        /**
         * 5. 将channel注册到selector上，监听连接事件
         */
        serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
        System.out.println("服务器启动成功！");
        /**
         * 6. 循环等待新接入的连接
         */
        for (;;) { // while(true) c for;;
            /**
             * TODO 获取可用channel数量
             */
            int readyChannels = selector.select()
            /**
             * TODO 注意这里
             */
            if (readyChannels == 0) continue;
            /**
             * 获取可用channel的集合
             */
            Set<SelectionKey> selectionKeys = selector.selectedKeys();
           Iterator iterator = selectionKeys.iterator();
            while (iterator.hasNext()) {
                /**
                 * selectionKey实例
                 */
   SelectionKey selectionKey = (SelectionKey) iterator.next();
                /**
                 * **移除Set中的当前selectionKey**
                 */
                iterator.remove();
                /**
                 * 7. 根据就绪状态，调用对应方法处理业务逻辑
                 */
                /**
                 * 如果是 接入事件
                 */
                if (selectionKey.isAcceptable()) {
                    acceptHandler(serverSocketChannel, selector);
                }
                /**
                 * 如果是 可读事件
                 */
                if (selectionKey.isReadable()) {
                    readHandler(selectionKey, selector);}}}}
```

项目问题

endpoint实现了AQS共享模式，访问计数器在1w以内，连接器线程。

欺骗线程池阻塞队列已经满了。

先不进行奢侈的字节数组的复制，而是拿到某信息的坐标（打标），在需要的时候转为String

工人线程组缓存是对任务实例的复用，完成任务后就成员变量清空，给下一个任务用，达到不用一个工作一个实例的目的。缓存就是把任务实例放在CurrentQueueList里

动态代理的代理类是在运行是动态生成的，编译期是看不到了该class文件的。静态代理的代理类已经写好，编译可以看到编译好的代理类class文件

[外观模式](https://join.qq.com/apply.php)

[idea收藏一个类](https://jingyan.baidu.com/album/aa6a2c14b8bed80d4c19c4f6.html?picindex=1)