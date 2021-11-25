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
3. Netty支持的常用解码器
        
        1. 固定长度解码器 FixedLengthFrameDecoder：通过构造函数设置固定长度的大小 frameLength，无论接收方一次获取多大的数据，都会严格按照 frameLength 进行解码；
        2. 特殊分隔符解码器 DelimiterBasedFrameDecoder
            delimiters：指定特殊分隔符，通过写入 ByteBuf 作为参数传入，如果指定的多个分隔符为 \n 和 \r\n，DelimiterBasedFrameDecoder 会退化成使用 LineBasedFrameDecoder 进行解析；
            maxLength：报文最大长度的限制，如果超过 maxLength 还没有检测到指定分隔符，将会抛出 TooLongFrameException
            failFast：设置 failFast 可以控制抛出 TooLongFrameException 的时机，
                如果 failFast=true，那么在超出 maxLength 会立即抛出 TooLongFrameException，不再继续进行解码。
                如果 failFast=false，那么会等到解码出一个完整的消息后才会抛出 TooLongFrameException。
            stripDelimiter：判断解码后得到的消息是否去除分隔符。
        3. 长度域解码器 LengthFieldBasedFrameDecoder：解决 TCP 拆包/粘包问题最常用的解码器
            特有属性：
![image](https://user-images.githubusercontent.com/41152743/142339106-f03aa0f8-8908-4449-a526-9042405d61f6.png)
            与固定长度解码器和特定分隔符解码器相似的属性：
![image](https://user-images.githubusercontent.com/41152743/142339189-fc883439-da5c-41ec-baeb-248e3cdc2eeb.png)
            具体使用示例：io.netty.handler.codec.LengthFieldBasedFrameDecoder
            
                1. 典型的基于消息长度 + 消息内容的解码：
                2. 解码结果需要截断。
                3. 长度字段包含消息长度和消息内容所占的字节。
                4. 基于长度字段偏移的解码。
                5. 长度字段与内容字段不再相邻。
                6. 基于长度偏移和长度修正的解码。
                7. 长度字段包含除 Content 外的多个其他字段。
 3. writeAndFlush 事件传播分析
 
        1. 属于出站操作，从channlPipeline的Tail节点开始进行事件传播，一直向前传播到 Head 节点。
            context.channel().writeAndFlush()->
            io.netty.channel.AbstractChannel#writeAndFlush(java.lang.Object)->
            io.netty.channel.DefaultChannelPipeline#writeAndFlush(java.lang.Object)->
            io.netty.channel.AbstractChannelHandlerContext#write():TailContext
                AbstractChannelHandlerContext 会默认初始化一个 ChannelPromise 完成该异步操作，ChannelPromise 内部持有当前的 Channel 和 EventLoop，
                此外还 ChannelPromise 中注册回调监听 listener 来获得异步操作的结果。
        2. 核心步骤：
            1. findContextOutbound():找到 Pipeline 链表中下一个 Outbound 类型的 ChannelHandler，直到 Head 节点结束。
            2. inEventLoop()：判断当前线程的身份标识，如果当前线程和 EventLoop 分配给当前 Channel 的线程是同一个线程的话，那么所提交的任务将被立即执行；
                否则当前的操作将被封装成一个 Task 放入到 EventLoop 的任务队列，稍后执行；
            3. next.invokeWriteAndFlush(m, promise) ，它会执行下一个 ChannelHandler 节点的 write 方法，重复执行 write 方法，继续寻找下一个 Outbound 节点。
        3. 写Buffer队列：io.netty.channel.DefaultChannelPipeline.HeadContext#write
            1. 数据将会在 Pipeline 中一直寻找 Outbound 节点并向前传播，直到 Head 节点结束，由 Head 节点完成最后的数据发送。       
                首先，对message进行过滤，如果使用的不是 DirectByteBuf，那么它会将 msg 转换成 DirectByteBuf；
                然后，将数据缓存在 ChannelOutboundBuffer 的缓存内，每次传入的数据被封装成一个Entry对象添加到链表中。
                    第一个被写到缓冲区的节点 flushedEntry、第一个未被写到缓冲区的节点 unflushedEntry和最后一个节点 tailEntry。
        4. 刷新Buffer队列：io.netty.channel.DefaultChannelPipeline.HeadContext#flush
            1. 首先将ChannelOutboundBuffer 的缓存内的unflushedEntry数据刷新到flushedEntry中；
            2. io.netty.channel.nio.AbstractNioByteChannel#doWrite，根据设置的自旋次数，将数据真正写入到Socket缓冲区。
### 3.5 内存管理
#### 1. 堆外内存
    1. 堆内内存由 JVM GC 自动回收内存，但是GC 是需要时间开销成本的，堆外内存由于不受 JVM 管理；
    2. 堆外内存需要手动释放，当出现内存泄漏问题时排查起来会相对困难；
    3. 当进行网络 I/O 操作、文件读写时，堆内内存都需要转换为堆外内存，直接使用堆外内存可以减少一次内存拷贝；
    4. 堆外内存可以实现进程之间、JVM 多实例之间的数据共享。
1. Java 中堆外内存的分配方式有两种：ByteBuffer#allocateDirect和Unsafe#allocateMemory
        
        1.ByteBuffer#allocateDirect：调用的是DirectByteBuffer 构造函数，通过 ByteBuffer 分配的堆外内存不需要手动回收，它可以被 JVM 自动回收。
            回收：
                1.  -XX:MaxDirectMemorySize 指定堆外内存的上限大小，当超过其大小时，触发一次Full GC进行清理回收，如果在 Full GC 之后还是无法满足堆外内存的分配，那么程序将会抛出 OOM 异常。
                2.  ByteBuffer.allocateDirect 分配的过程中，在 Bits.reserveMemory 方法中也会主动调用 System.gc() 强制执行 Full GC，但是在生产环境一般都是设置了 -XX:+DisableExplicitGC，System.gc() 是不起作用的。
           回收原理：
               1. DirectByteBuffer初始化时,包含堆外内存的地址、大小以及 Cleaner 对象的引用。其中 Cleaner 对象是虚引用PhantomReference 的子类，配合引用队列ReferenceQueue 联合使用;
               2. 当发生 GC 时，DirectByteBuffer 对象被回收，此时 Cleaner 对象不再有任何引用关系；
               3. 此时Cleaner对象会被JVM挂到PendingList上，然后有一个固定的线程扫描这个List，遇到Cleaner对象就执行 clean() 方法，将该将 Cleaner 对象从 Cleaner 链表中移除，然后调用unsafe.freeMemory 方法清理堆外内存。
                
        2.Unsafe#allocateMemory ：所分配的内存必须自己手动释放，否则会造成内存泄漏，unsafe.freeMemory(address)。
2. Netty分配堆外内存的方式：ByteBuf
    
        1.  ByteBuffer 的基本属性:mark <= position <= limit <= capacity。
                mark：为某个读取过的关键位置做标记，方便回退到该位置；
                position：当前读取的位置；
                limit：buffer 中有效的数据长度大小；
                capacity：初始化时的空间容量。
           缺点：
                1. 分配的长度是固定的，无法动态扩缩容；
                2. 只能通过 position 获取当前可操作的位置，需要频繁调用 flip、rewind 方法切换读写状态
          ByteBuf的优势：
                1. 容量可以按需动态扩展，类似于 StringBuffer；
                2. 读写采用了不同的指针，读写模式可以随意切换；
                3. 通过内置的复合缓冲类型可以实现零拷贝；
                4. 支持引用计数、缓存池
        2. ByteBuf简介
            1. 内部结构:读指针 readerIndex、写指针 writeIndex、最大容量 maxCapacity
   ![image](https://user-images.githubusercontent.com/41152743/142405657-364957d3-1b80-49ae-b01d-b823e1b99736.png)
            
                废弃字节：已经丢弃的无效字节数据；
                可读字节：通过 writeIndex - readerIndex 计算，表示可以被读取的字节内容，当 readerIndex == writeIndex 时，表示 ByteBuf 已经不可读。
                可写字节: 写入数据都会存储到可写字节区域，当 writeIndex 超过 capacity，表示 ByteBuf 容量不足，需要扩容。
                可扩容字节：最多还可以扩容多少字节，超过 maxCapacity 再写入就会出错。
           2. 引用计数
                实现了 ReferenceCounted 接口，ByteBuf 的生命周期是由引用计数所管理。
                只要引用计数大于 0，表示 ByteBuf 还在被使用；当 ByteBuf 不再被其他对象所引用时，引用计数为 0，那么代表该对象可以被释放。
                此外，当引用计数为 0，该 ByteBuf 可以被放入到对象池中，避免每次使用 ByteBuf 都重复创建；
                可以利用引用计数的特点实现内存泄漏检测工具，Netty 会对分配的 ByteBuf 进行抽样分析，检测 ByteBuf 是否已经不可达且引用计数大于 0，判定内存泄漏的位置并输出到日志中，
                    需要关注日志中 LEAK 关键字。
           3. 分类
                1. Heap/Direct 就是堆内和堆外内存：Heap 指的是在 JVM 堆内分配，底层依赖的是字节数据，Direct 则是堆外内存，不受 JVM 限制，分配方式依赖 JDK 底层的 ByteBuffer。
                2. Pooled/Unpooled 表示池化还是非池化内存：
                    Pooled 是从预先分配好的内存中取出，使用完可以放回 ByteBuf 内存池，等待下一次分配；
                    Unpooled 是直接调用系统 API 去申请内存，确保能够被 JVM GC 管理回收。
                3. Unsafe/非 Unsafe 的区别在于操作方式是否安全：
                    Unsafe 表示每次调用 JDK 的 Unsafe 对象操作物理内存，依赖 offset + index 的方式操作数据；
                    非 Unsafe 则不需要依赖 JDK 的 Unsafe 对象，直接通过数组下标的方式操作数据。
           4. 核心API
                1. 指针操作 API
                    readerIndex() & writeIndex()：返回当前读写指针的位置；
                    markReaderIndex() & resetReaderIndex()：用于保存 readerIndex 的位置、将当前 readerIndex 重置为之前保存的位置。
                2. 数据读写 API
                    isReadable()、readableBytes()等
                3. 内存管理API
                    release() & retain()：引用计数的增减；
                    slice() & duplicate()：前者默认截取 readerIndex 到 writerIndex 之间的数据，后者截取的是整个原始 ByteBuf 信息
                    copy()：从原始的 ByteBuf 中拷贝所有信息，所有数据都是独立的
 #### 2. 内存分配器
 内部碎片：内存是按 Page 进行分配的，每个大小为4K，如果需要很小的内存，也会分配 4K 大小的 Page，单个 Page 内只有一部分字节都被使用，剩余的字节形成了内部碎片。
 
 外部碎片：分配较大内存块时产生的，操作系统只能通过分配连续的 Page 才能满足要求，这些 Page 被频繁的回收并重新分配，Page 之间就会出现小的空闲内存块。
 
1. 常用内存分配器
    
        ptmalloc ：基于 glibc 实现的内存分配器，它是一个标准实现，兼容性较好。但是多线程之间内存无法实现共享，只能每个线程都独立使用各自的内存，在内存开销上比较浪费；
        tcmalloc(thread-caching malloc)：为每个线程分配了一个局部缓存，对于小对象的分配，可以直接由线程局部缓存来完成，对于大对象的分配场景，尝试采用自旋锁来减少多线程的锁竞争问题。
        jemalloc：带有线程缓存，将内存分配粒度划分为 Small、Large、Huge 三个分类，并记录了很多 meta 数据，所以空间占用上要略多于 tcmalloc，在大内存分配场景下，内存碎片要少于 tcmalloc。 
![image](https://user-images.githubusercontent.com/41152743/142984535-722de2dd-33e0-4bd0-b137-aae1443154a2.png)
        jemalloc的核心概念：
            
            1. tcache ：每个线程私有的缓存，用于 small 和 large 场景下的内存分配。每个 tcahe 会对应一个 arena，有一个bin 数组，称为tbin。分配内存时，优先从线程中对应的 tcache 中进行分配，分配失败，则找到对应的arena进行内存分配。
            2. arena： 内存由一定数量的 arenas 负责管理，每个用户线程都会被绑定到一个 arena 上，默认每个CPU分配4个arena。
            3. bin：每个 arena 都包含一个 bin 数组，用于管理不同档位的内存单元。
            4. chunk：每个 arena 被划分为若干个 chunks，负责管理用户内存块的数据结构，以 Page 为单位管理内存，默认大小是 4M，即 1024 个连续 Page。每个 chunk 可被用于多次小内存的申请，但是在大内存分配的场景下只能分配一次。
            5. run：每个 chunk 包含若干个 runs，每个run由连续的Page组成，每个 bin 管理相同类型的 run，run 结构具体的大小由不同的 bin 决定。run 才是实际分配内存的操作对象；
            6. region：每个 run 会被划分为一定数量的 regions，在小内存的分配场景，region 相当于用户内存；
        
2. 常用内存分配器算法
        
    1. 动态内存分配（Dynamic memory allocation）：堆内存分配，简称 DMA，根据程序运行过程中的需求即时分配内存，且分配的内存大小就是程序需求的大小。
        
        从一整块内存中按需分配，对于分配出的内存会记录元数据，同时还会使用空闲分区链维护空闲内存，以地址递增的顺序将空闲分区以双向链表的形式连接在一起，查找空闲内存的策略如下：
        
        1.首次适应算法（first fit）：从空闲分区链中找到第一个满足分配条件的空闲分区，然后划分出一块可用内存给请求进程，剩余的空闲分区仍然保留在空闲分区链中。每次都从低地址开始查找，造成低地址部分会不断被分配，同时也会产生很多小的空闲分区。
        2.循环首次适应算法：从上次找到的空闲分区的下⼀个空闲分区开始查找，比⾸次适应算法空闲分区的分布更加均匀，但是会造成空闲分区链中大的空闲分区会越来越少。
        3.最佳适应算法（best fit）：空闲分区链以空闲分区大小递增的顺序以双向链表的形式连接在一起，每次从空闲分区链的开头进行查找。每次请求后，重新按分区大小进行排序，会有性能损耗问题，空间利用率更高，同样也会留下很多较难利用的小空闲分区。
    2. 伙伴算法
        
        采用分离适配的设计思想，将物理内存按照 2 的次幂进行划分，内存分配时也是按照 2 的次幂大小进行按需分配。效地减少了外部碎片，但是有可能会造成非常严重的内部碎片，最严重的情况会带来 50% 的内存碎片。具体分配过程如下，例如需要分配10K大小的内存块：
        
            1. 首先需要找到存储 2^4 连续 Page 所对应的链表，
            2. 查找 2^4 链表中是否有空闲的内存块，如果有则分配成功；
            3. 如果不存在，则继续沿数组向上查找，链表中每个节点存储 2^5 的连续 Page；
            4. 如果 2^5 链表中存在空闲的内存块，则取出该内存块并将它分割为 2 个 2^4 大小的内存块，其中一块分配给进程使用，剩余的一块链接到 2^4 链表中。
        归还内存流程如下：
        
            1. 首先检查其伙伴块的内存是否释放，(大小相同，而且两个块的地址是连续的，其中低地址的内存块起始地址必须为 2 的整数次幂)。
            2. 如果是空闲的，则将两个内存合并成更大的块，重复上述伙伴块机制的检查，直至伙伴块不是空闲的
            3. 最后将该内存块按照实际大小归还到对应的链表中；
            4. 频繁的合并会造成CPU浪费，当链表中的内存块个数大于某个阈值时，才会触发合并操作。
    3. Slab 算法
        
        在伙伴算法的基础上，对小内存的场景专门做了优化，采用了内存池的方案，解决内部碎片问题。
        
        Linux内核采用的是该算法，提供了高速缓存机制，使用缓存存储内核对象，当内核需要分配内存时，基本上可以通过缓存中获取。
        
        还可以支持通用对象的初始化操作，避免对象重复初始化的开销。
        
        具体逻辑：
        
            1. 维护着大小不同的 Slab 集合。在最顶层是 cache_chain，cache_chain 中维护着一组 kmem_cache 引用，kmem_cache 负责管理一块固定大小的对象池。
            2. 通常会提前分配一块内存，然后将这块内存划分为大小相同的 slot，不会对内存块再进行合并，同时使用位图 bitmap 记录每个 slot 的使用情况。
            3. kmem_cache 中包含三个 Slab 链表：完全分配使用 slab_full、部分分配使用 slab_partial和完全空闲 slabs_empty
            4. 每个链表中维护的 Slab 都是一个或多个连续 Page，每个 Slab 被分配多个对象进行存储
            5. 释放内存时不会丢弃已经分配的对象，而是将它保存在缓存中，当下次再为对象分配内存时，直接会使用最近释放的内存块。
            6. 单个 Slab 可以在不同的链表之间移动。
 #### 3. Netty高性能内存管理
 1. 内存规格
 ![image](https://user-images.githubusercontent.com/41152743/142986435-262073c4-8743-4b1a-b8aa-f2ddad07f2dc.png)
    
    Tiny：0 ~ 512B 之间的内存块，Subpage最小的划分单位为 16B，按 16B 依次递增，16B、32B、48B ...... 496B
    
    Small：512B ~ 8K 之间的内存块，Subpage划分为512B、1024B、2048B、4096B四种 
    
    Normal： 8K ~ 16M 的内存块
    
    Huge：代表大于 16M 的内存块，直接使用非池化的方式进行内存分配。
    
    在每个区域内定义了更细粒度的内存分配单位，包括Chunk、Page、Subpage。
        
            1. Subpage：负责 Page 内的内存分配，将 Page 划分为多个相同的子块进行分配，根据不同的规格进行不同的划分。
            2. Page：Chunk 用于管理内存的单位，Netty 中的 Page 的大小为 8K。
            3. Chunk ：Netty 向操作系统申请内存的单位，以理解为 Page 的集合，每个 Chunk 默认大小为 16M。
2. 内存池架构设计
    
    1. PoolArea：采用固定数量的多个 Arena 进行内存分配，Arena 的默认数量与 CPU 核数有关。
 ![image](https://user-images.githubusercontent.com/41152743/142994043-dd82a1ff-c780-46c2-9576-4b3375ab0ab4.png)
        
        1. PoolSubpage 数组：用于分配小于 8K 的内存，存放Tiny 和 Small 类型的内存块，采用向上取整的方式分配节点进行分配；
            
            1. PoolSubpage 通过位图 bitmap 记录子内存是否已经被使用，bit 的取值为 0 或者 1;
            2. PoolArena 在创建是会初始化smallSubpagePools，分配内存时从PoolChunk中找到一个 PoolSubpage 节点,并进行等分8k/分配的内存，然后找到这个节点对应的PoolArea，
            将这个 PoolSubpage 节点与smallSubpagePools[1]对应的head节点连接组成双向链表，下次再分配同样规格的内存时，查找PoolArea中的smallSubpagePools是否存在可用的PoolSubpage。
        2. PoolChunkList：用于存储不同利用率的Chunk，构成一个双向循环链表。
        
            qInit，内存使用率为 0 ~ 25% 的 Chunk，用于存储初始化分配的PoolChunk,即使内存被完全释放也不会被回收，避免PoolChunk的重复初始化工作。
            q000，内存使用率为 1 ~ 50% 的 Chunk。
            q025，内存使用率为 25% ~ 75% 的 Chunk。
            q050，内存使用率为 50% ~ 100% 的 Chunk。
            q075，内存使用率为 75% ~ 100% 的 Chunk。
            q100，内存使用率为 100% 的 Chunk。
            
            1. 在分配大于 8K 的内存时，其链表的访问顺序是 q050->q025->q000->qInit->q075，优先选择q050，是因为使PoolChunk 的使用率范围保持在中间水平，降低了 PoolChunk 被回收的概率。
            2. 每个 PoolChunkList 都有内存使用率的上下限：minUsage 和 maxUsage，如果使用率超过 maxUsage，那么会从当前 PoolChunkList 移除，并移动到下一个；
                如果使用率小于 minUsage，那么 PoolChunk 会从当前 PoolChunkList 移除，并移动到前一个。
            3.如果 PoolChunk 的使用率一直处于临界值，会导致 PoolChunk 在两个 PoolChunkList 不断移动，造成性能损耗。
   2. PoolChunk：真正存储内存数据的地方，每个 PoolChunk 的默认大小为 16M，Netty的内存分配和回收基于 PoolChunk 完成的
        
            1. 理解为 Page 的集合，每个子内存块采用 PoolSubpage 表示。
            2. 使用伙伴算法将 每个PoolChunk 分配成 2048 个 Page，最终形成一颗满二叉树，二叉树中所有子节点的内存都属于其父节点管理
   3. PoolThreadCache & MemoryRegionCache
            
            1. PoolThreadCache：本地线程缓存，缓存 Tiny、Small、Normal 三种类型的数据
                内存释放时，并没有将缓存归还给 PoolChunk，而是使用 PoolThreadCache 缓存起来，当下次有同样规格的内存分配时，直接从 PoolThreadCache 取出使用即可。
            2. MemoryRegionCache：实际上是一个队列，当内存释放时，将内存块加入到队列中，下次再分配同样规格的内存时，直接从队列中取出空闲的内存块。
3. 内存分配实现原理

io.netty.buffer.PoolChunk#allocate：

    1. 分配内存大于 8K 时，PoolChunk 中采用的 Page 级别的内存分配策略。
    2. 分配内存小于 8K 时，由 PoolSubpage 负责管理的内存分配策略。
    3. 分配内存小于 8K 时，为了提高内存分配效率，由 PoolThreadCache 本地线程缓存提供的内存分配。io.netty.buffer.PoolArena#allocate(io.netty.buffer.PoolThreadCache, io.netty.buffer.PooledByteBuf<T>, int)
    
 4. 内存回收实现原理
 
 io.netty.buffer.PoolThreadCache#allocate
 
    当用户线程释放内存时会将内存块缓存到本地线程的私有缓存 PoolThreadCache 中，这样在下次分配内存时会提高分配效率，但是当内存块被用完一次后，再没有分配需求，那么一直驻留在内存中又会造成浪费。 
    
    1. 默认每执行 8192 次 allocate()，就会调用一次 trim() 进行内存整理;
    2. Netty 在线程退出的时候还会回收该线程的所有内存,PoolThreadCache 重载了 finalize() 方法，在销毁前执行缓存回收的逻辑.

#### 4. 对象池Recycler
1. Recycler简介

    1. 当需要某个对象时，优先从对象池中获取对象实例。通过重用对象，能避免频繁地创建和销毁所带来的性能损耗，而且对 JVM GC 友好。
    2. Recycler 是 Netty 提供的自定义实现的轻量级对象回收站;
    3. 内部结构：Stack、WeakOrderQueue、Link、DefaultHandle
![image](https://user-images.githubusercontent.com/41152743/143019229-5ca725e2-40ca-4eca-abbc-6f55797b3d88.png)
        1. Stack：用于存储当前本线程回收的对象，在多线程的场景下，Netty 为了避免锁竞争问题，每个线程都会持有各自的对象池，内部通过 FastThreadLocal 来实现每个线程的私有化。
![image](https://user-images.githubusercontent.com/41152743/143430463-8ba2b443-2101-4355-aa02-cf3ea693503f.png)
        2. WeakOrderQueue：用于存储其他线程回收到当前线程所分配的对象，例如：ThreadB 回收到 ThreadA 所分配的内存时，就会被放到 ThreadA 的 WeakOrderQueue 当中。
        3. Link：每个 WeakOrderQueue 中都包含一个 Link 链表，回收对象都会被存在 Link 链表中的节点上，每个 Link 节点默认存储 16 个对象。
        4. DefaultHandle：保存了实际回收的对象。
    4. 具体原理
    
        io.netty.util.Recycler#get：获取对象的主流程：
            当 Stack 中 elements 有数据时，直接从栈顶弹出；
            当 Stack 中 elements 没有数据时，尝试从 WeakOrderQueue 中回收一个 Link 包含的对象实例到 Stack 中，然后从栈顶弹出。
    5.  对象回收原理
        
        io.netty.util.Recycler.DefaultHandle#recycle：分为同线程回收(当前线程回收自己分配的对象)、异线程回收
        1. 同线程回收直接向 Stack 中添加对象，异线程回收向 WeakOrderQueue 中的 Link 添加对象。
        2. 对象回收都会控制回收速率，每 8 个对象会回收一个，其他的全部丢弃。








    
