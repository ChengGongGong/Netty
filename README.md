# Netty
## 1.简介
### 1.1 JDK 原生NIO的API的问题：

    1. Java NIO 的 API 非常复杂，需要熟悉Selector、ServerSocketChannel、SocketChannel、ByteBuffer 等组件；
    2. 开发量和难度增加，核心逻辑较简单，需要补齐可靠性方面的问题，例如：网络波动导致的重连、半包读写等；
    3. JDK自身的bug，Epoll Bug，导致Selector空轮询，cpu使用率达到100%
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
     
        
        
        
        
        
        
