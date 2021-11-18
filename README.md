# Netty
## 1.简介
### 1.1 JDK 原生NIO的API的问题：

    1. Java NIO 的 API 非常复杂，需要熟悉Selector、ServerSocketChannel、SocketChannel、ByteBuffer 等组件；
    2. 开发量和难度增加，核心逻辑较简单，需要补齐可靠性方面的问题，例如：网络波动导致的重连、半包读写等；
    3. JDK自身的bug：Epoll Bug，导致Selector空轮询，cpu使用率达到100%、TCP 断线重连、keep-alive 检测等问题
    注：Epoll bug简介
    1. 现象：Selectors-用于判断channle发生IO事件的选择器，SelectionKeys-负责IO事件的状态与绑定 
        Java 原生 NIO 来编写服务器应用时，正常情况下selector.select()操作是阻塞的，只有被监听的fd(文件描述符)有读写操作时，才被唤醒，在该bug下，没有任何fd有读写请求，select()操作依旧被唤醒，因此selectKeys返回的是个空数组，从而导致 while(true) 被不断执行，最后导致某个 CPU 核心的利用率飙升到 100%。
    2.产生原因：归结于linux的内核
     在部分Linux的2.6的kernel中，poll和epoll对于突然中断的连接socket会对返回的eventSet事件集合置为POLLHUP，也可能是POLLERR，eventSet事件集合发生了变化，这就可能导致Selector会被唤醒。
    3. 解决方案：创建一个新的Selector
    Jetty中：
        记录select()返回为0的次数(记做jvmBug次数)，在时间控制范围内，如果jvmBug次数超过指定阈值，则新建一个selector
     Netty中：
        在每次进行 selector.select(timeoutMillis) 时记录开始时间和结束时间，如果持续时间小于timeoutMillis，表明没有阻塞这么长的时间，可能触发了jdk的空轮询bug，
        当空轮询的次数超过一个阀值的时候，默认是512，就开始重建selector。
 ### 1.2 Netty的优点
        1. 入门简单，使用方便，文档齐全，只需依赖JDK
        2. 高性能，高吞吐，低延迟，资源消耗少
        3. 灵活的线程模型，支持阻塞和非阻塞的I/O 模型。
        4. 代码质量高，目前主流版本基本没有 Bug
        
## 2.知识体系
1.核心组件
 
        1. Bootstrap：netty程序编码模式，服务端启动流程
        2. Channel：java NIO封装、网络I/O操作接口、channel设计理念
        3. EventLoop&EventLoopGroup：Reactor线程模型(单线程、多线程、主从Reactor线程模型)、Netty线程模型(推荐主从Reactor多线程模型：Boss线程池实现、worker线程池实现)
        4. ChannelPipeline：链式结构实现、出站/入站概念、事件处理
        5. ChannelHandler：I/O事件生命周期，注册感兴趣的事件、自定义拦截器
2. 拆包/粘包
        
        1. TCP通用的拆包/粘包解决方案；
        2. Netty编码实现：MessageToByteEncoder、ByteToMessageDecoder
        3. Netty拆包/粘包内置方案：行解码器、定长解码器、分隔符解码器、长度域解码器
3. 内存管理
        
        1. 堆外内存
        2. ByteBuf：数据结构设计、核心API接口、分类、引用计数
        3. Netty高性能内存管理设计：linux内存管理基本知识、内存分配器原理、内存池和对象池设计
        4. 零拷贝技术
4. 其他
        
        1. 高性能数据结构：时间轮、无锁队列、FastThreadLocal
        2. 设计模式：单例模式、责任链模式、观察者模式、装饰器模式
## 3.I/O模型
### 3.1 linux主要的I/O模式
I/O 请求可以分为两个阶段，分别为调用阶段和执行阶段：

第一个阶段为I/O 调用阶段：即用户进程向内核发起系统调用。

第二个阶段为I/O 执行阶段：内核等待 I/O 请求处理完成返回。该阶段分为两个过程：首先等待数据就绪，并写入内核缓冲区；随后将内核缓冲区数据拷贝至用户态缓冲区。

![image](https://user-images.githubusercontent.com/41152743/141888549-3af74670-4a62-4357-806f-48ffff5e7777.png)

    1. 同步阻塞I/O(BIO)
        应用进程向内核发起 I/O 请求，发起调用的线程一直等待内核返回结果。只能使用多线程模型，一个请求对应一个线程
![image](https://user-images.githubusercontent.com/41152743/141888602-5f9b5437-4b91-4c63-9c8b-1b8fb2c4b71e.png)

    2. 同步非阻塞 I/O（NIO）
        应用进程向内核发起 I/O 请求后不再会同步等待结果，而是会立即返回，通过轮询的方式获取请求结果。但是轮询过程中大量的系统调用导致上下文切换开销很大。
![image](https://user-images.githubusercontent.com/41152743/141888716-a34f79af-59e9-4b54-8820-548571d6cfc4.png)

    3. I/O 多路复用
        一个线程处理多个 I/O 句柄的操作。多路指的是多个数据通道，复用指的是使用一个或多个固定线程来处理每一个 Socket。
        多个连接会共用一个 Selector 对象，由 Selector 感知连接的读写事件，只需要很少的线程定期从 Selector 上查询连接的读写状态即可。
        select、poll、epoll 都是 I/O 多路复用的具体实现，线程一次 select 调用可以获取内核态中多个数据通道的数据状态；
        在该场景下，当有数据就绪时，需要一个事件分发器（Event Dispather），它负责将读写事件分发给对应的读写事件处理器（Event Handler）
        事件分发器有两种设计模式：Reactor 和 Proactor，Reactor 采用同步 I/O， Proactor 采用异步 I/O
![image](https://user-images.githubusercontent.com/41152743/141888994-7c115d9b-f4c4-4fd3-a3e3-c1234869cc23.png)

    4. 信号驱动 I/O
        半异步的 I/O 模型，在使用信号驱动 I/O 时，当数据准备就绪后，内核通过发送一个 SIGIO 信号通知应用进程，应用进程就可以开始读取数据了
![image](https://user-images.githubusercontent.com/41152743/141889177-aeb61e07-819d-4d22-80af-a66cf2057135.png)

    5. 异步 I/O
        从内核缓冲区拷贝数据到用户态缓冲区的过程也是由系统异步完成，应用进程只需要在指定的数组中引用数据即可
### 3.2 网络框架   
    Netty 和 Tomcat 最大的区别在于对通信协议的支持。
    1. Tomcat 是一个 HTTP Server，它主要解决 HTTP 协议层的传输；
        Netty 不仅支持 HTTP 协议，还支持 SSH、TLS/SSL 等多种应用层的协议，而且能够自定义应用层协议；
    2. Tomcat需要遵循 Servlet 规范，在 Servlet 3.0 之前采用的是同步阻塞模型，Tomcat 6.x 版本之后已经支持 NIO，性能得到较大提升;
       Netty不需要受到Servlet规范的约束，可以最大化发挥 NIO 特性。
    3. 仅需HTTP服务器，推荐使用Tomcat；面向TCP网络的应用开发，推荐使用Netty
### 3.3 逻辑架构
![image](https://user-images.githubusercontent.com/41152743/141937073-0c5c4d29-b918-489d-89c1-d7588e797230.png)

典型网络分层架构设计，共分为网络通信层、事件调度层、服务编排层。

1. 网络通信层

    1.1 BootStrap&ServerBootStrap：负责整个Netty的启动、初始化、服务器连接等过程。
    
        Bootstrap用于客户端引导用于连接远端服务器，只绑定一个 EventLoopGroup。
        ServerBootStrap用于服务端绑定本地端口，绑定两个EventLoopGroup，通常称为 Boss 和 Worker。Boss 会不停地接收新的连接，然后将连接分配给一个个 Worker 处理连接。
    1.2 Channel:供了基本的 API 用于网络 I/O 操作,如 register、bind、connect、read、write、flush 等。AbstractChannel 是整个家族的基类，主要实现类包括：
    
        NioServerSocketChannel(EpollServerSocketChannel)： 异步 TCP 服务端。
        NioSocketChannel： 异步 TCP 客户端。
        OioServerSocketChannel: 同步 TCP 服务端。
        OioSocketChannel: 同步 TCP 客户端。
        NioDatagramChannel: 异步 UDP 连接。
        OioDatagramChannel: 同步 UDP 连接
     channel事件回调状态：
![image](https://user-images.githubusercontent.com/41152743/141942977-e8db6e77-ef33-44af-9dee-eab489738588.png)

具体配置示例如下：

       public void start() {
        synchronized (waitLock) {
            //1. 配置线程池
            initEventPool();
            ServerBootstrap bootstrap = new ServerBootstrap();

           
            bootstrap.group(bossGroup, workGroup)
            //2.channel初始化
             //2.1 设置channel类型
                    .channel(useEpoll() ? EpollServerSocketChannel.class : NioServerSocketChannel.class)
            //2.2 设置channel参数-option 主要负责设置 Boss 线程组
                    .option(ChannelOption.SO_REUSEADDR, nettyParam.isReuseaddr())
                    .option(ChannelOption.SO_BACKLOG, nettyParam.getBacklog())
                    .option(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT)
                    .option(ChannelOption.SO_RCVBUF, nettyParam.getRevbuf())
            //2.3 注册 ChannelHandler
                     .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) {
                            initHandler(ch.pipeline(), nettyParam);
                        }
                    })
            //2.2 设置channel参数， childOption 对应的是 Worker 线程组。
                    .childOption(ChannelOption.TCP_NODELAY, nettyParam.isNodelay())
                    .childOption(ChannelOption.SO_KEEPALIVE, nettyParam.isKeepAlive())
                    .childOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);
            //3. 端口绑定
            bootstrap.bind(UniqueIpUtils.getHost(), nettyParam.getWebPort()).addListener((ChannelFutureListener) channelFuture -> {
                if (channelFuture.isSuccess()) {
                    log.info("服务端启动成");                 
                } else {
                
                }
            });
        }
    }
其中具体的channel属性如下：
![image](https://user-images.githubusercontent.com/41152743/141976727-00909ca9-b4b9-409d-9db4-83e58c124094.png)

    public  void initHandler(ChannelPipeline channelPipeline, NettyParam nettyParam){
        intProtocolHandler(channelPipeline,nettyParam);
       //netty的心跳机制
        /**
         * 在服务器端会根据读空闲时间来检查一下channelRead方法被调用的情况，
         * 如果在该时间内的channelRead()方法都没有被触发，
         * 就会调用userEventTriggered()方法
         */
        channelPipeline.addLast(new IdleStateHandler(readerIdleTimeSeconds,writerIdleTimeSeconds,allIdleTimeSeconds));

        //自定义业务逻辑处理器-文本消息处理器
        channelPipeline.addLast(new DefaultAbstractHandler());
    }
    
     private  void intProtocolHandler(ChannelPipeline channelPipeline, NettyParam nettyParam){
        //http编码器
        channelPipeline.addLast(BootstrapConstant.HTTP_CODE, new HttpServerCodec());

        //HTTP消息聚合
        channelPipeline.addLast(BootstrapConstant.AGGREGATOR, new HttpObjectAggregator(nettyParam.getMaxContext()));

        //分块发送数据，防止发送大文件导致内存溢出
        channelPipeline.addLast(BootstrapConstant.CHUNKED_WRITE,new ChunkedWriteHandler());

        /**
         *   WebSocketServerProtocolHandler处理器
         *   处理了webSocket 协议的握手请求处理，以及 Close、Ping、Pong控制帧的处理
         */
        channelPipeline.addLast(BootstrapConstant.WEB_SOCKET_HANDLER,new WebSocketServerProtocolHandler(nettyParam.getWebSocketPath()));
    }
    
    
2. 事件调度层：负责监听网络连接和读写操作，然后触发各种类型的网络事件

    采用Reactor 线程模型对各类事件进行聚合处理，
    通过 Selector 主循环线程集成多种事件（ I/O 事件、信号事件、定时事件等），
    实际的业务处理逻辑是交由服务编排层中相关的 Handler 完成。
    
    2.1 EventLoopGroup & EventLoop：EventLoopGroup本质是一个线程池，负责接收I/O请求，并分配线程执行请求。
        
        1. 一个 EventLoopGroup 往往包含一个或者多个 EventLoop，EventLoop 用于处理 Channel 生命周期内的所有 I/O 事件，如 accept、connect、read、write 等 I/O 事件。
        2. EventLoop 同一时间会与一个线程绑定，每个 EventLoop 负责处理多个 Channel。
        3. 每新建一个 Channel，EventLoopGroup 会选择一个 EventLoop 与其绑定。该 Channel 在生命周期内都可以对 EventLoop 进行多次绑定和解绑。
    NioEventLoopGroup 也是 Netty 中最被推荐使用的线程模型，与Reactor线程模型的对应：
        
        1. 单线程模型：EventLoopGroup 只包含一个 EventLoop，Boss 和 Worker 使用同一个EventLoopGroup；
        2. 多线程模型：EventLoopGroup 包含多个 EventLoop，Boss 和 Worker 使用同一个EventLoopGroup；
        3. 主从多线程模型：EventLoopGroup 包含多个 EventLoop，Boss 是主 Reactor，Worker 是从 Reactor，它们分别使用不同的 EventLoopGroup，主 Reactor 负责新的网络连接 Channel 创建，然后把 Channel 注册到从 Reactor。
        4. Reactor运行机制：
            连接注册：Channel 建立后，注册至 Reactor 线程中的 Selector 选择器。
            事件轮询：轮询 Selector 选择器中已注册的所有 Channel 的 I/O 事件。
            事件分发：为准备就绪的 I/O 事件分配相应的处理线程。
            任务处理：Reactor 线程还负责任务队列中的非 I/O 任务，每个 Worker 线程从各自维护的任务队列中取出任务异步执行。
            
   2.2. EventLoop：每个 EventLoop 线程都维护一个 Selector 选择器和任务队列 taskQueue。
   
            每当事件发生时，应用程序都会将产生的事件放入事件队列当中，然后 EventLoop 会轮询从队列中取出事件执行或者将事件分发给相应的事件监听者执行。
            事件执行的方式通常分为立即执行、延后执行、定期执行几种。


   2.3 NioEventLoop：主要负责处理 I/O 事件、普通任务和定时任务。
   
        io.netty.util.concurrent.SingleThreadEventExecutor#execute(java.lang.Runnable, boolean)-添加任务
        io.netty.channel.nio.NioEventLoop#run-(事件轮询、事件分发、任务处理)
        
      1. 事件处理机制：无锁串行化的设计思路
 ![image](https://user-images.githubusercontent.com/41152743/142127529-07cb44a0-b173-463e-96a5-82f5d78d8e6a.png)
 
            Channel 生命周期的所有事件处理都是线程独立的，不同的 NioEventLoop 线程之间不会发生任何交集。
            完成数据读取后，会调用绑定的 ChannelPipeline 进行事件传播，依次传递给ChannelHandler，加工完成后传递给下一个，整个过程串行化执行，不会发生线程上下文切换的问题。
        
       缺点：  
            不能执行时间过长的 I/O 操作，一旦某个 I/O 事件发生阻塞，那么后续的所有 I/O 事件都无法执行，甚至造成事件积压
            
      2. 任务处理机制：兼顾执行任务队列中的任务，遵循FIFO规则，保证任务执行的公平性，分为普通任务、定时任务、尾部队列
      
             普通任务：SingleThreadEventExecutor#execute(java.lang.Runnable, boolean)，向任务队列中taskQueue 中添加任务。
                        taskQueue的实现类是多生产者单消费者队列 MpscChunkedArrayQueue，在多线程并发添加任务时，可以保证线程安全
             定时任务：AbstractScheduledEventExecutor#schedule()，向定时任务队列scheduledTaskQueue 添加任务，采用优先队列 PriorityQueue 实现。
             尾部队列：tailTasks相比于普通任务队列优先级较低，在每次执行完 taskQueue 中任务后会去获取尾部队列中任务执行
      3. 最佳实践
            1. 网络连接建立过程中三次握手、安全认证的过程会消耗不少时间，建议采用 Boss 和 Worker 两个 EventLoopGroup，有助于分担 Reactor 线程的压力。
            2. Reactor 线程模式适合处理耗时短的任务场景，对于耗时较长的ChannelHandler可以考虑维护一个业务线程池，将编解码后的数据封装成Task异步处理，
                如果业务逻辑执行时间较短，建议直接在 ChannelHandler 中执行。
            3. 不宜设计过多的 ChannelHandler。          
3. 服务编排层：负责组装各类服务，它是 Netty 的核心处理链，用以实现网络事件的动态编排和有序传播

    3.1 ChannelPipeline：负责组装各种 ChannelHandler，用以实现网络事件的动态编排和有序传播
 ![image](https://user-images.githubusercontent.com/41152743/142147501-d70c123f-9aff-4afe-a548-b61a9a34ee4f.png)
    
        1. 可以理解为ChannelHandler 的实例列表——内部通过双向链表将不同的 ChannelHandler 链接在一起；
        2. 当 I/O 读写事件触发时，ChannelPipeline 会依次调用 ChannelHandler 列表对 Channel 的数据进行拦截和处理；
        3. ChannelPipeline 是线程安全的，因为每一个新的 Channel 都会对应绑定一个新的 ChannelPipeline，一个 ChannelPipeline 关联一个 EventLoop，一个 EventLoop 仅会绑定一个线程。
        4. ChannelPipeline 中包含入站 ChannelInboundHandler 和出站 ChannelOutboundHandler 两种处理器，客户端和服务端一次完整的请求应答过程可以分为三个步骤：
            客户端出站（请求数据）、服务端入站（解析数据并执行业务逻辑）、服务端出站（响应结果）。
        5. ChannelPipeline 的双向链表分别维护了 HeadContext 和 TailContext 的头尾节点，自定义的ChannelHandler在Head和Tail之间。
           1. HeadContext 既是 Inbound 处理器，也是 Outbound 处理器，作为头节点负责读取数据并开始传递 InBound 事件，数据处理完成后，会反方向经过 Outbound 处理器，最终传递到 HeadContext；
           2. TailContext 只实现了 ChannelInboundHandler 接口，用于终止InBound事件传播，作为 OutBound 事件传播的第一站，仅仅是将 OutBound 事件传递给上一个节点。
           3. Inbound 事件的传播方向为 Head -> Tail，而 Outbound 事件传播方向是 Tail -> Head。
    3.2 ChannelHandler & ChannelHandlerContext：数据的编解码以及加工处理操作都是由 ChannelHandler 完成的

        1. ChannelHandler 有两个重要的子接口：ChannelInboundHandler和ChannelOutboundHandler，分别拦截入站和出站的各种 I/O 事件
             ChannelInboundHandler 的事件回调方法与触发时机如下：
![image](https://user-images.githubusercontent.com/41152743/142152317-a1204fd6-e036-4368-b4e1-cf7cd56984a5.png) 

                ChannelOutboundHandler的事件回调方法与触发时机如下：
                
![image](https://user-images.githubusercontent.com/41152743/142153557-45cb157b-503d-4c6b-a718-5967191bf792.png)

        2. ChannelHandler异常处理：
            异常事件的处理顺序与 ChannelHandler 的添加顺序相同，会依次向后传播；
            如果用户没有对异常进行拦截处理，最后将由 Tail 节点统一处理。DefaultChannelPipeline#onUnhandledInboundException
            异常处理最佳实践：在 ChannelPipeline 自定义处理器的末端添加统一的异常处理器。
        3. ChannelHandlerContext 用于保存 ChannelHandler 上下文，通过它可以知道ChannelPipeline 和 ChannelHandler 的关联关系。
        4. ChannelHandlerContext 可以实现 ChannelHandler 之间的交互，包含了 ChannelHandler 生命周期的所有事件

![image](https://user-images.githubusercontent.com/41152743/141957515-bbf2a0aa-52c0-4a60-887d-df3dedf06bd6.png)

### 3.4 拆包/粘包
#### 1.拆包/粘包
       网络通信的过程中，每次可以发送的数据包大小是受多种因素限制的(MTU 最大传输单元、MSS 最大分段大小、滑动窗口)，
        如果一次传输的网络包数据大小超过传输单元大小，那么我们的数据可能会拆分为多个数据包发送出去；
        如果每次请求的网络包数据都很小，一共请求了 10000 次，TCP 并不会分别发送 10000 次。因为 TCP 采用的 Nagle 算法对此作出了优化。
 1. MTU 最大传输单元和 MSS 最大分段大小
 ![image](https://user-images.githubusercontent.com/41152743/142165758-d4e8559c-2f61-439f-a027-fa813c0161e7.png)

        MTU（Maxitum Transmission Unit）：最大传输单元，链路层一次最大传输数据的大小，一般来说大小为 1500 byte
        MSS（Maximum Segement Size）：最大分段大小，指 TCP 最大报文段长度，它是传输层一次发送最大数据的大小。
        计算关系：MSS = MTU - IP 首部 - TCP首部
        拆包：如果MSS + TCP 首部 + IP 首部 > MTU，那么数据包将会被拆分为多个发送，这就是拆包现象。
2.滑动窗口

        TCP 传输层用于流量控制的一种有效措施，是指数据接收方设置的窗口大小，随后接收方会把窗口大小告诉发送方，以此限制发送方每次发送数据的大小，从而达到流量控制的目的。
        因此，数据发送方不需要每发送一组数据就阻塞等待接收方确认，允许发送方同时发送多个数据分组，每次发送的数据都会被限制在窗口大小内；
        TCP 并不会为每个报文段都回复 ACK 响应，它会对多个报文段回复一次 ACK，如果在一定时间范围内未收到某个报文段，将会丢弃其他报文段，发送方会发起重试。
3. Nagle 算法

         TCP/IP 拥塞控制方法，主要用于解决频繁发送小数据包而带来的网络拥塞问题。
         思路：在数据未得到确认之前先写入缓冲区，等待数据确认或者缓冲区积攒到一定大小再把数据包发送出去。
         Linux 在默认情况下是开启 Nagle 算法的，在大量小数据包的场景下可以有效地降低网络开销，Netty 中为了使数据传输延迟最小化，就默认禁用了 Nagle 算法。
4. 拆包/粘包
    原因：
    
        1. 应用程序写入的数据大于套接字缓冲区大小，这将会发生拆包；
        2. 如果MSS + TCP 首部 + IP 首部 > MTU，那么数据包将会被拆分为多个发送，会发生拆包
        3. 应用程序写入数据小于套接字缓冲区大小，将多次写入缓冲区的数据一次发送出去，这将会发生粘包。
        4. 接收方法不及时读取套接字缓冲区数据，这将发生粘包。
    解决方案：提供一种机制来识别数据包的界限，定义应用层的通信协议。
        
        1. 消息长度固定：每个数据报文都需要一个固定的长度，当发送方的数据小于固定长度时，则需要空位补齐。
            缺点：无法很好设定固定长度的值，如果长度太大会造成字节浪费，长度太小又会影响消息传输 
        2. 特定分隔符：在每次发送报文的尾部加上特定分隔符，接收方就可以根据特殊分隔符进行消息拆分
            分隔符的选择一定要避免和消息体中字符相同，以免冲突，通常将消息进行编码，然后选择编码字符之外的字符作为特定分隔符。
            特定分隔符在消息协议足够简单的场景下比较高效，例如redis在通信过程中采用换行分隔符
        3. 消息长度+消息内容：消息头中存放消息的总长度，接收方在解析数据时，首先读取消息头的长度字段 Len，然后紧接着读取长度为 Len 的字节数据，该数据即判定为一个完整的数据报文
            使用方式灵活，此外消息头中还可以自定义其他必要的扩展字段，例如消息版本、算法类型等。
 #### 2.自定义协议通信
    1. 一个完备的网络协议需要具备的基本要素：
    
        1. 魔数：通信双方协商的一个暗号，通常采用固定的几个字节表示，防止任何人随便向服务器的端口上发送数据。接收到数据后会解析出前几个固定字节的魔数，然后做正确性比对。
        2. 协议版本号:随着需求变化，不同版本的协议对应的解析方法也是不同的；
        3. 序列化算法：数据发送方应该采用何种方法将请求的对象转化为二进制，以及如何再将二进制转化为对象；
        4. 报文类型：在不同的业务场景中，报文可能存在不同的类型。
        5. 长度域字段：代表请求数据的长度，接收方根据长度域字段获取一个完整的报文。
        6. 请求数据：通常为序列化之后得到的二进制流，每种请求数据的内容是不一样的。
        7. 状态：标识请求是否正常;
        8. 保留字段: 可选项，为了应对协议升级的可能性，可以预留若干字节的保留字段，以备不时之需
    2. Netty中的编码器和解码器
    
        1. 一次编解码器：MessageToByteEncoder/ByteToMessageDecoder，用于解决TCP拆包/粘包问题；
        2. 二次编解码器：MessageToMessageEncoder/MessageToMessageDecoder，对解析后的字节数据做对象模型的转换
        3. MessageToByteEncoder：io.netty.handler.codec.MessageToByteEncoder#write，编码器实现非常简单，不需要关注拆包/粘包问题。
        4. ByteToMessageDecoder：io.netty.handler.codec.ByteToMessageDecoder#decode，需要传入接收的数据ByteBuf及用来添加编码后消息的 List。
            由于 TCP 粘包问题，ByteBuf 中可能包含多个有效的报文，或者不够一个完整的报文，Netty 会重复回调 decode() 方法，直到没有解码出新的完整报文可以添加到 List 当中；
            或者 ByteBuf 没有更多可读取的数据为止，如果此时 List 的内容不为空，传递给下一个ChannelInboundHandler。
            io.netty.handler.codec.ByteToMessageDecoder#decodeLast：在 Channel 关闭后会被调用一次，主要用于处理 ByteBuf 最后剩余的字节数据
            ReplayingDecoder:ByteToMessageDecoder的抽象子类，封装了缓冲区的管理，在读取缓冲区数据时，无须再对字节长度进行检查，因为没有足够长度的字节数据，ReplayingDecoder 将终止解码操作。















