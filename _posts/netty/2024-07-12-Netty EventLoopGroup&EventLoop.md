---
layout:     post
title:      透彻理解 java 网络编程(十一)
subtitle:   EventLoopGroup 和 EventLoop
date:       2024-07-12
author:     yusihu
header-style: text
catalog: true
tags:
    - 网络编程
    - netty
    - 开源框架
---
Netty 服务端通过 ServerBootstrap 启动的时候，会创建两个 EventLoopGroup，它们本质是两个 Reactor 线程池：一个用于与客户端建立 TCP 连接，另一个用于处理 IO 相关的读写操作。下图是 ServerBootstrap 启动时，EventLoopGroup 的核心工作流程图：

![](/img/network-program/netty/EventLoopGroupWorkProcess.png)

EventLoopGroup 属于 Netty 最核心的接口之一，Netty 默认提供了 NioEventLoopGroup、OioEventLoopGroup 等多种实现。而 EventLoop 则是 EventLoopGroup 内部的事件处理器，每个 EventLoop 内部都有一个线程、一个 Java NIO Selector 和一个 taskQueue 任务队列，负责处理客户端请求和内部任务等。

本章，我就对 EventLoopGroup 和 EventLoop 的核心原理进行分析讲解。这里补充一句，如果觉得干巴巴看源码难度比较大，完全可以通过调试的方式跟踪，比如本章，我们可以自己创建 EventLoopGroup，注册 Channel：

```
    public class Demo {
        public static void main(String[] args) throws InterruptedException {
            // 1.创建一个ServerSocketChannel
            ChannelFactory channelFactory = new ReflectiveChannelFactory(NioServerSocketChannel.class);
            Channel channel = channelFactory.newChannel();
            // 2.初始化Pipeline
            channel.pipeline().addLast(new ChannelInitializer<Channel>() {
                @Override
                public void initChannel(final Channel ch) {
                    System.out.println("Chanel:" +ch);
                }
            });
            // 3.创建EventLoopGroup
            EventLoopGroup mainGroup = new NioEventLoopGroup(2);
            // 4.注册Channel
            ChannelFuture future = mainGroup.register(channel);
            future.addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture future) throws Exception {
                    System.out.println("future:" + future);
                }
            });
            Thread.sleep(60 * 3600 * 1000);
        }
    }
```

一、继承体系
------

在正式分析 EventLoopGroup 和 EventLoop 之前，我们需要先从全局了解下 Netty 的任务调度框架体系：

![](/img/network-program/netty/EventLoopGroup-EventLoop-Class.png)

从上图可以看到，Netty 的任务调度框架还是遵循了 `java.util.concurrent` 中的 Executor 调度体系：

*   `io.netty.util.concurrent`是 Netty 对 J.U.C 并发包功能的扩充，相当于 Netty 基于 J.U.C 实现了一套自己的任务调度基础体系；
*   `io.netty.channel`是 Netty 基于自己的任务调度框架，实现的一套 Channel 调度组件。

由于我们一般只使用 Netty 进行 NIO 网络编程，所以重点关注包`io.netty.channel`中的 EventLoopGroup 和 EventLoop 即可，它们是负责对 Channel 事件进行调度的处理器。

二、EventLoopGroup
----------------

前面说了，EventLoopGroup 本质是一个线程池。既然是线程池，就可以调整内部的线程数量，默认情况下，EventLoopGroup 中的线程数是 CPU 核数 * 2。

EventLoopGroup 内部管理着很多 EventLoop 对象，这些对象可以看成是 EventLoopGroup 内的线程，EventLoop 是在 EventLoopGroup 对象构造时创建的。每一个 EventLoop 负责处理一系列的 Channel，我们可以通过下图理解 EventLoopGroup 和 EventLoop 之间的关系：

![](/img/network-program/netty/EventLoopGroup-EventLoop-Channel.png)

Netty 的事件处理机制采用的是 **无锁串行化的设计思路** ：

- **BossEventLoopGroup** 和 **WorkerEventLoopGroup** 包含一个或者多个 NioEventLoop。BossEventLoopGroup 负责监听客户端的 Accept 事件，当事件触发时，将事件注册至 WorkerEventLoopGroup 中的一个 NioEventLoop 上。每新建一个 Channel， 只选择一个 NioEventLoop 与其绑定。所以说 Channel 生命周期的所有事件处理都是 **线程独立** 的，不同的 NioEventLoop 线程之间不会发生任何交集；  
- NioEventLoop 完成数据读取后，会调用绑定的 ChannelPipeline 进行事件传播，ChannelPipeline 也是 **线程安全** 的，数据会被传递到 ChannelPipeline 的第一个 ChannelHandler 中。数据处理完成后，将加工完成的数据再传递给下一个 ChannelHandler，整个过程是 **串行化** 执行，不会发生线程上下文切换的问题。

> 无锁串行化的设计不仅使系统吞吐量达到最大化，而且降低了用户开发业务逻辑的难度，不需要花太多精力关心线程安全问题。虽然单线程执行避免了线程切换，但是它的缺陷就是不能执行时间过长的 I/O 操作，一旦某个 I/O 事件发生阻塞，那么后续的所有 I/O 事件都无法执行，甚至造成事件积压。在使用 Netty 进行程序开发时，我们一定要对 ChannelHandler 的实现逻辑有充分的风险意识。

本节，我主要分析 EventLoopGroup 的实现类 NioEventLoopGroup 的源码：

![](/img/network-program/netty/NioEventLoop-Class.png)

### 2.1 创建 NioEventLoopGroup

我们先来看 NioEventLoopGroup 的构造，NioEventLoopGroup 对象的创建流程可以用下面这张时序图表示：

![](/img/network-program/netty/NioEventLoopGroup-Process.png)

NioEventLoopGroup 本身提供了很多构造器，我们重点关注下面这个即可：

```
    // NioEventLoopGroup.java
    
    public NioEventLoopGroup(int nThreads, Executor executor, EventExecutorChooserFactory chooserFactory,
                             final SelectorProvider selectorProvider,
                             final SelectStrategyFactory selectStrategyFactory,
                             final RejectedExecutionHandler rejectedExecutionHandler,
                             final EventLoopTaskQueueFactory taskQueueFactory) {
        super(nThreads, executor, chooserFactory, selectorProvider, selectStrategyFactory,
              rejectedExecutionHandler, taskQueueFactory);
    }
```

构造时可以指定：

*   _nThreads_：内部的 NioEventLoop 数量，如果不指定则为 CPU 核数的 2 倍；
*   _Executor_：J.U.C 中的任务执行器，每一个 EventLoop 对象内部都会包含一个 Executor，默认为 Netty 自定义的 ThreadPerTaskExecutor，即为每个任务创建一个线程处理；
*   _EventExecutorChooserFactory_：用于创建 Chooser 对象的工厂类，Chooser 可以看成一个负载均衡器，用于从 NioEventLoopGroup 中选择一个 EventLoop，默认采用 round-robin 算法；
*   _SelectorProvider_：Java NIO 提供的工具类，SelectorProvider 使用了 JDK 的 SPI 机制来创建 Selector、ServerSocketChannel、SocketChannel 等对象；
*   _SelectStrategyFactory_：用来创建 SelectStrategy 的工厂，SelectStrategy 是 Netty 用来控制 EventLoop 轮询方式的策略，默认为 DefaultSelectStrategy；
*   _RejectedExecutionHandler_：线程池的任务拒绝策略，默认是抛出 RejectedExecutionException 异常；
*   _EventLoopTaskQueueFactory_：任务队列工厂类，每一个 EventLoop 对象内部都会包含一个任务队列，这个工厂类就是用来创建队列的，默认会创建一个 Netty 自定义线程安全的 MpscUnboundedArrayQueue 无锁队列。

最终调用了父类 MultiThreadEventExecutorGroup 的构造方法，它内部维护了一个 _children_ 数组，保存 EventLoop 对象：

```
    // MultiThreadEventExecutorGroup.java
    
    protected MultithreadEventExecutorGroup(int nThreads, Executor executor,
                                            EventExecutorChooserFactory chooserFactory, Object... args) {
        checkPositive(nThreads, "nThreads");
        if (executor == null) {
            // 创建一个任务执行器
            executor = new ThreadPerTaskExecutor(newDefaultThreadFactory());
        }
        // 创建内部的EventLoop数组，nThreads默认为CPU核数*2
        children = new EventExecutor[nThreads];
    
        for (int i = 0; i < nThreads; i ++) {
            boolean success = false;
            try {
                // 创建EventLoop对象
                children[i] = newChild(executor, args);
                success = true;
            } catch (Exception e) {
                throw new IllegalStateException("failed to create a child event loop", e);
            } finally {
                if (!success) {
                    //...
                }
            }
        }
        // Chooser本质可以看成一个负载均衡器，用于选择一个内部的EventLoop，默认采用round-robin算法
        chooser = chooserFactory.newChooser(children);
       //...
    }
```

### 2.2 创建 EventLoop

MultiThreadEventExecutorGroup 内部通过`newChild`方法创建 NioEventLoop：

```
    // NioEventLoopGroup.java
    
    protected EventLoop newChild(Executor executor, Object... args) throws Exception {
        EventLoopTaskQueueFactory queueFactory = args.length == 4 ? (EventLoopTaskQueueFactory) args[3] : null;
        return new NioEventLoop(this, executor, (SelectorProvider) args[0],
            ((SelectStrategyFactory) args[1]).newSelectStrategy(), (RejectedExecutionHandler) args[2], queueFactory);
    }
```

NioEventLoop 对象的构造，首先调用了父类`SingleThreadEventLoop`的构造方法，核心是初始化一个任务队列保存到内部：

```
    // SingleThreadEventLoop.java
    
    protected SingleThreadEventLoop(EventLoopGroup parent, Executor executor,
                                    boolean addTaskWakesUp, Queue<Runnable> taskQueue, 
                                    Queue<Runnable> tailTaskQueue,
                                    RejectedExecutionHandler rejectedExecutionHandler) {
        // addTaskWakesUp默认为false，用于标记添加任务是否会唤醒线程
        super(parent, executor, addTaskWakesUp, taskQueue, rejectedExecutionHandler);
        // 忽略这个tailTasks
        tailTasks = ObjectUtil.checkNotNull(tailTaskQueue, "tailTaskQueue");
    }
```

SingleThreadEventLoop 又调用了父类 SingleThreadEventExecutor 的构造方法，本质就是设置一些字段值：

```
    // SingleThreadEventLoop.java
    
    protected SingleThreadEventExecutor(EventExecutorGroup parent, Executor executor,
                                        boolean addTaskWakesUp, Queue<Runnable> taskQueue,
                                        RejectedExecutionHandler rejectedHandler) {
        super(parent);
        this.addTaskWakesUp = addTaskWakesUp;
        this.maxPendingTasks = DEFAULT_MAX_PENDING_EXECUTOR_TASKS;
        this.executor = ThreadExecutorMap.apply(executor, this);
        // 初始化NioEventLoop内部的任务队列
        this.taskQueue = ObjectUtil.checkNotNull(taskQueue, "taskQueue");
        this.rejectedExecutionHandler = ObjectUtil.checkNotNull(rejectedHandler, "rejectedHandler");
    }
```

### 2.3 注册 Channel

前一章我已经讲解了 ServerBootStrap 的启动流程，ServerBootStrap 注册 ServerSocketChannel 就是通过`NioEventLoopGroup.register()`方法完成的（定义在父类 MultithreadEventLoopGroup 中）：

```
    // MultithreadEventLoopGroup.java
    
    @Override
    public ChannelFuture register(Channel channel) {
        return next().register(channel);
    }
    
    public EventLoop next() {
        return (EventLoop) super.next();
    }
```

#### 选择 EventLoop

EventLoopGroup 管理着内部的 EventLoop 数组，所以上述的`next()`方法就是选择一个 EventLoop：

```
    // MultithreadEventExecutorGroup.java
    
    private final EventExecutorChooserFactory.EventExecutorChooser chooser;
    
    public EventExecutor next() {
        return chooser.next();
    }
```

具体的选择策略通过 EventExecutorChooser 完成，默认为 DefaultEventExecutorChooserFactory，采用了 Round-Robin 轮询策略：

```
    // DefaultEventExecutorChooserFactory.java
    
    private static final class PowerOfTwoEventExecutorChooser implements EventExecutorChooser {
        private final AtomicInteger idx = new AtomicInteger();
        private final EventExecutor[] executors;
    
        PowerOfTwoEventExecutorChooser(EventExecutor[] executors) {
            this.executors = executors;
        }
    
        @Override
        public EventExecutor next() {
            // Round-Robin轮询算法
            return executors[idx.getAndIncrement() & executors.length - 1];
        }
    }
```

#### 异步注册

接着，调用父类 SingleThreadEventLoop 的 register 方法注册 Channel。这里将 NioServerSocketChannel 和 SingleThreadEventLoop 封装成了一个 DefaultChannelPromise 对象，DefaultChannelPromise 可以看成是一种特殊的 ChannelFuture，可以手动设置异步 Future 的执行结果状态：

```
    // SingleThreadEventLoop.java
    
    public ChannelFuture register(Channel channel) {
        return register(new DefaultChannelPromise(channel, this));
    }
    
    public ChannelFuture register(final ChannelPromise promise) {
        ObjectUtil.checkNotNull(promise, "promise");
        promise.channel().unsafe().register(this, promise);
        return promise;
    }
```

Channel 内部有一个 UnSafe 类，封装了对底层 Java NIO SocketChannel 的许多方法的调用：

```
    // AbstractChannel.AbstractUnsafe.java
    
    public final void register(EventLoop eventLoop, final ChannelPromise promise) {
        ObjectUtil.checkNotNull(eventLoop, "eventLoop");
        // 如果已经注册过
        if (isRegistered()) {
            promise.setFailure(new IllegalStateException("registered to an event loop already"));
            return;
        }
        // ...
    
        AbstractChannel.this.eventLoop = eventLoop;
        // 如果是当前线程是EventLoop内部的工作线程
        if (eventLoop.inEventLoop()) {
            register0(promise);
        } 
        else {
            try {
                // 提交一个异步任务
                eventLoop.execute(new Runnable() {
                    @Override
                    public void run() {
                        register0(promise);
                    }
                });
            } catch (Throwable t) {
               //...
            }
        }
    }
```

从上面代码可以看到，Channel 的注册是 EventLoop 通过内部的 Executor 执行器提交一个异步任务完成的，之所以这样做是因为了使得 EventLoop 通过一个内部的工作线程就可以实现线程安全的任务执行。

本节我们先重点关注 Channel 的注册，后面我会分析 EventLoop 的异步任务执行逻辑：

```
    // AbstractChannel.java
    
    private void register0(ChannelPromise promise) {
        try {
            // 由于是异步执行，所以检查一下任务状态，避免Channel已经关闭
            if (!promise.setUncancellable() || !ensureOpen(promise)) {
                return;
            }
            boolean firstRegistration = neverRegistered;
            // 执行注册
            doRegister();
            neverRegistered = false;
            registered = true;
    
            // 触发handlerAdded事件，在Pipeline中传播
            pipeline.invokeHandlerAddedIfNeeded();
    
            safeSetSuccess(promise);
            // 触发channelRegistered事件，在Pipeline中传播
            pipeline.fireChannelRegistered();
    
            // 如果有新连接建立，则触发channelActive事件
            if (isActive()) {
                if (firstRegistration) {
                    pipeline.fireChannelActive();
                } else if (config().isAutoRead()) {
                    beginRead();
                }
            }
        } catch (Throwable t) {
            //...
        }
    }
```

对于 NioServerSocketChannel 来说，doRegister 方法在 AbstractNioChannel 类中实现，它的内部实际就是通过 Java NIO 的`java.nio.channels.SelectableChannel`完成在`java.nio.channels.Selector`上注册的：

```
    // AbstractNioChannel.java
    
    protected void doRegister() throws Exception {
        boolean selected = false;
        for (;;) {
            try {
                // 这里调用了
                selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);
                return;
            } catch (CancelledKeyException e) {
                //...
            }
        }
    }
```

#### Pipieline 事件传播

回到`AbstractChannel.register0()`方法中，Channel 注册完成后，会生成一系列事件，并在该 Channel 的 pipeline 中传播，主要有三类事件：

*   handlerAdded 事件：通过 pipeline.invokeHandlerAddedIfNeeded() 触发；
*   channelRegistered 事件：通过 pipeline.fireChannelRegistered() 触发；
*   channelActive 事件：通过 pipeline.fireChannelActive() 触发。

```
    // AbstractChannel.java
    
    private void register0(ChannelPromise promise) {
        try {
            //...
            // 执行注册
            doRegister();
            neverRegistered = false;
            registered = true;
    
            // 首次注册成功时，触发handlerAdded事件，在Pipeline中传播
            pipeline.invokeHandlerAddedIfNeeded();
    
            safeSetSuccess(promise);
            // 触发channelRegistered事件，在Pipeline中传播
            pipeline.fireChannelRegistered();
    
            // 如果有新连接建立，则触发channelActive事件
            if (isActive()) {
                if (firstRegistration) {
                    pipeline.fireChannelActive();
                } else if (config().isAutoRead()) {
                    beginRead();
                }
            }
        } catch (Throwable t) {
            //...
        }
    }
```

我们先来看`pipeline.invokeHandlerAddedIfNeeded()`，我之前分析过 ServerBootStrap 的启动流程，那时我们创建了一个 ChannelInitializer 对象，并添加到 Channel 的 pipeline 中。

事实上，Channel 注册完成后，会调用 Pipeline 的`invokeHandlerAddedIfNeeded`方法，而该方法 ChannelHandler 的 handlerAdded 方法，由于初始时我们只往 Pipeline 中添加了一个 ChannelInitializer，所以会触发 ChannelInitializer 对象的 handlerAdded 方法：

```
    // ChannelInitializer.java
    
    public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
        if (ctx.channel().isRegistered()) {
            // 初始化Channel
            if (initChannel(ctx)) {
                removeState(ctx);
            }
        }
    }
```

ChannelInitializer 的 handlerAdded 方法内部，调用了 initChannel 这个模板方法，它的内部又调用了我们自己覆写的 initChannel，其实就是将我们自定义的 ChannelHandler 添加到 Pipeline 中：

```
    // ChannelInitializer.java
    
    private boolean initChannel(ChannelHandlerContext ctx) throws Exception {
        if (initMap.add(ctx)) { 
            try {
                // 这个initChannel方法是一个抽象方法，一般我们自己的实现是将自定义的handlers添加到pipeline中
                initChannel((C) ctx.channel());
            } catch (Throwable cause) {
                exceptionCaught(ctx, cause);
            } finally {
                ChannelPipeline pipeline = ctx.pipeline();
                // 然后将ChannelInitializer自身移除 
                if (pipeline.context(this) != null) {
                    pipeline.remove(this);
                }
            }
            return true;
        }
        return false;
    }
```

通过上面的代码可以看出，ChannelInitializer 最终会将自己从 Pipeline 移除，避免重复执行。下图示意了这一过程，LoggingHandler 和 EchoClientHandler 是我们自定义的业务 Handler：

![](/img/network-program/netty/ChannelInitializer.png)

* * *

我们再来看`Pipeline.fireChannelRegistered()`方法，这个方法就是往 Pipeline 中扔一个 channelRegistered 事件，register 属于 Inbound（入站）事件，Pipeline 接下来要做的就是执行流水线中所有 Inbound 类型的 handlers 中的 channelRegistered() 方法：

```
    // DefaultChannelPipeline.java
    
    public final ChannelPipeline fireChannelRegistered() {
        // Register事件属于入站事件，所以从head头开始触发
        AbstractChannelHandlerContext.invokeChannelRegistered(head);
        return this;
    }
```

```
    // AbstractChannelHandlerContext.java
    
    static void invokeChannelRegistered(final AbstractChannelHandlerContext next) {
        EventExecutor executor = next.executor();
        // 执行Handler节点的invokeChannelRegistered方法
        // 如果当前线程不是EventLoop内的工作线程，依然以提交异步任务方式执行
        if (executor.inEventLoop()) {
            next.invokeChannelRegistered();
        } else {
            executor.execute(new Runnable() {
                @Override
                public void run() {
                    next.invokeChannelRegistered();
                }
            });
        }
    }
    
    private void invokeChannelRegistered() {
        if (invokeHandler()) {    // 确保ChannelHandler#handlerAdded方法已经执行过
            try {
                // 调用当前Handler的channelRegistered方法
                ((ChannelInboundHandler) handler()).channelRegistered(this);
            } catch (Throwable t) {
                invokeExceptionCaught(t);
            }
        } else {
            fireChannelRegistered();
        }
    }
```

```
    // DefaultChannelPipeline.java
    
    public void channelRegistered(ChannelHandlerContext ctx) {
        // 判断是否需要执行当前节点的handlerAdded方法
        invokeHandlerAddedIfNeeded();
        // 向后传播注册inbound事件
        ctx.fireChannelRegistered();
    }
```

关键看当前 Handler 节点的 fireChannelRegistered 方法，它内部的`findContextInbound()`方法会沿着 pipeline 找到下一个 Inbound 类型的 handler：

```
    // AbstractChannelHandlerContext.java
    
    public ChannelHandlerContext fireChannelRegistered() {
        invokeChannelRegistered(findContextInbound(MASK_CHANNEL_REGISTERED));
        return this;
    }
    
    private AbstractChannelHandlerContext findContextInbound(int mask) {
        AbstractChannelHandlerContext ctx = this;
        EventExecutor currentExecutor = executor();
        // 找到下一个Inbound Handler
        do {
            ctx = ctx.next;
        } while (skipContext(ctx, currentExecutor, mask, MASK_ONLY_INBOUND));
        return ctx;
    }
```

通过上面的源码，我们也就理解了`pipeline.fireChannelRegistered()`方法和`context.fireChannelRegistered()`方法的区别：

*   pipeline.fireChannelRegistered() 是将 channelRegistered 事件抛到 pipeline 的 head 中，pipeline 中的 handlers 依次处理该事件；
*   context.fireChannelRegistered() 是当前 handler 处理完该事件后，向后传播给下一个 handler，依次执行。

* * *

最后，我们来看 channelActive 事件，它是通过 pipeline.fireChannelActive() 触发的，原理和 channelRegistered 事件是一样的：

```
    // DefaultChannelPipeline.java
    
    public final ChannelPipeline fireChannelActive() {
        // 从pipeline的头部Handler开始向后传播事件
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
    
    private void invokeChannelActive() {
        if (invokeHandler()) {
            try {
                ((ChannelInboundHandler) handler()).channelActive(this);
            } catch (Throwable t) {
                invokeExceptionCaught(t);
            }
        } else {
            fireChannelActive();
        }
    }
```

三、EventLoop
-----------

EventLoopGroup 可以看成是一个管理 EventLoop 的容器，真正执行任务的是 EventLoop，它是 Netty Reactor 线程模型的核心处理引擎。Netty 中的所有 I/O 操作都是异步执行的，比如上一节中讲解的 Channel 注册，其实只是向 EventLoop 中提交了一个任务，由 EventLoop 负责异步执行，那么它是如何高效地实现事件循环和任务处理机制的呢？

本节，我就来分析 NioEventLoop 的底层原理，NioEventLoop 的类继承体系如下图：

![](/img/network-program/netty/NioEventLoop-Class.png)

NioEventLoop 需要负责 IO 事件和非 IO 事件，NioEventLoop 内部有一个线程一直在执行 Selector 的 select 方法或者正在处理 SelectedKeys，如果有外部线程提交一个任务给 NioEventLoop ，任务就会被放到 它内部的 taskQueue 中，该队列是线程安全的，默认容量为 16：

![](/img/network-program/netty/NioEventLoop.png)

我们之前分析 EventLoopGroup 源码时已经看过 EventLoop 的构造了，那里只是创建了 EventLoop 对象，并没有启动它的内部线程。事实上，EventLoop 创建内部工作线程的时机是在第一个任务提交时，一般就是 SocketChannel 的 register 操作。

### 3.1 工作线程创建

我们从 Channel 的注册入手，看下 EventLoop 是如何进行任务调度的：

```
    // AbstractChannel.java
    
    public final void register(EventLoop eventLoop, final ChannelPromise promise) {
        //...
        AbstractChannel.this.eventLoop = eventLoop;
    
        if (eventLoop.inEventLoop()) {
            register0(promise);
        } else {
            try {
                // 重点看这里
                eventLoop.execute(new Runnable() {
                    @Override
                    public void run() {
                        register0(promise);
                    }
                });
            } catch (Throwable t) {
                logger.warn(
                    "Force-closing a channel whose registration task was not accepted by an event loop: {}",
                    AbstractChannel.this, t);
                closeForcibly();
                closeFuture.setClosed();
                safeSetFailure(promise, t);
            }
        }
    }
```

在 EventLoop 中，有一个重要的`execute(Runnable command)`方法（事实上继承自父类 SingleThreadEventExecutor）

```
    // SingleThreadEventExecutor.java
    
    public void execute(Runnable task) {
        ObjectUtil.checkNotNull(task, "task");
        execute(task, !(task instanceof LazyRunnable) && wakesUpForTask(task));
    }
    
    private void execute(Runnable task, boolean immediate) {
        // 1.判断当前执行线程是否为EventLoop内部的工作线程
        boolean inEventLoop = inEventLoop();
        // 2.添加任务到内部队列
        addTask(task);
        // 3.如果不是EventLoop内部线程提交的task，则判断内部线程是否已经启动，没有则启动内部线程
        if (!inEventLoop) {
            // 启动内部线程
            startThread();
            if (isShutdown()) {
                boolean reject = false;
                try {
                    if (removeTask(task)) {
                        reject = true;
                    }
                } catch (UnsupportedOperationException e) {
                }
                if (reject) {
                    // 执行拒绝策略
                    reject();
                }
            }
        }
    
        if (!addTaskWakesUp && immediate) {
            wakeup(inEventLoop);
        }
    }
```

继续看`startThread()`方法：

```
    // SingleThreadEventExecutor.java
    
    private void startThread() {
        if (state == ST_NOT_STARTED) {
            // 这里用了CAS操作，以非阻塞的线程安全方式更新
            if (STATE_UPDATER.compareAndSet(this, ST_NOT_STARTED, ST_STARTED)) {
                boolean success = false;
                try {
                    // 执行启动内部线程
                    doStartThread();
                    success = true;
                } finally {
                    if (!success) {
                        STATE_UPDATER.compareAndSet(this, ST_STARTED, ST_NOT_STARTED);
                    }
                }
            }
        }
    }
```

doStartThread 方法看起来很长，其实就是向 NioEventLoop 内部的 ThreadPerTaskExecutor 提交一个任务，ThreadPerTaskExecutor 会创建一个新线程来执行这个任务，这个线程就是 NioEventLoop 内部的工作线程：

```
    // SingleThreadEventExecutor.java
    
    private volatile Thread thread;
    
    private void doStartThread() {
        assert thread == null;
        // 对于NioEventLoop，这个executor就是ThreadPerTaskExecutor
        executor.execute(new Runnable() {
            @Override
            public void run() {
                // 将这个线程设置为NioEventLoop的内部工作线程
                thread = Thread.currentThread();
                if (interrupted) {
                    thread.interrupt();
                }
    
                boolean success = false;
                updateLastExecutionTime();
                try {
                    // 执行 SingleThreadEventExecutor 的 run() 方法，它在 NioEventLoop 中实现了
                    SingleThreadEventExecutor.this.run();
                    success = true;
                } catch (Throwable t) {
                    logger.warn("Unexpected exception from an event executor: ", t);
                } finally {
                   //...
                }
            }
        });
    }
```

工作线程会执行到上述代码的`SingleThreadEventExecutor.this.run()`这一行，这是一个抽象方法，需要子类也就是 NioEventLoop 自己实现，NioEventLoop 会在此处实现自己的核心任务调用逻辑。

### 3.2 任务调度流程

我们继续看 NioEventLoop 的`run`方法，这是 NioEventLoop 的核心逻辑，NioEventLoop 每次循环的处理流程都包含 **IO 事件轮询 select** 、 **事件处理 processSelectedKeys** 、 **任务处理 runAllTasks** 几个步骤，其中有几个关键点我重点说一下：

1.  selectStrategy 用于控制工作线程的`select`策略，在存在异步任务的场景，NioEventLoop 会优先保证 CPU 能够及时处理异步任务；
2.  ioRatio 参数用于控制 I/O 事件处理和内部任务处理的时间比例，默认值为 50。如果 ioRatio = 100，表示每次处理完 I/O 事件后，会执行所有的 task。如果 ioRatio <100，也会优先处理完 I/O 事件，再处理异步任务队列。所以无论如何，processSelectedKeys() 都是先执行的。

```
    // NioEventLoop.java
    
    protected void run() {
        int selectCnt = 0;
        // 一直循环执行
        for (;;) {
            try {
                int strategy;
                try {
                    // select处理策略，用于控制select循环行为，包含CONTINUE、SELECT、BUSY_WAIT三种策略
                    // Netty不支持BUSY_WAIT，所以 BUSY_WAIT 与 SELECT 的执行逻辑是一样的
                    strategy = selectStrategy.calculateStrategy(selectNowSupplier, hasTasks());
                    switch (strategy) {
                        case SelectStrategy.CONTINUE:
                            continue;
                        case SelectStrategy.BUSY_WAIT:
                        case SelectStrategy.SELECT:
                            long curDeadlineNanos = nextScheduledTaskDeadlineNanos();
                            if (curDeadlineNanos == -1L) {
                                curDeadlineNanos = NONE; 
                            }
                            nextWakeupNanos.set(curDeadlineNanos);
                            try {
                                if (!hasTasks()) {
                                    // 轮询I/O事件
                                    strategy = select(curDeadlineNanos);
                                }
                            } finally {
                                nextWakeupNanos.lazySet(AWAKE);
                            }
                        default:
                    }
                } catch (IOException e) {
                    //...
                    continue;
                }
    
                selectCnt++;
                cancelledKeys = 0;
                needsToSelectAgain = false;
                final int ioRatio = this.ioRatio;
                boolean ranTasks;
                // 根据ioRatio，选择执行IO操作还是内部队列中的任务
                if (ioRatio == 100) {
                    try {
                        if (strategy > 0) {
                            processSelectedKeys();
                        }
                    } finally {
                        ranTasks = runAllTasks();
                    }
                } else if (strategy > 0) {
                    final long ioStartTime = System.nanoTime();
                    try {
                        processSelectedKeys();
                    } finally {
                        final long ioTime = System.nanoTime() - ioStartTime;
                        ranTasks = runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
                    }
                } else {
                    ranTasks = runAllTasks(0); // This will run the minimum number of tasks
                }
    
                if (ranTasks || strategy > 0) {
                    if (selectCnt > MIN_PREMATURE_SELECTOR_RETURNS && logger.isDebugEnabled()) {
                        logger.debug("Selector.select() returned prematurely {} times in a row for Selector {}.", selectCnt - 1, selector);
                    }
                    selectCnt = 0;
                } else if (unexpectedSelectorWakeup(selectCnt)) { // 解决JDK的epoll空轮询问题
                    selectCnt = 0;
                }
            }
            //...
        }
    }
```

总之，NioEventLoop 的`run`方法是一个无限循环，不间断执行以下三件事情：

*   select：轮询 Selector 选择器中已经注册的所有 Channel 的 I/O 事件；
*   processSelectedKeys：根据 SelectedKeys，处理已经准备就绪的 I/O 事件；
*   runAllTasks：执行内部队列中的任务。

![](/img/network-program/netty/NioEventLoop-run.png)

同时，为了提升效率，NioEventLoop 会就是根据一定的策略和`ioRatio`参数配置，选择究竟是执行内部队列中的任务，还是执行 IO 操作。

#### epoll 空轮询

注意，NioEventLoop 的 run 方法中，有这么一段代码：

```
    // NioEventLoop.java
    
    protected void run() {
        int selectCnt = 0;
        // 一直循环执行
        for (;;) {
            //...
            if (ranTasks || strategy > 0) {
                if (selectCnt > MIN_PREMATURE_SELECTOR_RETURNS && logger.isDebugEnabled()) {
                    logger.debug("..."");
                }
                selectCnt = 0;
            } else if (unexpectedSelectorWakeup(selectCnt)) { // 解决JDK的epoll空轮询问题
                selectCnt = 0;
            }
            //...
        }
    }
    
    private boolean unexpectedSelectorWakeup(int selectCnt) {
        if (Thread.interrupted()) {
            return true;
        }
        if (SELECTOR_AUTO_REBUILD_THRESHOLD > 0 &&
            selectCnt >= SELECTOR_AUTO_REBUILD_THRESHOLD) {
            rebuildSelector();
            return true;
        }
        return false;
    }
```

上述的`unexpectedSelectorWakeup`方法用于解决 JDK NIO 中 Epoll 实现的空轮询问题。所谓 _**JDK Epoll 空轮询**_ ，是指 NIO 线程在没有感知到 select 事件时，应该处于阻塞状态，但是 JDK 的 epoll 实现会出现即使 Selector 轮询的事件列表为空，NIO 线程一样可以被唤醒，导致 CPU 100% 占用。

Netty 作为一个高性能、高可靠的网络框架，需要保证 I/O 线程的安全性，Netty 很巧妙的规避了这个问题， 它提供了一种检测机制判断 I/O 线程是否可能陷入空轮询，具体的实现方式如下：

1.  首先，EventLoop 在每次执行 select 操作前，都会记录当前时间 currentTimeNanos。通过时间比对，如果 Netty 发现 NIO 线程的阻塞时间并未达到预期，则认为可能触发了空轮询的 Bug；
2.  同时，Netty 引入了计数变量 selectCnt。在正常情况下，selectCnt 会重置，否则会对 selectCnt 自增计数。当 selectCnt 达到 `SELECTOR_AUTO_REBUILD_THRESHOLD`（默认 512） 阈值时，会触发重建 Selector 对象；
3.  然后，Netty 会将异常的 Selector 中所有的 SelectionKey 会重新注册到新建的 Selector 上，重建完成之后异常的 Selector 就可以废弃了。

总结一下，Netty 解决 _**JDK Epoll 空轮询**_ 问题的思路就是：引入计数器变量，统计一定时间窗口内 select 操作的执行次数，识别出可能存在异常的 Selector 对象，然后采用重建 Selector 的方式巧妙地避免了 JDK epoll 空轮询的问题。

### 3.3 轮询 I/O 事件

先来看轮询 IO 事件，比较简单，就是调用底层 Java NIO Selector 的 select 方法

```
    // NioEventLoop.java
    
    private int select(long deadlineNanos) throws IOException {
        if (deadlineNanos == NONE) {
            // 调用了底层NIO Selector的select方法
            return selector.select();
        }
        long timeoutMillis = deadlineToDelayNanos(deadlineNanos + 995000L) / 1000000L;
        return timeoutMillis <= 0 ? selector.selectNow() : selector.select(timeoutMillis);
    }
```

### 3.4 处理 I/O 事件

通过 select 过程，NioEventLoop 已经获取到准备就绪的 I/O 事件，接下来需要调用 `processSelectedKeys()` 方法处理 IO 事件：

```
    // NioEventLoop.java
    
    private SelectedSelectionKeySet selectedKeys;    // 保存java.nio.channels.SelectionKey的集合
    
    private void processSelectedKeys() {
        if (selectedKeys != null) {
            // Netty优化过的selectedKeys
            processSelectedKeysOptimized();
        } else {
            // 正常处理逻辑
            processSelectedKeysPlain(selector.selectedKeys());
        }
    }
```

处理 I/O 事件时有两种选择，Netty 是否采用优化策略由 `DISABLE_KEYSET_OPTIMIZATION`参数决定：

1.  处理 Netty 优化过的 selectedKeys；
2.  正常的处理逻辑。

根据是否设置了 `selectedKeys` 来判断采用哪种策略，这两种策略的差异就是使用的 selectedKeys 集合不同。第一种 Netty 优化过的 selectedKeys 是 `SelectedSelectionKeySet` 类型，而正常逻辑使用的是 JDK HashSet 类型。

#### processSelectedKeysPlain

我们先来看正常的处理逻辑：

```
    // NioEventLoop.java
    
    private void processSelectedKeysPlain(Set<SelectionKey> selectedKeys) {
        if (selectedKeys.isEmpty()) {
            return;
        }
    
        Iterator<SelectionKey> i = selectedKeys.iterator();
        for (;;) {
            final SelectionKey k = i.next();
            final Object a = k.attachment();
            i.remove();
    
            if (a instanceof AbstractNioChannel) {
                // I/O 事件由 Netty 负责处理
                processSelectedKey(k, (AbstractNioChannel) a);
            } else {
                // 用户自定义任务
                @SuppressWarnings("unchecked")
                NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
                processSelectedKey(k, task);
            }
    
            if (!i.hasNext()) {
                break;
            }
    
            // Netty在处理I/O事件时，如果发现超过256个（默认）Channel从Selector对象中移除，就会将needsToSelectAgain置true，重新做一次轮询，从而确保 keySet 的有效性
            if (needsToSelectAgain) {
                selectAgain();
                selectedKeys = selector.selectedKeys();
                if (selectedKeys.isEmpty()) {
                    break;
                } else {
                    i = selectedKeys.iterator();
                }
            }
        }
    }
```

NioEventLoop 会遍历已经就绪的 SelectionKey，SelectionKey 的类型可能是`AbstractNioChannel`或`NioTask`，这两种类型对应的处理方式是不同的，我们关注 AbstractNioChannel 类型即可：

```
    // NioEventLoop.java
    
    private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
        final AbstractNioChannel.NioUnsafe unsafe = ch.unsafe();
        if (!k.isValid()) {    // 检查 Key 是否合法
            final EventLoop eventLoop;
            try {
                eventLoop = ch.eventLoop();
            } catch (Throwable ignored) {
                return;
            }
            if (eventLoop == this) {    
                unsafe.close(unsafe.voidPromise());    // Key 不合法，直接关闭连接
            }
            return;
        }
    
        try {
            int readyOps = k.readyOps();
            // 处理连接事件
            if ((readyOps & SelectionKey.OP_CONNECT) != 0) {
                // 将该事件从事件集合中清除，避免事件集合中一直存在连接建立事件
                int ops = k.interestOps();
                ops &= ~SelectionKey.OP_CONNECT;
                k.interestOps(ops);
                // 通知上层连接已经建立
                unsafe.finishConnect();
            }
            // 处理可写事件
            if ((readyOps & SelectionKey.OP_WRITE) != 0) {
                ch.unsafe().forceFlush();
            }
            // 处理可读事件
            if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
                unsafe.read();
            }
        } catch (CancelledKeyException ignored) {
            unsafe.close(unsafe.voidPromise());
        }
    }
```

*   **OP_CONNECT 连接事件** ：EventLoop 内部调用了`unsafe.finishConnect()`，底层调用`pipeline().fireChannelActive()` 方法，这时会产生一个 Inbound 事件，在 Pipeline 中传播，依次调用 ChannelHandler 的 channelActive() 方法，通知各个 ChannelHandler 连接建立成功；
    
*   **OP_WRITE 可写事件** ：内部会执行`ch.unsafe().forceFlush()`操作，将数据刷到对端，最终会调用 Java NIO 中`Channel .write()`方法执行底层写操作；
    
*   **OP_READ 可读事件** ：Netty 将 OP_READ 和 OP_ACCEPT 事件进行了统一封装，都通过`unsafe.read()`进行处理，unsafe.read() 的处理逻辑如下：
    
    *   从 Channel 中读取数据并存储到分配的 ByteBuf；
    *   调用`pipeline.fireChannelRead()`方法产生 Inbound 事件，在 Pipeline 中传播，依次调用 ChannelHandler 的 channelRead() 方法处理数据；
    *   调用`pipeline.fireChannelReadComplete()`方法完成读操作，同样在 Pipeline 传播；
    *   执行 `removeReadOp()`清除 OP_READ 事件。

#### processSelectedKeysOptimized

介绍完正常的 I/O 事件处理 processSelectedKeysPlain 后，我们再分析 Netty 优化的 processSelectedKeysOptimized 源码就轻松很多：

```
    // NioEventLoop.java
    
    private void processSelectedKeysOptimized() {
        for (int i = 0; i < selectedKeys.size; ++i) {
            final SelectionKey k = selectedKeys.keys[i];
            selectedKeys.keys[i] = null;
    
            final Object a = k.attachment();
            if (a instanceof AbstractNioChannel) {
                processSelectedKey(k, (AbstractNioChannel) a);
            } else {
                @SuppressWarnings("unchecked")
                NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
                processSelectedKey(k, task);
            }
    
            if (needsToSelectAgain) {
                // null out entries in the array to allow to have it GC'ed once the Channel close
                // See https://github.com/netty/netty/issues/2363
                selectedKeys.reset(i + 1);
    
                selectAgain();
                i = -1;
            }
        }
    }
```

可以发现 processSelectedKeysOptimized 与 processSelectedKeysPlain 的主要区别就是 selectedKeys 的遍历方式不同，来看下 SelectedSelectionKeySet 的源码：

```
    // SelectedSelectionKeySet.java
    
    final class SelectedSelectionKeySet extends AbstractSet<SelectionKey> {
        SelectionKey[] keys;
        int size;
    
        SelectedSelectionKeySet() {
            keys = new SelectionKey[1024];
        }
    
        public boolean add(SelectionKey o) {
            if (o == null) {
                return false;
            }
    
            keys[size++] = o;
            if (size == keys.length) {
                increaseCapacity();
            }
    
            return true;
        }
        //...
    }
```

可以看到，SelectedSelectionKeySet 内部维护了一个 SelectionKey 数组，所以`processSelectedKeysOptimized`可以直接通过遍历数组取出 I/O 事件，相比`processSelectedKeysPlain`采用的 JDK HashSet 的遍历效率更高。

SelectedSelectionKeySet 内部通过 size 变量记录数组的逻辑长度（每次执行 add 操作，会将元素添加到 SelectionKey[] 尾部，当 size 等于 SelectionKey[] 的真实长度时，自动扩容），相比于 HashSet，由于不需要考虑哈希冲突的问题，所以可以实现`O(1)`时间复杂度的 add 操作。

> Netty 在创建 Selector 时，会通过反射的方式，将 Selector 对象内部的 selectedKeys 和 publicSelectedKeys 替换为 SelectedSelectionKeySet，而原先 selectedKeys 和 publicSelectedKeys 这两个字段都是 HashSet 类型。

### 3.5 内部任务处理

NioEventLoop 不仅负责处理 I/O 事件，还要兼顾执行任务队列中的任务。任务队列遵循 FIFO 规则，可以保证任务执行的公平性，Netty 没有使用 J.U.C 中的并发队列，而是自己实现了 **多生产者单消费者队列 MpscChunkedArrayQueue** ，这个我会在后续章节分析。

NioEventLoop 处理的任务类型基本可以分为三类：

*   **普通任务** ：通过`NioEventLoop.execute()`方法向任务队列 taskQueue 中添加任务，例如 NioServerSocketChannel 的 注册；
*   **定时任务** ：通过 `NioEventLoop.schedule()`方法向定时任务队列 scheduledTaskQueue 添加一个定时任务，用于周期性执行该任务，例如，心跳消息发送等。定时任务队列 scheduledTaskQueue 采用优先队列 PriorityQueue 实现；
*   **尾部队列** ：tailTasks 相比于普通任务队列优先级较低，在每次执行完 taskQueue 中任务后会去获取尾部队列中任务执行。尾部任务并不常用，主要用于做一些收尾工作，例如统计事件循环的执行时间、监控信息上报等。

EventLoop 处理内部任务有两个方法：

*   runAllTasks；
*   runAllTasks(long timeoutNanos)。

第二种方式携带了超时时间，防止 NIO 工作线程因为处理任务时间过长而导致 I/O 事件阻塞，我们来看下 NioEventLoop 中的`runAllTasks(long timeoutNanos)`方法：

```
    // NioEventLoop.java
    
    protected boolean runAllTasks(long timeoutNanos) {
        // 合并定时任务到普通任务队列
        fetchFromScheduledTaskQueue();
        // 从普通任务队列中取出任务并处理
        Runnable task = pollTask();
        if (task == null) {
            afterRunningAllTasks();
            return false;
        }
        // 计算任务处理的超时时间
        final long deadline = timeoutNanos > 0 ? ScheduledFutureTask.nanoTime() + timeoutNanos : 0;
        long runTasks = 0;
        long lastExecutionTime;
        for (;;) {
            // 安全执行任务，执行调用Runnable任务的run()方法同步执行
            safeExecute(task);
            runTasks ++;
            // 每执行 64 个任务检查一下是否超时
            if ((runTasks & 0x3F) == 0) {
                lastExecutionTime = ScheduledFutureTask.nanoTime();
                if (lastExecutionTime >= deadline) {
                    break;
                }
            }
            // 取出下一个任务，继续执行
            task = pollTask();
            if (task == null) {
                lastExecutionTime = ScheduledFutureTask.nanoTime();
                break;
            }
        }
        // 收尾工作，执行尾部队列tailTasks的任务
        afterRunningAllTasks();
        this.lastExecutionTime = lastExecutionTime;
        return true;
    }
```

EventLoop 内部的定时任务队列`scheduledTaskQueue`本质是一个优先级队列，里面的定时任务只有满足截止时间 `deadlineNanos`小于当前时间，才可以取出合并到普通任务队列中。由于优先级队列是有序的，所以只要取出的定时任务不满足合并条件，那么队列中剩下的任务都不会满足条件，合并操作完成并退出。

最后，NioEventLoop 执行内部队列中的任务的流程是比较简单的，就是不断取出 Runnable 任务，最终直接调用 run 方法执行：

```
    // AbstractEventExecutor.java
    
    protected static void safeExecute(Runnable task) {
        try {
            // 安全执行任务，执行调用Runnable任务的run()方法同步执行
            task.run();
        } catch (Throwable t) {
            logger.warn("A task raised an exception. Task: {}", task, t);
        }
    }
```

另外，整个执行过程中还有一个收尾动作`afterRunningAllTasks`，主要是执行尾部队列 tailTasks 的任务。尾部队列并不常用，主要用于一些统计场景，比如，我们想对 Netty 的运行状态进行统计分析，如任务循环耗时、占用内存大小等等，则可以向尾部队列添加一个任务执行统计。

四、总结
----

本章，我对 Netty 中核心的 EventLoopGroup 和 EventLoop 的底层原理进行了分析，核心是 EventLoop 的设计和实现。EventLoop 的设计思想被广泛运用于各类的开源框架中，如 Kafka、Redis、Nginx 等等，学习它的源码可以让我们对 Java NIO 有更深层次的理解。

通过 EventLoop 的底层源码解读，我们应该意识到在使用 Netty 进行日常开发时，需要遵循一些原则：

1.  尽量使用主从 Reactor 模型，即采用 Boss 和 Worker 两个 EventLoopGroup，分担 Reactor 线程的压力；
2.  Reactor 线程模式只适合处理耗时短的任务场景，对于耗时较长的业务 ChannelHandler，最好自己维护一个业务线程池，异步处理处理任务，避免因为 ChannelHandler 阻塞而造成 EventLoop 不可用。
3.  在设计业务架构时，需要明确业务分层和 Netty 分层之间的界限，不要一味地将业务逻辑都添加到 ChannelHandler 中，完全可以基于 MQ 等中间件进行解耦。