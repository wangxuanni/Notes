



> [官方文档](http://dubbo.apache.org/zh-cn/docs/user/quick-start.html)

## 说一下 Dubbo 的工作原理？注册中心挂了可以继续通信吗？

### dubbo 工作原理

- 第一层：service 层，接口层，给服务提供者和消费者来实现的

- 第二层：config 层，配置层，主要是对 dubbo 进行各种配置的

- 第三层：proxy 层，服务代理层，无论是 consumer 还是 provider，dubbo 都会给你生成代理，代理之间进行网络通信

- 第四层：registry 层，服务注册层，负责服务的注册与发现

- 第五层：cluster 层，集群层，封装多个服务提供者的路由以及负载均衡，将多个实例组合成一个服务

- 第六层：monitor 层，监控层，对 rpc 接口的调用次数和调用时间进行监控

- 第七层：protocal 层，远程调用层，封装 rpc 调用

- 第八层：exchange 层，信息交换层，封装请求响应模式，同步转异步

- 第九层：transport 层，网络传输层，抽象 mina 和 netty 为统一接口

- 第十层：serialize 层，数据序列化层

  ![img](file:///C:\Users\home.11\AppData\Roaming\Tencent\Users\957681484\QQ\WinTemp\RichOle\7M5Y4P9[0~N~L1]CH6O9%_J.png)

### 工作流程

- 第一步：provider 向注册中心去注册
- 第二步：consumer 从注册中心订阅服务，注册中心会通知 consumer 注册好的服务
- 第三步：consumer 调用 provider
- 第四步：consumer 和 provider 都异步通知监控中心

![dubbo原理](C:\Users\home.11\Desktop\小a笔记\images\dubbo原理.png)

### 注册中心挂了可以继续通信吗？

可以，因为刚开始初始化的时候，消费者会将提供者的地址等信息**拉取到本地缓存**，所以注册中心挂了可以继续通信。

## Dubbo 支持哪些序列化协议？

### dubbo 支持不同的通信协议

- dubbo 协议

默认就是走 dubbo 协议，单一**长连接**，进行的是 **NIO** 异步通信，基于 **hessian** 作为序列化协议。使用的场景是：传输数据量小（每次请求在 100kb 以内），但是并发量很高。

### dubbo 支持的序列化协议

dubbo 支持 hession、Java 二进制序列化、json、SOAP 文本序列化多种序列化协议。但是 hessian 是其默认的序列化协议。

### 说一下 Hessian 的数据结构

Hessian 的对象序列化机制有 8 种原始类型：

- 原始二进制数据
- boolean
- 64-bit date（64 位毫秒值的日期）
- 64-bit double
- 32-bit int
- 64-bit long
- null
- UTF-8 编码的 string

另外还包括 3 种递归类型：

- list for lists and arrays
- map for maps and dictionaries
- object for objects

还有一种特殊的类型：

- ref：用来表示对共享对象的引用。

### 为什么 PB 的效率是最高的？

可能有一些同学比较习惯于 `JSON` or `XML` 数据存储格式，对于 `Protocol Buffer` 还比较陌生。`Protocol Buffer` 其实是 Google 出品的一种轻量并且高效的结构化数据存储格式，性能比 `JSON`、`XML` 要高很多。

其实 PB 之所以性能如此好，主要得益于两个：**第一**，它使用 proto 编译器，自动进行序列化和反序列化，速度非常快，应该比 `XML` 和 `JSON` 快上了 `20~100` 倍；**第二**，它的数据压缩效果好，就是说它序列化后的数据量体积小。因为体积小，传输起来带宽和速度上会有优化。

## 负载均衡策略和集群容错策略都有哪些？动态代理策略呢？

- dubbo 工作原理：服务注册、注册中心、消费者、代理通信、负载均衡；
- 网络通信、序列化：dubbo 协议、长连接、NIO、hessian 序列化协议；
- 负载均衡策略、集群容错策略、动态代理策略：dubbo 跑起来的时候一些功能是如何运转的？怎么做负载均衡？怎么做集群容错？怎么生成动态代理？
- dubbo SPI 机制：你了解不了解 dubbo 的 SPI 机制？如何基于 SPI 机制对 dubbo 进行扩展？

### dubbo 负载均衡策略

#### random loadbalance

默认情况下，dubbo 是 random load balance ，即**随机**调用实现负载均衡，可以对 provider 不同实例**设置不同的权重**，会按照权重来负载均衡，权重越大分配流量越高，一般就用这个默认的就可以了。

#### roundrobin loadbalance

这个的话默认就是均匀地将流量打到各个机器上去（轮询），但是如果各个机器的性能不一样，容易导致性能差的机器负载过高。所以此时需要调整权重，让性能差的机器承载权重小一些，流量少一些。

#### leastactive loadbalance

这个就是自动感知一下，如果某个机器性能越差，那么接收的请求越少，越不活跃，此时就会给**不活跃的性能差的机器更少的请求**。

#### consistanthash loadbalance

一致性 Hash 算法，相同参数的请求一定分发到一个 provider 上去，provider 挂掉的时候，会基于虚拟节点均匀分配剩余的流量，抖动不会太大。**如果你需要的不是随机负载均衡**，是要一类请求都到一个节点，那就走这个一致性 Hash 策略。

### dubbo 集群容错策略

#### failover cluster 模式

失败自动切换，自动重试其他机器，**默认**就是这个，常见于读操作。（失败重试其它机器）

可以通过以下几种方式配置重试次数：

```
<dubbo:service retries="2" />
```

或者

```
<dubbo:reference retries="2" />
```

或者

```
<dubbo:reference>
    <dubbo:method name="findFoo" retries="2" />
</dubbo:reference>
```

#### failfast cluster 模式

一次调用失败就立即失败，常见于非幂等性的写操作，比如新增一条记录（调用失败就立即失败）

#### failsafe cluster 模式

出现异常时忽略掉，常用于**不重要**的接口调用，比如记录日志。

配置示例如下：

```
<dubbo:service cluster="failsafe" />
```

或者

```
<dubbo:reference cluster="failsafe" />
```

#### failback cluster 模式

失败了后台自动记录请求，然后定时重发，比较适合于写消息队列这种。

#### forking cluster 模式

**并行调用**多个 provider，只要一个成功就立即返回。常用于实时性要求比较高的读操作，但是会浪费更多的服务资源，可通过 `forks="2"` 来设置最大并行数。

#### broadcacst cluster

逐个调用所有的 provider。任何一个 provider 出错则报错（从`2.1.0` 版本开始支持）。通常用于通知所有提供者更新缓存或日志等本地资源信息。

### dubbo动态代理策略

默认使用 javassist 动态字节码生成，创建代理类。但是可以通过 spi 扩展机制配置自己的动态代理策略。

## Dubbo 的 spi 思想？

### SPI

**spi，service provider interface`，服务提供接口。服务方提供一个接口，由指定配置去加载实现类（实现类可以是自定义插件）。**

spi用于插件扩展的场景。比如，Java 定义了一套 jdbc 的接口，但是并没有提供 jdbc 的实现类。我们要根据自己使用的数据库，引入相关jar包。比如 `mysql-jdbc-connector.jar`、`oracle-jdbc-connector.jar` 。项目跑的时候，接口底层使用引入的 jar 中提供的实现类。

有什么用：可以实现插件扩展功能。

### dubbo 的 spi 

dubbo 自己实现了一套 spi 机制。在系统运行的时候，去找一个你配置的 Protocol实现类，加载到 jvm 中来实例化对象。如果你没配置，那就走默认的实现好了。

```
Protocol protocol = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();
```

*Protocol 接口*,@SPI("dubbo")` 意思是通过 SPI 机制来提供实现类。实现类是通过 ‘dubbo’ 作为默认 key 去配置文件里找到的，‘dubbo ’找到实现类就是 com.alibaba.dubbo.rpc.protocol.dubbo.DubboProtocol。

 `@Adaptive` 注解，这两个方法会被代理实现。

```java
@SPI("dubbo")  
public interface Protocol {  
      
    int getDefaultPort();  
  
    @Adaptive  
    <T> Exporter<T> export(Invoker<T> invoker) throws RpcException;  
  
    @Adaptive  
    <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException;  

    void destroy();  
 
}  
```

*配置文件*，就是上面@SPI("dubbo")去找的配置文件。在 dubbo 自己的 jar 里，在`/META_INF/dubbo/internal/com.alibaba.dubbo.rpc.Protocol`文件中：

```
dubbo=com.alibaba.dubbo.rpc.protocol.dubbo.DubboProtocol
http=com.alibaba.dubbo.rpc.protocol.http.HttpProtocol
hessian=com.alibaba.dubbo.rpc.protocol.hessian.HessianProtocol
```

dubbo 使用spi扩展机制就可以实现许多扩展点，比如协议、集群、路由、负载均衡、注册中心、动态代理、序列化、线程池、网络传输。



### 如何自己扩展 dubbo 中的组件

[官方文档说明](http://dubbo.apache.org/zh-cn/docs/dev/impls/protocol.html)

1. 编写三个接口
2. 添加配置文件的key、value信息

```xml
src
 |-main
    |-java
        |-com
            |-xxx
                |-XxxProtocol.java (实现Protocol接口)
                |-XxxExporter.java (实现Exporter接口)
                |-XxxInvoker.java (实现Invoker接口)
    |-resources
        |-META-INF
            |-dubbo
                |-org.apache.dubbo.rpc.Protocol (纯文本文件，内容为：xxx=com.xxx.XxxProtocol)
```

更具体一点：完成上面提到的接口和配置文件，把这个工程打成 jar 包。

在另一个 `dubbo provider` 工程依赖你自己 jar，在 spring 配置文件配置：

```
<dubbo:protocol name=”my” port=”20000” />
```

通过上述方式，可以替换掉大量的 dubbo 内部的组件，打个自己的 jar 包，然后配置一下。

## 基于 Dubbo 进行服务治理、服务降级、失败重试以及超时重试？

## 分布式服务接口的幂等性如何设计（比如不能重复扣款）？

##  分布式服务接口请求的顺序性如何保证？

## 如何自己设计一个类似 Dubbo 的 RPC 框架？



- 上来你的服务就得去注册中心注册吧，你是不是得有个注册中心，保留各个服务的信息，可以用 zookeeper 来做，对吧。
- 然后你的消费者需要去注册中心拿对应的服务信息吧，对吧，而且每个服务可能会存在于多台机器上。
- 接着你就该发起一次请求了，咋发起？当然是基于动态代理了，你面向接口获取到一个动态代理，这个动态代理就是接口在本地的一个代理，然后这个代理会找到服务对应的机器地址。
- 然后找哪个机器发送请求？那肯定得有个负载均衡算法了，比如最简单的可以随机轮询是不是。
- 接着找到一台机器，就可以跟它发送请求了，第一个问题咋发送？你可以说用 netty 了，nio 方式；第二个问题发送啥格式数据？你可以说用 hessian 序列化协议了。
- 服务器那边一样的，需要针对你自己的服务生成一个动态代理，监听某个网络端口了，然后代理你本地的服务代码。接收到请求的时候，就调用对应的服务代码，对吧。