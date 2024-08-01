---
layout:     post
title:      透彻理解 java 网络编程(十)
subtitle:   Bootstrap引导器
date:       2024-07-11
author:     yusihu
header-style: text
catalog: true
tags:
    - 网络编程
    - netty
    - 开源框架
---
通过上一章的讲解，大家应该对 Netty 的基本使用以及整体架构有了初步了解。从本章开始，我将正式分析 Netty 的底层原理并对部分源码进行讲解。

我们在使用 Netty 时，首先就需要使用 Bootstrap（客户端）和 ServerBootstrap（服务端）来对 Netty 中的各类核心组件进行装配。本章，我就对 Bootstrap 和 ServerBootstrap 的底层原理进行讲解。

一、Bootstrap 启动流程
----------------

我们先来回顾一下 Netty 中的 **AbstractBootstrap** 类，它是`Bootstrap`和`ServerBootstrap`的抽象父类，封装了一些公有方法：

![](/img/network-program/netty/BootstrapClass.png)

> 事实上，即时我们不使用 Bootstrap，也可以手动创建 Channel、完成各种设置和启动、注册到 EventLoop，但是整个过程会比较麻烦。所以，通常都是基于 Bootstrap 工具类完成 Netty 核心组件的拼装。

Bootstrap 的启动流程，也就是 Netty 组件的组装、配置，以及 Netty 服务端或客户端的启动流程。本节，我以服务端 ServerBootstrap 的使用为例，讲解 Netty Server 的启动流程，一共可以分为以下几个步骤：

*   配置 EventLoopGroup 线程组；
*   配置 Channel 的类型；
*   设置 ServerSocketChannel 对应的 Handler；
*   设置网络监听的端口；
*   设置 SocketChannel 对应的 Handler；
*   配置 Channel 参数；
*   启动 Netty Server。

![](/img/network-program/netty/BootstrapStart.png)

### 1.1 分配 Reactor 线程池

首先，创建一个服务端的 ServerBootstrap 实例，ServerBootstrap 支持无参构造函数（后续小节分析 ServerBootstrap 源码时我会详细讲解它的内部构造）：

```
    // 创建一个服务端的启动器
    ServerBootstrap bootstrap = new ServerBootstrap();
```

接着，创建两个 Reactor 线程池，并赋值给 ServerBootstrap 启动器实例：

```
    // boss线程池
    EventLoopGroup bossLoopGroup = new NioEventLoopGroup(1);
    // worker线程池
    EventLoopGroup workerLoopGroup = new NioEventLoopGroup(10);
    // 分配线程池
    bootstrap.group(bossLoopGroup, workerLoopGroup);
```

上述这两个 NioEventLoopGroup 线程池，一个负责监听客户端的连接事件并建立连接，名为 bossLoopGroup；另一个负责监听读写 IO 事件和 Handler 业务处理，名为 workerLoopGroup。

事实上，ServerBootstrap 支持在_单线程_、_多线程_和_主从多线程_间切换，也就是也可以只配置一个线程池，并且可以控制线程池中的线程数量。这种灵活的配置方式可以最大程度地满足不同用户的个性化定制需求。

### 1.2 设置父 Channel 类型

Netty 不止支持 Java NIO，也支持阻塞式 IO（也叫 BIO 或 OIO）。下面配置的是 Java NIO 类型的 Channel，方法如下：

```
    bootstrap.channel(NioServerSocketChannel.class);
```

当然，我们可以指定`OioServerSocketChannel.class`等等。

> ServerBootstrap 内部会通过反射的方式创建 NioServerSocketChannel 对象，NioServerSocketChannel 本质是对`java.nio.channels.ServerSocketChannel`的封装。

### 1.3 设置父 Channel 的 Handler

设置 ServerSocketChannel 对应的 Handler：

```
    bootstrap.handler(new LoggingHandler(LogLevel.INFO));    // LoggingHandler是自定义的Handler
```

### 1.4 设置父 Channel 监听端口

底层本质是通过`java.net.ServerSocket`监听端口：

```
    bootstrap.localAddress(new InetSocketAddress(port));
```

### 1.5 配置父 Channel 参数

通过 ServerBootstrap 的`option()`方法，可以对 NioServerSocketChannel 进行参数配置，底层本质是设置 ServerSocket 的参数：

```
    bootstrap.option(ChannelOption.SO_KEEPALIVE, true);
    bootstrap.option(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);
```

### 1.6 配置子 Channel 参数

通过 ServerBootstrap 的`childOption()`方法，可以对每一个 NioSocketChannel 进行参数配置，本质也是设置底层的 Socket 参数：

```
    bootstrap.childOption(ChannelOption.SO_KEEPALIVE, true);
    bootstrap.childOption(ChannelOption.TCP_NODELAY, true);
```

### 1.7 配置子 Channel 的 Pipeline

父 Channel 会负责连接的建立，连接建立完成后都会创建一个新的子 Channel，即 **NioSocketChannel** 。每一个子 Channel 都拥有自己的独立 Pipeline，我们可以对它进行装配：

```
    bootstrap.childHandler(new ChannelInitializer<SocketChannel>() {
        // 建立客户端连接时，会创建一个子通道并初始化
        protected void initChannel(SocketChannel ch) throws Exception {
            // 向子通道的流水线添加Handler业务处理器
            ch.pipeline().addLast(new XXXXHandler());
        }
    });
```

上述`ChannelInitializer.initChannel()`方法会在子 Channel 被创建后调用。另外，父 Channel 也就是 NioServerSocketChannel，也是拥有 Pipeline 的，但是它的处理逻辑是固定的：接受新连接，创建子通道，初始化子通道，所以不需要特别配置。如果我们希望 NioServerSocketChannel 在建立连接后进行特殊的业务处理，可以使用`ServerBootstrap.handler(ChannelHandler handler)`方法，为父通道设置 ChannelInitializer 初始化器，后面分析源码时会详细讲解。

> NioSocketChannel 本质是对`java.nio.channels.SocketChannel`的封装。

### 1.8 绑定并启动

ServerBootstrap 的`bind()`方法会绑定端口并返回一个 ChannelFuture 对象，因为整个过程是异步的，所以调用`ChannelFuture.sync()`同步等待绑定过程的完成：

```
    ChannelFuture channelFuture = bootstrap.bind().sync();
    Logger.info(" 服务器启动成功，监听端口: " + channelFuture.channel().localAddress());
```

至此，Netty Server 启动完成。

> Netty 中的 IO 操作，都会返回异步任务实例（如 ChannelFuture 实例），可以通过阻塞等待或者增加事件监听器两种方式，获得 IO 操作的真正结果。

### 1.9 等待 Channel 关闭

如果要阻塞当前线程直到通道关闭，可以使用`Channel.closeFuture()`方法，获取通道关闭的异步任务。然后调用 sync() 方法，阻塞等待直到通道被关闭：

```
    ChannelFuture closeFuture = channelFuture.channel().closeFuture();
    closeFuture.sync();
```

另外还需要注意，关闭 Channel 后，需要释放 Reactor 线程池资源：

```
    // 释放掉所有资源，包括创建的反应器线程
    workerLoopGroup.shutdownGracefully();
    bossLoopGroup.shutdownGracefully();
```

> 关闭 EventLoopGroup 线程池时，会关闭内部的 EventLoop 线程，也会关闭内部的 Selector 选择器、轮询线程以及所有子通道。在子通道关闭后，会释放掉底层的资源，如 TCP Socket 文件描述符等。

二、ServerBootstrap 源码分析
----------------------

我们已经从使用层面了解了 ServerBootstrap 的启动流程，再来从源码层面分析 ServerBootstrap 的机理。ServerBootstrap 本质是对父类 AbstractBootstrap 的增强，增加了对主从 Reactor 模式的支持。

### 2.1 构造

ServerBootstrap 提供了两种类型的构造器，我们一般都是使用无参构造器：

```
    // ServerBootstrap.java
    
    public ServerBootstrap() { }
    
    private ServerBootstrap(ServerBootstrap bootstrap) {
        super(bootstrap);
        childGroup = bootstrap.childGroup;
        childHandler = bootstrap.childHandler;
        synchronized (bootstrap.childOptions) {
            childOptions.putAll(bootstrap.childOptions);
        }
        childAttrs.putAll(bootstrap.childAttrs);
    }
```

父类 AbstractBootstrap 的构造：

```
    // AbstractBootstrap.java
    
    AbstractBootstrap() {
    }
    
    AbstractBootstrap(AbstractBootstrap<B, C> bootstrap) {
        group = bootstrap.group;
        channelFactory = bootstrap.channelFactory;
        handler = bootstrap.handler;
        localAddress = bootstrap.localAddress;
        synchronized (bootstrap.options) {
            options.putAll(bootstrap.options);
        }
        attrs.putAll(bootstrap.attrs);
    }
```

### 2.2 group 方法

ServerBootstrap 的 group 方法用于分配 EventLoopGroup 线程池，通过方法的重载信息可以看出，Netty 可以在_单线程_、_多线程_和_主从多线程_之间切换：

```
    // ServerBootstrap.java
    
    public class ServerBootstrap extends AbstractBootstrap<ServerBootstrap, ServerChannel> {
    
        private volatile EventLoopGroup childGroup;
    
        @Override
        public ServerBootstrap group(EventLoopGroup group) {
            // 如果只使用一个线程池，则主从Reactor模式退化为单Reactor模式
            return group(group, group);
        }
    
        public ServerBootstrap group(EventLoopGroup parentGroup, EventLoopGroup childGroup) {
            super.group(parentGroup);
            if (this.childGroup != null) {
                throw new IllegalStateException("childGroup set already");
            }
            this.childGroup = ObjectUtil.checkNotNull(childGroup, "childGroup");
            return this;
        }
        //...
    }
```

来看父类 AbstractBootstrap 的`group(EventLoopGroup group)`方法：

```
    // AbstractBootstrap.java
    
    volatile EventLoopGroup group;
    
    public B group(EventLoopGroup group) {
        ObjectUtil.checkNotNull(group, "group");
        if (this.group != null) {
            throw new IllegalStateException("group set already");
        }
        this.group = group;
        return self();
    }
```

也就是说，Main Reactor 线程池本质是在父类中设置的，ServerBootstrap 只是配置子线程池和相关参数。

### 2.3 channel 方法

ServerBootstrap 的 channel 方法继承自父类 AbstractBootstrap：

```
    // AbstractBootstrap.java
    
    public B channel(Class<? extends C> channelClass) {
        return channelFactory(new ReflectiveChannelFactory<C>(ObjectUtil.checkNotNull(channelClass, "channelClass")
        ));
    }
    
    public B channelFactory(ChannelFactory<? extends C> channelFactory) {
        ObjectUtil.checkNotNull(channelFactory, "channelFactory");
        if (this.channelFactory != null) {
            throw new IllegalStateException("channelFactory set already");
        }
    
        this.channelFactory = channelFactory;
        return self();
    }
```

其实就是实例化了一个用于创建 Channel 的工厂，后续 ServerBootstrap 调用 bind 方法时会通过反射创建 NioServerSocketChannel 对象：

```
    // ReflectiveChannelFactory.java
    
    public class ReflectiveChannelFactory<T extends Channel> implements ChannelFactory<T> {
    
        private final Constructor<? extends T> constructor;
    
        public ReflectiveChannelFactory(Class<? extends T> clazz) {
            ObjectUtil.checkNotNull(clazz, "clazz");
            try {
                this.constructor = clazz.getConstructor();
            } 
            //...
        }
    
        @Override
        public T newChannel() {
            try {
                // 通过反射调用Channel的无参构造函数，完成对象实例化
                return constructor.newInstance();
            } 
            //...
        }
    }
```

我们来看下 NioServerSocketChannel 的构造就会明白，它的底层实际是封装了 JDK 的 ServerSocketChannel：

```
    // NioServerSocketChannel.java
    
    public NioServerSocketChannel() {
        this(newSocket(DEFAULT_SELECTOR_PROVIDER));
    }
    
    public NioServerSocketChannel(SelectorProvider provider) {
        this(newSocket(provider));
    }
    
    private static ServerSocketChannel newSocket(SelectorProvider provider) {
        try {
            return provider.openServerSocketChannel();
        } catch (IOException e) {
            throw new ChannelException(
                "Failed to open a server socket.", e);
        }
    }
```

SelectorProvider 是 JDK NIO 中的抽象类实现，通过`openServerSocketChannel()`方法可以创建服务端的 ServerSocketChannel。SelectorProvider 会根据 OS 类型和版本的不同，返回不同的实现类。

### 2.4 option 和 attr 方法

option 用于对父 Channel（即 NioServerSocketChannel）本身的底层 TCP 参数进行配置，而 attr 用于配置父 Channel 的自定义属性（后续可以可以在 Pipineline 中使用这些属性）。

这个两个方法同样是父类 AbstractBootstrap 的方法：

```
    // AbstractBootstrap.java
    
    public abstract class AbstractBootstrap<B extends AbstractBootstrap<B, C>, C extends Channel> implements Cloneable {
        private final Map<ChannelOption<?>, Object> options = new LinkedHashMap<ChannelOption<?>, Object>();
        private final Map<AttributeKey<?>, Object> attrs = new ConcurrentHashMap<AttributeKey<?>, Object>();
    
        public <T> B option(ChannelOption<T> option, T value) {
            ObjectUtil.checkNotNull(option, "option");
            synchronized (options) {
                if (value == null) {
                    options.remove(option);
                } else {
                    options.put(option, value);
                }
            }
            return self();
        }
    
        public <T> B attr(AttributeKey<T> key, T value) {
            ObjectUtil.checkNotNull(key, "key");
            if (value == null) {
                attrs.remove(key);
            } else {
                attrs.put(key, value);
            }
            return self();
        }    
    }
```

AbstractBootstrap 只是将这些配置值保存到内部的字段中，后续初始化 Channel 时再使用它们。

#### AttributeKey

通过`ServerBootstrap.attr()`设置的 Channel 自定义属性，都保存到了 AbstractBootstrap 内部的一个 ConcurrentHashMap 中，这个 Map 的键是 AttributeKey。ServerBootstrap 在后续的初始化 Channel 过程中，会将这些属性值设置到 Channel 中。

Netty 中有一个 AttributeMap 接口，根据接口契约，它的实现类必须是线程安全的，attr 方法的入参就是一个 AttributeKey 对象，泛型用来指明 Value 值类型，返回的是一个 Attribute 对象，内部封装了实际的 Value 值：

```
    public interface AttributeMap {
        /**
         * 获取指定Key对应的Value，Value的类型即泛型T
         */
        <T> Attribute<T> attr(AttributeKey<T> key);
    
        /**
         * 指定的Key是否存在
         */
        <T> boolean hasAttr(AttributeKey<T> key);
    }
```

我们使用 AttributeMap 时，本质把原始 Key 封装成 AttributeKey，原始 Value 封装成 Attribute：

![](/img/network-program/netty/AttributeMap.png)

Netty 中的所有 Channel 都实现了 AttributeMap 接口：

```
    public interface Channel extends AttributeMap, ChannelOutboundInvoker, Comparable<Channel> { }
```

所以，我们可以在自定义的 ChannelHandler 业务处理器中直接使用 AttributeKey，比如：

```
    public class DispatcherHandler extends ChannelInboundHandlerAdapter {
    
        private AttributeKey<String> key = AttributeKey.valueOf("Id");
    
        @Override
        public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
            Attribute<String> channelAttr = ctx.channel().attr(key);
            // 将Value值设置为1
            channelAttr.set("1");
    
            Attribute<String> contextAttr = ctx.attr(key);
            assert contextAttr.get() == "1"
        }
    }
```

注意，在 Netty 4.1 版本以后`ChannelHandlerContext`和`Channel`的`attr`方法设置的属性作用域是完全相同的，也就是说：

```
    Channel.attr() == ChannelHandlerContext.attr()
```

我们可以看下 AbstractChannelHandlerContext 的 attr 方法，内部就是先获取所属的 Channel：

```
    // AbstractChannelHandlerContext.java
    
    public <T> Attribute<T> attr(AttributeKey<T> key) {
        return channel().attr(key);
    }
```

#### ChannelOption

再来看 ChannelOption， ChannelOption 主要是用于配置 Channel 底层的原生参数，比如 TCP 参数等等。这些参数的 Key 已经在 ChannelOption 中以静态变量的方式写死了，我们只能指定其 value 值，如果通过 ChannelOption 设置了一个不存在的 Key，Netty 会以日志形式提示错误信息，但是不会抛出异常。

我们通过`ServerBootstrap.option()`的代码可以看到，ServerBootstrap 会将属于 Channel 的 option 对象和 value 存放到内部的 LinkedHashMap 中。后续当 ServerBootstrap 绑定到具体的端口时，在其`init()`方法当中，会将这个 Map 中的每一项绑定到具体 Channel 中：

```
    // ServerBootstrap.java
    
    @Override
    void init(Channel channel) {
        setChannelOptions(channel, newOptionsArray(), logger);
        //...
    }
```

```
    // AbstractBootstrap.java
    
    static void setChannelOptions(
        Channel channel, Map.Entry<ChannelOption<?>, Object>[] options, InternalLogger logger) {
        // 遍历并为Channel设置参数
        for (Map.Entry<ChannelOption<?>, Object> e: options) {
            setChannelOption(channel, e.getKey(), e.getValue(), logger);
        }
    }
    
    private static void setChannelOption(
        Channel channel, ChannelOption<?> option, Object value, InternalLogger logger) {
        try {
            if (!channel.config().setOption((ChannelOption<Object>) option, value)) {
                logger.warn("Unknown channel option '{}' for channel '{}'", option, channel);
            }
        }
        //...
    }
```

内部调用了 Channel 的 config() 方法，返回一个 ChannelConfig 对象，然后调用该对象的 setOption 方法：

```
    // DefaultServerSocketChannelConfig.java
    
    public <T> boolean setOption(ChannelOption<T> option, T value) {
        validate(option, value);
        // 限定了Option只能是指定范围的类型
        if (option == SO_RCVBUF) {
            setReceiveBufferSize((Integer) value);
        } else if (option == SO_REUSEADDR) {
            setReuseAddress((Boolean) value);
        } else if (option == SO_BACKLOG) {
            setBacklog((Integer) value);
        } else {
            return super.setOption(option, value);
        }
        return true;
    }
```

最后，我们来看下常用的 ChannelOption 有哪些：

**SO_RCVBUF / SO_SNDBUF**  
此为 TCP 参数。每个 TCP socket（套接字）在内核中都有一个发送缓冲区和一个接收缓冲区，这两个选项就是用来设置 TCP 连接的这两个缓冲区大小的。TCP 的全双工的工作模式以及 TCP 的滑动窗口便是依赖于这两个独立的缓冲区及其填充的状态。

**TCP_NODELAY**  
此为 TCP 参数。表示是否立即发送数据，默认值为 True（Netty 默认为 true，而操作系统默认为 false）。该值用于设置是否关闭 Nagle 算法，该算法将小的碎片数据连接成更大的报文（或数据包）来最小化所发送报文的数量，如果需要发送一些较小的报文，则需要禁用该算法。Netty 默认禁用该算法，从而最小化报文传输的延时。

**SO_KEEPALIVE**  
此为 TCP 参数。表示底层 TCP 协议的心跳机制。true 为连接保持心跳，默认值为 false。启用该功能时，TCP 会主动探测空闲连接的有效性。可以将此功能视为 TCP 的心跳机制，需要注意的是：默认的心跳间隔是 7200s，即 2 小时。Netty 默认关闭该功能。

**SO_REUSEADDR**  
此为 TCP 参数。设置为 true 时表示地址复用，默认值为 false。有四种情况需要用到这个参数设置：

1.  当有一个有相同本地地址和端口的 socket1 处于`TIME_WAIT`状态时，而我们希望启动的程序的 socket2 要占用该地址和端口。例如在重启服务且保持先前端口时；
2.  有多块网卡或用 IP Alias 技术的机器在同一端口启动多个进程，但每个进程绑定的本地 IP 地址不能相同；
3.  单个进程绑定相同的端口到多个 socket（套接字）上，但每个 socket 绑定的 IP 地址不同；
4.  完全相同的地址和端口的重复绑定。但这只用于 UDP 的多播，不用于 TCP。

**SO_LINGER**  
此为 TCP 参数。表示关闭 socket 的延迟时间，默认值为 - 1，表示禁用该功能：

*   -1 表示 socket.close() 方法立即返回，但操作系统底层会将发送缓冲区全部发送到对端；
*   0 表示 socket.close() 方法立即返回，操作系统放弃发送缓冲区的数据，直接向对端发送 RST 包，对端收到复位错误。
*   正整数值表示调用 socket.close() 方法的线程被阻塞，直到延迟时间到来、发送缓冲区中的数据发送完毕，若超时，则对端会收到复位错误。

**SO_BACKLOG**  
此为 TCP 参数。表示服务器端接收连接的队列长度，如果队列已满，客户端连接将被拒绝。Linux 中默认为 128。如果连接建立频繁，服务器处理新连接较慢，可以适当调大这个参数。

**SO_BROADCAST**  
此为 TCP 参数。表示设置广播模式。

> 这里提一句，笔者本人非常不喜欢 Netty 的 AttributeKey 和 ChannelOption 的设计（包含很多其它开源框架也有类似问题，比如 Eureka），明明简单的 KV 存储就可以搞定的事情，非得整了一大套框架代码，搞得过于晦涩！其实完全可以像 Kafka 那样只使用常量、枚举、原生 Map 作为配置项，辅以文档说明，清晰简洁。

### 2.5 childOption 和 childAttr 方法

childOption 和 childAttr 这两个方法与上一节中的 option 和 attr 方法类似，只不过它们是作用余子 Channel，也就是 NioSocketChannel：

```
    // ServerBootstrap.java
    
    public class ServerBootstrap extends AbstractBootstrap<ServerBootstrap, ServerChannel> {
    
        private final Map<ChannelOption<?>, Object> childOptions = 
            new LinkedHashMap<ChannelOption<?>, Object>();
    
        private final Map<AttributeKey<?>, Object> childAttrs = 
            new ConcurrentHashMap<AttributeKey<?>, Object>();
    
        public <T> ServerBootstrap childOption(ChannelOption<T> childOption, T value) {
            ObjectUtil.checkNotNull(childOption, "childOption");
            synchronized (childOptions) {
                if (value == null) {
                    childOptions.remove(childOption);
                } else {
                    childOptions.put(childOption, value);
                }
            }
            return this;
        }
    
        public <T> ServerBootstrap childAttr(AttributeKey<T> childKey, T value) {
            ObjectUtil.checkNotNull(childKey, "childKey");
            if (value == null) {
                childAttrs.remove(childKey);
            } else {
                childAttrs.put(childKey, value);
            }
            return this;
        }    
    }
```

ServerBootstrap 只是将这些配置值保存到内部的字段中，后续初始化 Channel 时再使用它们。

### 2.6 childHandler 方法

ServerBootstrap 的 childHandler 方法用于给子 Channel 分配一个业务处理器：

```
    // ServerBootstrap.java
    
    private volatile ChannelHandler childHandler;
    
    public ServerBootstrap childHandler(ChannelHandler childHandler) {
        this.childHandler = ObjectUtil.checkNotNull(childHandler, "childHandler");
        return this;
    }
```

我们在装配 ServerBootstrap 时，一般会使用 **ChannelInitializer** 这个类，关于它的作用我这里简单提一下，后续对 ChannelHandler 源码讲解时再深入说明。

Netty 中的每一个 Channel 通道，都拥有一条自己的 Pipeline 流水线，流水线负责装配自己的 Handler 业务处理器。那么，什么时候进行装配呢？一般是在 Channel 初始化时就完成的。比如下面的示例代码：

```
    // 装配子Channel流水线
    serverBootstrap.childHandler(new ChannelInitializer<SocketChannel>() {
        // 有连接到达时，会创建一个NioSocketChannel
        protected void initChannel(SocketChannel ch) throws Exception {
            // 为这个NioSocketChannel的Pipeline流水线添加一个Handler业务处理器
            ch.pipeline().addLast(new NettyDiscardHandler());
        }
    });
```

### 2.7 bind 方法

ServerBootstrap 的 bind 方法最为复杂，核心流程可以分为四个阶段：

1.  创建 NioServerSocketChannel 对象；
2.  初始化 NioServerSocketChannel 对象，比如设置 Channel 参数，装配 Pipeline 等等；
3.  将 NioServerSocketChannel 注册到 EventLoopGroup 的某个 EventLoop 中；
4.  为 NioServerSocketChannel 执行端口绑定。

整个流程大量运用了 ChannelFuture，所以比较晦涩，我们来通过源码看一下：

```
    // AbstractBootstrap.java
    
    public ChannelFuture bind() {
        validate();
        SocketAddress localAddress = this.localAddress;
        if (localAddress == null) {
            throw new IllegalStateException("localAddress not set");
        }
        return doBind(localAddress);
    }
    
    private ChannelFuture doBind(final SocketAddress localAddress) {
        // 1.创建、初始化、注册Channel
        final ChannelFuture regFuture = initAndRegister();
        final Channel channel = regFuture.channel();
        if (regFuture.cause() != null) {
            return regFuture;
        }
    
        // 2.1 如果注册完成
        if (regFuture.isDone()) {
            // 3.绑定端口
            ChannelPromise promise = channel.newPromise();
            doBind0(regFuture, channel, localAddress, promise);
            return promise;
        } 
        // 2.2 如果没有注册完成
        else {
            final PendingRegistrationPromise promise = new PendingRegistrationPromise(channel);
            // 添加一个回调监听器
            regFuture.addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture future) throws Exception {
                    Throwable cause = future.cause();
                    if (cause != null) {
                        promise.setFailure(cause);
                    } else {
                        // 3.绑定端口
                        promise.registered();
                        doBind0(regFuture, channel, localAddress, promise);
                    }
                }
            });
            return promise;
        }
    }
```

其实核心步骤就是在`initAndRegister`和`doBind0`这两个方法中完成的： initAndRegister() 负责 Channel 初始化和注册，doBind0() 用于端口绑定。

#### initAndRegister

initAndRegister 方法分为三个步骤：

1.  通过 ChannelFactory 工厂创建了 Channel；
2.  对 Channel 进行初始化配置；
3.  从 EventLoopGroup 中选择一个 EventLoop，注册该 Channel。

```
    // AbstractBootstrap.java
    
    final ChannelFuture initAndRegister() {
        Channel channel = null;
        try {
            // 1.创建一个Channel，对于ServerBootstrap来说，一般是NioServerSocketChannel
            channel = channelFactory.newChannel();
            // 2.初始化Channel
            init(channel);
        } catch (Throwable t) {
            //...
        }
        // 3.注册Channel到Selector（每个NioEventLoop内部都有一个java.nio.channels.Selector）
        ChannelFuture regFuture = config().group().register(channel);
        return regFuture;
    }
```

> EventLoopGroup 中的每一个 EventLoop 对象内部都封装了 java.nio.channels.Selector。

我们来看下`ServerBootstrap.init()`方法是如何对 Channel 进行初始化的：

```
    // ServerBootstrap.java
    
    @Override
    void init(Channel channel) {
        // 1.为ServerSocketChannel配置TCP等参数
        setChannelOptions(channel, newOptionsArray(), logger);
        // 2.为ServerSocketChannel配置自定义属性
        setAttributes(channel, newAttributesArray());
    
        ChannelPipeline p = channel.pipeline();
        final EventLoopGroup currentChildGroup = childGroup;
        final ChannelHandler currentChildHandler = childHandler;
        final Entry<ChannelOption<?>, Object>[] currentChildOptions = newOptionsArray(childOptions);
        final Entry<AttributeKey<?>, Object>[] currentChildAttrs = newAttributesArray(childAttrs);
    
        // 3.装配pipeline流水线
        p.addLast(new ChannelInitializer<Channel>() {
            @Override
            public void initChannel(final Channel ch) {
                // 注意：这里的ch就是上面的ServerSocketChannel
                final ChannelPipeline pipeline = ch.pipeline();
                // 这个Handler就是我们通过ServerBootstrap.handler(XXX)装配的
                ChannelHandler handler = config.handler();
                if (handler != null) {
                    // 将自定义的Handler添加到Pipeline中
                    pipeline.addLast(handler);
                }
                // 向ServerSocketChannel（父Channel）所属的EventLoop中提交一个异步任务
                ch.eventLoop().execute(new Runnable() {
                    @Override
                    public void run() {
                        // ServerBootstrapAcceptor用于将建立连接的SocketChannel转发给子Reactor线程池
                        pipeline.addLast(new ServerBootstrapAcceptor(
                            ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
                    }
                });
            }
        });
    }
```

上面的重点是对 ServerSocketChannel 的 pipeline 流水线的装配：

1.  首先，ServerSocketChannel 初始化时，会在流水线中添加一个 ChannelInitializer 处理器。这是一个特殊的 **入站** 处理器，它的 initChannel 方法会在该 ServerSocketChannel 注册完成后被调用；
2.  ChannelInitializer 的 initChannel 方法，会向 Pipieline 添加我们自定义的 Handler；
3.  接着，向所属的 EventLoop 中提交一个异步任务，这个任务的作用就是在 ServerSocketChannel 的流水线中添加一个 ServerBootstrapAcceptor 处理器。ServerBootstrapAcceptor 也是一个特殊的 **入站** 处理器，它的作用就是当建立新的 SocketChannel 连接时，将 SocketChannel 注册到子 Reactor 线程池中的一个 EventLoop 上；
4.  最后，ChannelInitializer 会被从 ServerSocketChannel 的流水线中移除，防止多次执行。

也就是说，当 ServerSocketChannel 初始化并注册完成后，它的 Pipeline 流水线最终只有我们自定义的 Handler 和一个 ServerBootstrapAcceptor 处理器：

![](/img/network-program/netty/Bootstrap-Pipeline.png)

#### doBind0

ServerSocketChannel 的初始化和注册完成后，还需要进行最后一步操作——绑定端口。整个流程的核心操作就是：调用 JDK 底层进行端口绑定，并触发 Pipeline 的`channelActive`事件，把`OP_ACCEPT`事件注册到 Channel 的事件集合中。

```
    // AbstractBootstrap.java
    
    private static void doBind0(
        final ChannelFuture regFuture, final Channel channel,
        final SocketAddress localAddress, final ChannelPromise promise) {
    
        // 向ServerSocketChannel所属的EventLoop中提交一个异步任务
        channel.eventLoop().execute(new Runnable() {
            @Override
            public void run() {
                if (regFuture.isSuccess()) {
                    channel.bind(localAddress, promise).addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
                } else {
                    promise.setFailure(regFuture.cause());
                }
            }
        });
    }
```

上述整个流程是异步的，也就是说只是向 EventLoop（确切说是 NioEventLoop）提交了一个异步任务，EventLoop 内部包含了一个任务队列，以及唯一个工作线程，会不断的从队列取出任务执行。

我们来看实际的端口绑定操作：

```
    // DefaultChannelPipeline.java
    
    @Override
    public final ChannelFuture bind(SocketAddress localAddress, ChannelPromise promise) {
        // 选择Pipeline中的队尾节点进行端口绑定
        return tail.bind(localAddress, promise);
    }
```

很奇怪，端口绑定操作竟然是在 Pipeline 中执行的，而且是选择了尾部的 Handler 执行：

```
    // AbstractChannelHandlerContext.java
    
    @Override
    public ChannelFuture bind(final SocketAddress localAddress, final ChannelPromise promise) {
        //...
        // 从tail开始往前，找到第一个出站的Handler，此时只有ServerBootstrapAcceptor满足
        final AbstractChannelHandlerContext next = findContextOutbound(MASK_BIND);
        EventExecutor executor = next.executor();
        if (executor.inEventLoop()) {
            // 绑定端口
            next.invokeBind(localAddress, promise);
        } else {
            safeExecute(executor, new Runnable() {
                @Override
                public void run() {
                    next.invokeBind(localAddress, promise);
                }
            }, promise, null, false);
        }
        return promise;
    }
    
    private void invokeBind(SocketAddress localAddress, ChannelPromise promise) {
        if (invokeHandler()) {
            try {
                // 调用出站Handler的bind方法
                ((ChannelOutboundHandler) handler()).bind(this, localAddress, promise);
            } catch (Throwable t) {
                notifyOutboundHandlerException(t, promise);
            }
        } else {
            bind(localAddress, promise);
        }
    }
```

最后看 invokeBind 操作，又回到了 DefaultChannelPipeline：

```
    // DefaultChannelPipeline.java
    
    public void bind(
        ChannelHandlerContext ctx, SocketAddress localAddress, ChannelPromise promise) {
        // 关键是这里
        unsafe.bind(localAddress, promise);
    }
```

上面的 unsafe 其实是一个定义在`io.netty.channel.Channel`中的类，封装了 Channel 对底层的 NIO 操作，所以 bind 操作本质就是 ServerSocketChannel 的 bind 操作：

```
    // AbstractChannel.AbstractUnsafe.java
    
    public final void bind(final SocketAddress localAddress, final ChannelPromise promise) {
        assertEventLoop();
    
        if (!promise.setUncancellable() || !ensureOpen(promise)) {
            return;
        }
    
        boolean wasActive = isActive();
        try {
            // 利用原生的NIO ServerSocketChannel完成端口绑定
            doBind(localAddress);
        } catch (Throwable t) {
            safeSetFailure(promise, t);
            closeIfClosed();
            return;
        }
    
        if (!wasActive && isActive()) {
            // 完成端口绑定后，Channel处于Active状态，调用pipeline.fireChannelActive()触发channelActive事件
            invokeLater(new Runnable() {
                @Override
                public void run() {
                    pipeline.fireChannelActive();
                }
            });
        }
    
        safeSetSuccess(promise);
    }
```

最终会执行 NioServerSocketChannel 的 doBind 方法：

```
    // NioServerSocketChannel.java
    
    @Override
    protected void doBind(SocketAddress localAddress) throws Exception {
        // 获取Java NIO中的原生ServerSocketChannel绑定端口
        if (PlatformDependent.javaVersion() >= 7) {
            javaChannel().bind(localAddress, config.getBacklog());
        } else {
            javaChannel().socket().bind(localAddress, config.getBacklog());
        }
    }
```

从上面代码可以看出，绕了这么一大圈，本质就是用 Java NIO 的 ServerSocketChannel 完成端口的绑定。Netty 之所以绕这么一大圈，是因为 **端口绑定** 这一操作在 Netty 里定义为 **出站** 操作，Netty 中 Channel 相关的所有操作都会通过 Pipeline 流水线触发，这也是为什么了在 Pipeline 接口中定义 bind 方法的原因。

此外，端口绑定完成后，会触发 Pipeline 的 channelActive 事件：

```
    // DefaultChannelPipeline.java
    
    public final ChannelPipeline fireChannelActive() {
        // 从head开始触发channelActive事件
        AbstractChannelHandlerContext.invokeChannelActive(head);
        return this;
    }
```

```
    // AbstractChannelHandlerContext.java
    
    static void invokeChannelActive(final AbstractChannelHandlerContext next) {
        EventExecutor executor = next.executor();
        if (executor.inEventLoop()) {
            next.invokeChannelActive();
        } else {
            executor.execute(new Runnable() {
                @Override
                public void run() {
                    next.invokeChannelActive();
                }
            });
        }
    }
```

可以看到，事件从 Head 节点开始触发，执行完 channelActive 事件传播之后，Head 节点会调用`readIfIsAutoRead()`方法触发 Channel 的 read 事件：

```
    // DefaultChannelPipeline.HeadContext.java
    
    public void channelActive(ChannelHandlerContext ctx) {
        // 传播channelActive事件
        ctx.fireChannelActive();
    
        readIfIsAutoRead();
    }
    
    private void readIfIsAutoRead() {
        if (channel.config().isAutoRead()) {
            channel.read();
        }
    }
```

最终调用到 AbstractNioChannel 中的 read() 方法，又从 Pipieline 的 tail 节点开始触发传递`read`事件，注意这个`read`是一个 Outbound 出站事件：

```
    // AbstractNioChannel.java
    
    public Channel read() {
        pipeline.read();
        return this;
    }
```

```
    // DefaultChannelPipeline.java
    
    public final ChannelPipeline read() {
        // 触发传递read出站事件
        tail.read();
        return this;
    }
```

```
    // AbstractChannelHandlerContext.java
    
    public ChannelHandlerContext read() {
        // 获取下一个Outbound Handler
        final AbstractChannelHandlerContext next = findContextOutbound(MASK_READ);
        EventExecutor executor = next.executor();
        if (executor.inEventLoop()) {
            // 触发read事件
            next.invokeRead();
        } else {
            Tasks tasks = next.invokeTasks;
            if (tasks == null) {
                next.invokeTasks = tasks = new Tasks(next);
            }
            executor.execute(tasks.invokeReadTask);
        }
        return this;
    }
```

最终会传递到 head 节点：

```
    // DefaultChannelPipeline.HeadContext.java
    
    public void read(ChannelHandlerContext ctx) {
        unsafe.beginRead();
    }
```

底层最终调用 AbstractNioChannel 的 doBeginRead 方法，：

```
    // AbstractNioChannel.java
    
    protected void doBeginRead() throws Exception {
        final SelectionKey selectionKey = this.selectionKey;
        if (!selectionKey.isValid()) {
            return;
        }
    
        readPending = true;
        final int interestOps = selectionKey.interestOps();
        if ((interestOps & readInterestOp) == 0) {
            selectionKey.interestOps(interestOps | readInterestOp);
        }
    }
```

上述的`readInterestOp`参数就是在前面初始化 ServerSocketChannel 所传入的`SelectionKey.OP_ACCEPT`事件，所以至此 EventLoop 才会开始监听该 ServerSocketChannel 上的`OP_ACCEPT`事件。

### 2.8 ServerBootstrapAcceptor

最后，我们来看下 ServerBootstrapAcceptor 这个 **入站** Handler 处理器，它的作用就是当 NioServerSocketChannel 完成新的 SocketChannel 连接建立后，将这些 Channel 注册到子 Reactor 线程池中：

```
    // ServerBootstrap.java
    
    private static class ServerBootstrapAcceptor extends ChannelInboundHandlerAdapter {
    
        private final EventLoopGroup childGroup;
        private final ChannelHandler childHandler;
        private final Entry<ChannelOption<?>, Object>[] childOptions;
        private final Entry<AttributeKey<?>, Object>[] childAttrs;
        private final Runnable enableAutoReadTask;
    
        ServerBootstrapAcceptor(
            final Channel channel, EventLoopGroup childGroup, ChannelHandler childHandler,
            Entry<ChannelOption<?>, Object>[] childOptions, Entry<AttributeKey<?>, Object>[] childAttrs) {
            this.childGroup = childGroup;
            this.childHandler = childHandler;
            this.childOptions = childOptions;
            this.childAttrs = childAttrs;
            enableAutoReadTask = new Runnable() {
                @Override
                public void run() {
                    channel.config().setAutoRead(true);
                }
            };
        }
    
        @Override
        public void channelRead(ChannelHandlerContext ctx, Object msg) {
            // 1.收到一个新建立连接的SocketChannel
            final Channel child = (Channel) msg;
            // 2.在Channel的pipeline中添加业务处理器
            child.pipeline().addLast(childHandler);
            // 3.配置Channel的参数和自定义属性
            setChannelOptions(child, childOptions, logger);
            setAttributes(child, childAttrs);
    
            try {
                // 4.向子Reactor线程池（也就是子EventLoopGroup）注册该Channel
                childGroup.register(child).addListener(new ChannelFutureListener() {
                    @Override
                    public void operationComplete(ChannelFuture future) throws Exception {
                        if (!future.isSuccess()) {
                            forceClose(child, future.cause());
                        }
                    }
                });
            } catch (Throwable t) {
                forceClose(child, t);
            }
        }
        //...
    }
```

> 服务端 ServerSocketChannel 的 channelRead 事件只会在新连接接入时触发。

ServerBootstrapAcceptor 通过`childGroup.register()`方法，将 NioSocketChannel 注册到 Worker 工作线程中，并注册`OP_READ`事件到 NioSocketChannel 的事件集合。

关于服务端如何处理客户端新建连接的具体源码（`ServerBootstrap.register()`），我这里不贴了，它的内部会调用`pipeline.fireChannelRegistered()`方法传播`channelRegistered`事件，然后再调用 `pipeline.fireChannelActive()`方法传播`channelActive`事件，最终会将`SelectionKey.OP_READ`事件注册到 Channel 的事件集合。

* * *

至此，ServerBootstrap 的源码就分析完了，从源码层面看 Netty Server 的启动流程， 对后续 Netty 的深入使用非常有帮助，我总结一下整个流程：

*   **创建服务端 Channel** ：本质是创建 JDK 底层原生的 Channel，并初始化几个重要的属性，包括 id、unsafe、pipeline 等；
*   **初始化服务端 Channel** ：设置 Socket 参数以及用户自定义属性，并添加两个特殊的处理器 ChannelInitializer 和 ServerBootstrapAcceptor；
*   **注册服务端 Channel** ：调用 JDK 底层将 Channel 注册到 Selector 上；
*   **端口绑定** ：调用 JDK 底层进行端口绑定，并触发 channelActive 事件，把`OP_ACCEPT`事件注册到 Channel 的事件集合中。

> 至于客户端使用的 Bootstrap，底层源码和 ServerBootstrap 是类似的，我就不开展了，读者可以自行阅读 Netty 源码。

三、总结
----

本章，我对 Bootstrap 装配类的启动流程以及 ServerBootstrap 源码进行了深入分析和讲解。最后，来总结一下：

1.  ServerBootstrap 支持主从 Reactor 模式，我们可以配置主从 Reactor 线程池，本质都是 EventLoopGroup 对象；
2.  EventLoopGroup 内部包含了很多 EventLoop 对象，而每个 EventLoop 都封装了 Java NIO 中的 Selector 选择器，同时包含一个工作线程，一个任务队列，一个任务执行器；
3.  ServerBootstrap 会创建并初始化 ServerSocketChannel，然后注册到主 Reactor 线程池内部的一个 EventLoop 中，EventLoop 中的工作线程会监听该 Channel 上的`OP_ACCEPT`事件，当发生该事件时会进行新连接的建立，并将建立成功后新连接——SocketChannel，在 ServerSocketChannel 的 pipeline 中传播处理；
4.  ServerSocketChannel 的 pipeline 中的尾部有一个特殊的 **入站** 处理器——ServerBootstrapAcceptor，当它接收到 SocketChannel 后，会将该 Channel 注册到子 Reactor 线程池中的一个 EventLoop 中，由该 EventLoop 的 pipeline 进行处理。

可以用下面这张图来描述上述的整个流程，里卖弄还有很多关于 NioEventLoopGroup 和 NioEventLoop 的细节，我会在下一章详解讲解：

![](/img/network-program/netty/BootstrapProcess.png)