---
layout:     post
title:      透彻理解 java 网络编程(十二)
subtitle:   ChannelPipeline 和 ChannelHandler
date:       2024-07-13
author:     yusihu
header-style: text
catalog: true
tags:
    - 网络编程
    - netty
    - 开源框架
---
EventLoop 虽然是 Netty 的调度中心，负责监听各类事件：I/O 事件、信号事件、定时事件等，但与我们实际开发最息息相关的却是 ChannelPipeline 和 ChannelHandler。

这一节，我就对 Netty 中的这两个组件进行深入分析。

一、ChannelPipeline
-----------------

Netty 的流水线 ChannelPipeline 用以实现网络事件的动态编排和有序传播，基于 **责任链设计模式（Chain of Responsibility）** 设计，内部是一个 **双向链表** 结构，支持动态地添加和删除 ChannelHandler 业务处理器。

### 1.1 内部结构

Channel、ChannelPipeline、ChannelHandlerContext、ChannelHandler 四者的关系可以用下面这张图表示：

![](/img/network-program/netty/ChannelPipeline-structrue.png)

- 每个 Channel 会绑定一个 ChannelPipeline，Pipeline 初始化时有 Head 和 Tail 两个节点；  
- 每个 ChannelPipeline 包含多个 ChannelHandlerContext，所有 ChannelHandlerContext 之间组成了双向链表；  
- 每个 ChannelHandler 都对应一个 ChannelHandlerContext。

_**这里为什么需要多一层 ChannelHandlerContext 对 ChannelHandler 进行封装呢？**_

因为如果没有 ChannelHandlerContext 的这层封装，那我们在 ChannelHandler 之间传递事件时，前置后置的通用逻辑就要在每个 ChannelHandler 里都实现一份。而定义 ChannelHandlerContext，可以将 ChannelHandler 生命周期的所有事件，如 connect、bind、read、flush、write、close 等都抽取出来，减少代码耦合。

我们再从源码层面看下 ChannelPipeline 双向链表的构造。

ChannelPipeline 的双向链表分别维护了 `HeadContext` 头节点和 `TailContext` 尾节点。我们自定义的 ChannelHandler 会插入到 Head 和 Tail 之间，HeadContext 和 TailContext 的继承关系如下图：

![](/img/network-program/netty/HeadContext-TailContext.png)

从上述类图，我们可以看出以下关键几点：

1.  HeadContext 既是 Inbound 处理器，也是 Outbound 处理器，它分别实现了 ChannelInboundHandler 和 ChannelOutboundHandler；
2.  HeadContext 作为 Pipeline 的头结点，负责读取数据并开始传递 InBound 事件，当数据处理完成后，数据会反方向经过 Outbound 处理器，最终传递到 HeadContext。所以，HeadContext 又是处理 Outbound 事件的最后一站；
3.  TailContext 只是 Inbound 处理器，它只实现了 ChannelInboundHandler 接口；
4.  TailContext 作为 Pipeline 的尾结点，会在 ChannelInboundHandler 调用链路的最后一步执行，用于终止 Inbound 事件传播。同时，TailContext 节点作为 OutBound 事件传播的第一站，会将 OutBound 事件传递给上一个节点。

正式因为这样的设计，如果由 Channel 直接触发事件传播，那么调用链路将贯穿整个 ChannelPipeline。如果由某个 ChannelHandlerContext 触发，则只会从当前的 ChannelHandler 开始执行事件传播，该过程不会从头贯穿到尾，在一定场景下，可以提高程序性能。

### 1.2 出 / 入站处理

根据网络数据的流向，ChannelPipeline 包含 **入站 ChannelInboundHandler** 和 **出站 ChannelOutboundHandler** 两种处理器。

客户端与服务端通信时，数据从客户端发向服务端的过程叫出站，反之称为入站。数据先由一系列 InboundHandler 处理后入站，然后再由相反方向的 OutboundHandler 处理完成后出站，如下图所示：

![](/img/network-program/netty/ChannelPipeline-in-out.png)

比如，我们经常使用的解码器 Decoder 就是入站操作，编码器 Encoder 就是出站操作。

这里补充一下，Netty 默认提供了一个 EmbeddedChannel，模拟入站与出站的操作，底层不进行实际的传输，不需要启动 Netty 服务器和客户端，常常用于测试。EmbeddedChannel 提供了一组方法，可以分别模拟客户端和服务端的出入站操作：

<table><thead><tr><th>方法名称</th><th>说明</th></tr></thead><tbody><tr><td>writeInbound</td><td>向 Channel 写入 Inbound 入站数据，这些数据会在 Pipeline 上经过入站处理器依次处理</td></tr><tr><td>readInbound</td><td>从 Chanel 读取 Inbound 入站数据，返回 Pipeline 上最后一个入站处理器处理后的数据</td></tr><tr><td>readOutbound</td><td>从 Channel 读取 Outbound 出站数据，返回 Pipeline 上最后一个出站处理器处理后的数据</td></tr><tr><td>writeOutbound</td><td>向 Channel 写入 Outbound 出站数据，这些数据会在 Pipeline 上经过出站处理器依次处理</td></tr></tbody></table>

我们通过一张图来理解下：

![](/img/network-program/netty/ChannelPipeline-in-out-handle.png)

下面，我通过示例来讲解下 ChannelPipeline 对出入站事件的处理。

#### 入站处理流程

我们先通过一个示例来理解下 ChannelPipeline 的入站处理流程：

```
    public class InboundDemo {
        static class SimpleInHandlerA extends ChannelInboundHandlerAdapter {
            @Override
            public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
                System.out.println("入站处理器A: 被回调: " + msg);
                super.channelRead(ctx, msg);
            }
        }
    
        static class SimpleInHandlerB extends ChannelInboundHandlerAdapter {
            @Override
            public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
                System.out.println("入站处理器B: 被回调: " + msg);
                super.channelRead(ctx, msg);
            }
        }
    
        static class SimpleInHandlerC extends ChannelInboundHandlerAdapter {
            @Override
            public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
                System.out.println("入站处理器C: 被回调: " + msg);
                super.channelRead(ctx, msg);
            }
        }
    
        public static void main(String[] args) {
            testInboud();
        }
    
        public static void testInboud() {
            ChannelInitializer channelInitializer = new ChannelInitializer<EmbeddedChannel>() {
                @Override
                protected void initChannel(EmbeddedChannel ch) throws Exception {
                    ch.pipeline().addLast(new SimpleInHandlerA());
                    ch.pipeline().addLast(new SimpleInHandlerB());
                    ch.pipeline().addLast(new SimpleInHandlerC());
                }
            };
            EmbeddedChannel channel = new EmbeddedChannel(channelInitializer);
            ByteBuf buf = Unpooled.buffer();
            buf.writeInt(90);
            channel.writeInbound(buf);
        }
    }
```

上面代码中，我创建了三个入站处理器，然后向 Channel 写入数据，触发一个 read 入站事件，执行结果如下：

```
    入站处理器A: 被回调 
    入站处理器B: 被回调 
    入站处理器C: 被回调
```

注意，我在每个 Handler 内部都调用了父类`ChannelInboundHandlerAdapter`的 channelRead 方法，就是为了在 Pipeline 中传递事件，当然我们也可以直接调用`ChannelHandlerContext.fireChannelRead`方法：

```
    // ChannelInboundHandlerAdapter.java
    
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        ctx.fireChannelRead(msg);
    }
```

![](/img/network-program/netty/InBound-handle-fire.png)

> 如果不调用`super.channelRead(ctx, msg);`，则事件不会向后传播。

#### 出站处理流程

我们再来通过一个示例来理解下 ChannelPipeline 的出站处理流程：

```
    public class OutboundDemo {
        static class SimpleOutHandlerA extends ChannelOutboundHandlerAdapter {
            @Override
            public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
                System.out.println("出站处理器A: 被回调 ");
                super.write(ctx, msg, promise);
            }
        }
    
        static class SimpleOutHandlerB extends ChannelOutboundHandlerAdapter {
            @Override
            public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
                System.out.println("出站处理器B: 被回调 ");
                super.write(ctx, msg, promise);
            }
        }
    
        static class SimpleOutHandlerC extends ChannelOutboundHandlerAdapter {
            @Override
            public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
                System.out.println("出站处理器C: 被回调 ");
                super.write(ctx, msg, promise);
            }
        }
    
        public static void main(String[] args) {
            testOutboud();
        }
    
        public static void testOutboud() {
            ChannelInitializer channelInitializer = new ChannelInitializer<EmbeddedChannel>() {
                @Override
                protected void initChannel(EmbeddedChannel ch) throws Exception {
                    ch.pipeline().addLast(new SimpleOutHandlerA());
                    ch.pipeline().addLast(new SimpleOutHandlerB());
                    ch.pipeline().addLast(new SimpleOutHandlerC());
                }
            };
            EmbeddedChannel channel = new EmbeddedChannel(channelInitializer);
            ByteBuf buf = Unpooled.buffer();
            buf.writeInt(90);
            channel.writeOutbound(buf);
        }
    }
```

上面代码中，我创建了三个出站处理器，然后向 Channel 写入数据，触发一个 write 出站事件，执行结果如下：

```
    出站处理器C: 被回调 
    出站处理器B: 被回调 
    出站处理器A: 被回调
```

注意，我在每个 Handler 内部都调用了父类`ChannelOutboundHandlerAdapter`的 write 方法，就是为了在 Pipeline 中传递事件，当然我们也可以直接调用`ChannelHandlerContext.writeAndFlush`方法。

这里还要特别注意下， **出站处理次序为从后向前，最后加入的出站处理器，反而执行在最前面** ，如下图：

![](/img/network-program/netty/OutBound-handle-fire.png)

### 1.3 ChannelHandlerContext

在 Handler 业务处理器被添加到 Pipeline 中时，会创建一个 ChannelHandlerContext 对象，它代表了 ChannelHandler 业务处理器和 ChannelPipeline 流水线之间的关联。

我们在处理出 / 入站事件时，如果通过 Channel 或 ChannelPipeline 来直接调用事件方法，则事件会在整条流水线中传播。然而，如果是通过 ChannelHandlerContext 进行调用，就只会从当前的节点开始执行 Handler 业务处理器，并传播到同类型处理器的下一站（节点）。

![](/img/network-program/netty/Channel_ChannelHandlerContext.png)

### 1.4 writeAndFlush 原理

服务端在接收到客户端的请求后，需要将响应结果编码后写回客户端。这个过程一般是通过`ChannelPipeline.writeAndFlush`方法完成的。事实上，writeAndFlush 主要分为两个步骤，write 和 flush：

*   **write：** 将待发送数据从 Pipeline 的 Tail 节点（或者当前 Context 节点）向前传播，直到 Head 节点，并写入其内部的 ChannelOutboundBuffer（一个单向链表缓存区）；
*   **flush：** 将 ChannelOutboundBuffer 中的数据写到 TCP 发送缓冲区，如果 ChannelOutboundBuffer 的数据全部 flush 完，则取消对`OP_WRITE`事件的关注。

我们先来看当调用`ChannelPipeline.writeAndFlush`方法的 **整体流程** ：

```
    // DefaultChannelPipeline.java
    
    final AbstractChannelHandlerContext tail;
    
    public final ChannelFuture writeAndFlush(Object msg) {
        return tail.writeAndFlush(msg);
    }
```

```
    // AbstractChannelHandlerContext.java
    
    public ChannelFuture writeAndFlush(Object msg) {
        return writeAndFlush(msg, newPromise());
    }
    
    public ChannelFuture writeAndFlush(Object msg, ChannelPromise promise) {
        write(msg, true, promise);
        return promise;
    }
    
    private void write(Object msg, boolean flush, ChannelPromise promise) {
        //...
        // 找到Pipeline链表中下一个Outbound类型的ChannelHandler节点
        final AbstractChannelHandlerContext next = findContextOutbound(flush ?
                                                                       (MASK_WRITE | MASK_FLUSH) : MASK_WRITE);
        final Object m = pipeline.touch(msg, next);
        EventExecutor executor = next.executor();
        // 判断当前线程是否是NioEventLoop中的工作线程
        if (executor.inEventLoop()) {
            if (flush) {    // 因为flush == true，所以流程走到这里
                next.invokeWriteAndFlush(m, promise);
            } else {
                next.invokeWrite(m, promise);
            }
        } else {
            final WriteTask task = WriteTask.newInstance(next, m, promise, flush);
            if (!safeExecute(executor, task, promise, m, !flush)) {
                task.cancel();
            }
        }
    }
```

上述的`AbstractChannelHandlerContext.write`方法，执行逻辑如下：

1.  调用 `findContextOutbound` 方法找到 Pipeline 链表中的下一个 Outbound 类型的 ChannelHandler；
    
2.  通过 `inEventLoop` 方法判断当前线程是否为 NioEventLoop 中的工作线程，如果是则立即执行，否则封装成一个 Task 任务扔到 EventLoop 的任务队列中，稍后执行；
    
3.  因为 flush == true，所以直接执行 `next.invokeWriteAndFlush(m, promise)`这行代码，它的执行分两步：
    
    *   先执行 write 操作，即调用下一个 ChannelHandler 节点的 write 方法；
    *   再执行 flush 操作，真正将数据从底层的 SocketChannel 发送到对端。

```
    // AbstractChannelHandlerContext.java
    
    void invokeWriteAndFlush(Object msg, ChannelPromise promise) {
        if (invokeHandler()) {
            // 1.write流程
            invokeWrite0(msg, promise);
            // 2.flush流程（后面分析）
            invokeFlush0();
        } else {
            writeAndFlush(msg, promise);
        }
    }
```

可以看出，Netty 的 Pipeline 设计非常精妙，调用 writeAndFlush 时数据是在 Outbound 类型的 ChannelHandler 节点之间进行传播，直到 Head 节点结束，最终数据由 Head 节点调用底层的 SocketChannel 完成发送。

#### write 流程

我们先来看 write 流程，也就是 AbstractChannelHandlerContext 的 invokeWrite0 方法：

```
    // AbstractChannelHandlerContext.java
    
    private void invokeWrite0(Object msg, ChannelPromise promise) {
        try {
            // 调用ChannelHandler的write方法
            ((ChannelOutboundHandler) handler()).write(this, msg, promise);
        } catch (Throwable t) {
            notifyOutboundHandlerException(t, promise);
        }
    }
```

很简单，就是调用前一个 ChannelHandler 的 write 方法，层层递进直到 Head 节点，所以我们直接看 Head 节点的 write 方法源码：

```
    // HeadContext.java
    
    private final Unsafe unsafe;
    
    public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) {
        unsafe.write(msg, promise);
    }
```

可以看出，Head 节点是通过调用 Unsafe 对象完成数据写入的，Unsafe 对应的是`NioSocketChannelUnsafe`对象实例，内部最终调用到 `AbstractChannel.AbstractUnsafe.write()` 方法：

```
    // AbstractChannel.AbstractUnsafe.java
    
    public final void write(Object msg, ChannelPromise promise) {
        assertEventLoop();
    
        ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
        if (outboundBuffer == null) {
            try {
                ReferenceCountUtil.release(msg);
            } finally {
                safeSetFailure(promise, newClosedChannelException(initialCloseCause, "write(Object, ChannelPromise)"));
            }
            return;
        }
    
        int size;
        try {
            // 过滤消息
            msg = filterOutboundMessage(msg);
            size = pipeline.estimatorHandle().size(msg);
            if (size < 0) {
                size = 0;
            }
        } catch (Throwable t) {
            try {
                ReferenceCountUtil.release(msg);
            } finally {
                safeSetFailure(promise, t);
            }
            return;
        }
        // 向Buffer中添加数据
        outboundBuffer.addMessage(msg, size, promise);
    }
```

上述方法有两个重要的点需要指出：

1.  `filterOutboundMessage`方法会对待写入的 msg 进行过滤，如果 msg 使用的不是 DirectByteBuf，那么它会将 msg 转换成 DirectByteBuf；
2.  ChannelOutboundBuffer 可以理解为一个缓存结构，源码最后一行`outboundBuffer.addMessage`是在向这个缓存中添加数据，也就是说调用`AbstractUnsafe.write`方法只是将数据存储在 ChannelOutboundBuffer 的缓存内，而 ChannelOutboundBuffer 才是理解数据发送的关键。

下面我们重点分析一下 ChannelOutboundBuffer 的内部构造：

```
    // ChannelOutboundBuffer.java
    
    private Entry flushedEntry;
    private Entry unflushedEntry;
    private Entry tailEntry;
    
    public void addMessage(Object msg, int size, ChannelPromise promise) {
        // 将数据包装成Entry节点，入链表
        Entry entry = Entry.newInstance(msg, size, total(msg), promise);
        if (tailEntry == null) {
            flushedEntry = null;
        } else {
            Entry tail = tailEntry;
            tail.next = entry;
        }
        tailEntry = entry;
        if (unflushedEntry == null) {
            unflushedEntry = entry;
        }
        // 判断缓存水位线
        incrementPendingOutboundBytes(entry.pendingSize, false);
    }
```

ChannelOutboundBuffer 是一个单向链表结构，每次传入的数据都会被封装成一个 Entry 对象添加到链表中。ChannelOutboundBuffer 包含三个非常重要的指针：

1.  已经被 flush 到底层 Socket 缓冲区的 **节点 flushedEntry** ；
2.  第一个未被 flush 到底层 Socket 缓冲区的 **节点 unflushedEntry** ；
3.  最后一个 **节点 tailEntry。**

在初始状态下这三个指针都指向 NULL，当我们每次调用 write 方法时，都会调用 addMessage 方法改变这三个指针的指向，可以参考下图理解指针的移动过程：

![](/img/network-program/netty/ChannelOutboundBuffer.png)

上图中：

1.  第一次调用 write，因为链表里没有数据，所以 unflushedEntry 和 tailEntry 指针都指向第一个添加的数据 msg1。flushedEntry 指针在没有触发 flush 动作时会一直指向 NULL；
2.  第二次调用 write，tailEntry 指针会指向新加入的 msg2（链表的尾插法）；
3.  第 N 次调用 write，tailEntry 指针会不断指向新加入的 msgN，unflushedEntry 和 tailEntry 指针之间的数据都是未 flush 入 Socket 缓冲区的。

由于内存容量有限，所以 addMessage 方法每次写入数据后都会调用`incrementPendingOutboundBytes`方法判断 **缓存的水位线** ，具体源码如下：

```
    // ChannelOutboundBuffer.java
    
    private static final int DEFAULT_LOW_WATER_MARK = 32 * 1024;    // 低水位线32Kb
    private static final int DEFAULT_HIGH_WATER_MARK = 64 * 1024;    // 高水位线64Kb
    
    private void incrementPendingOutboundBytes(long size, boolean invokeLater) {
        if (size == 0) {
            return;
        }
    
        long newWriteBufferSize = TOTAL_PENDING_SIZE_UPDATER.addAndGet(this, size);
        if (newWriteBufferSize > channel.config().getWriteBufferHighWaterMark()) {
            setUnwritable(invokeLater);
        }
    }
```

上述`incrementPendingOutboundBytes`方法的逻辑非常简单，每次添加数据时都会累加已缓存的总数据字节数，然后判断大小是否超过所设置的高水位线（默认 64KB），如果超过则底层的 SocketChannel 会被设置为不可写状态（取消对`OP_WRITE`事件的监听），直到缓存数据大小低于低水位线（默认 32KB）后，SocketChannel 才恢复成可写状态。

至此，整个 write 流程就结束了，下面回到 flush 流程。

#### flush 流程

回到 AbstractChannelHandlerContext 的`invokeFlush0`方法，内部调用了上一个 ChannelHandler 的 flush 方法，同样会从 Tail 节点开始传播到 Head 节点：

```
    // AbstractChannelHandlerContext.java
    
    private void invokeFlush0() {
        try {
            // 调用上一个ChannelHandler的flush方法
            ((ChannelOutboundHandler) handler()).flush(this);
        } catch (Throwable t) {
            invokeExceptionCaught(t);
        }
    }
```

我们跟进下 HeadContext 的 flush 源码：

```
    // HeadContext.java
    
    private final Unsafe unsafe;
    
    public void flush(ChannelHandlerContext ctx) {
        unsafe.flush();
    }
```

内部调用`AbstractChannel.AbstractUnsafe.flush`方法：

```
    // AbstractChannel
    
    public final void flush() {
        assertEventLoop();
    
        ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
        if (outboundBuffer == null) {
            return;
        }
    
        outboundBuffer.addFlush();
        flush0();
    }
```

可以看出 flush 的核心逻辑主要分为两个步骤：`addFlush` 和 `flush0`，下面我们逐一对它们进行分析。

首先看下 addFlush 方法的源码：

```
    // ChannelOutboundBuffer.java
    
    public void addFlush() {
        Entry entry = unflushedEntry;
        if (entry != null) {
            if (flushedEntry == null) {
                flushedEntry = entry;
            }
            do {
                flushed ++;
                if (!entry.promise.setUncancellable()) {
                    int pending = entry.cancel();
                    // 减去待发送的数据，如果总字节数低于低水位，那么 Channel 将变为可写状态
                    decrementPendingOutboundBytes(pending, false, true);
                }
                entry = entry.next;
            } while (entry != null);
            unflushedEntry = null;
        }
    }
```

上述方法同样会操作 ChannelOutboundBuffer 中的缓存数据，此时 flushedEntry 指针有所改变，变更为 unflushedEntry 指针所指向的数据，然后 unflushedEntry 指针指向 NULL，flushedEntry 指针指向的数据会被真正发送到 Socket 缓冲区，如下图：

![](/img/network-program/netty/ChannelOutboundBuffer-flush.png)

接着再来看下 flush0 方法的源码：

```
    // AbstractChannelHandlerContext.java
    
    protected void flush0() {
        if (inFlush0) {
            return;
        }
    
        final ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
        if (outboundBuffer == null || outboundBuffer.isEmpty()) {
            return;
        }
    
        inFlush0 = true;
        if (!isActive()) {
            try {
                if (!outboundBuffer.isEmpty()) {
                    if (isOpen()) {
                        outboundBuffer.failFlushed(new NotYetConnectedException(), true);
                    } else {              outboundBuffer.failFlushed(newClosedChannelException(initialCloseCause, "flush0()"), false);
                    }
                }
            } finally {
                inFlush0 = false;
            }
            return;
        }
    
        try {
            doWrite(outboundBuffer);
        } catch (Throwable t) {
            handleWriteError(t);
        } finally {
            inFlush0 = false;
        }
    }
```

flush0 的实际调用层次很深，但其实核心的逻辑在于 doWrite 方法：

```
    // NioSocketChannel.java
    
    protected void doWrite(ChannelOutboundBuffer in) throws Exception {
        // Java NIO的SocketChannel
        SocketChannel ch = javaChannel();
        int writeSpinCount = config().getWriteSpinCount();
        do {
            if (in.isEmpty()) {
                clearOpWrite();
                return;
            }
    
            int maxBytesPerGatheringWrite = ((NioSocketChannelConfig)config).getMaxBytesPerGatheringWrite();
            ByteBuffer[] nioBuffers = in.nioBuffers(1024, maxBytesPerGatheringWrite);
            int nioBufferCnt = in.nioBufferCount();
            switch (nioBufferCnt) {
                case 0:
                    writeSpinCount -= doWrite0(in);
                    break;
                case 1: {
                    ByteBuffer buffer = nioBuffers[0];
                    int attemptedBytes = buffer.remaining();
                    final int localWrittenBytes = ch.write(buffer);
                    if (localWrittenBytes <= 0) {
                        incompleteWrite(true);
                        return;
                    }
                    adjustMaxBytesPerGatheringWrite(attemptedBytes, localWrittenBytes, maxBytesPerGatheringWrite);
                    in.removeBytes(localWrittenBytes);
                    --writeSpinCount;
                    break;
                }
                default: {
                    long attemptedBytes = in.nioBufferSize();
                    final long localWrittenBytes = ch.write(nioBuffers, 0, nioBufferCnt);
                    if (localWrittenBytes <= 0) {
                        incompleteWrite(true);
                        return;
                    }
                    adjustMaxBytesPerGatheringWrite((int) attemptedBytes, (int) localWrittenBytes,
                                                    maxBytesPerGatheringWrite);
                    in.removeBytes(localWrittenBytes);
                    --writeSpinCount;
                    break;
                }
            }
        } while (writeSpinCount > 0);
        incompleteWrite(writeSpinCount < 0);
    }
```

该方法负责将数据真正写入到 Socket 缓冲区。doWrite 方法的处理流程主要分为三步：

1.  根据配置获取自旋锁的次数`writeSpinCount`。当我们向 Socket 底层写数据的时候，如果每次要写入的数据量很大，是不可能一次将数据写完的，所以只能分批写入。Netty 在不断调用执行写入逻辑的时候，EventLoop 线程可能一直在等待，这样有可能会阻塞其他事件处理。所以这里自旋锁的次数相当于控制一次写入数据的最大的循环执行次数，如果超过所设置的自旋锁次数，那么写操作将会被暂时中断；
2.  根据自旋锁次数，重复调用`doWriteInternal`方法发送数据，每成功发送一次数据，自旋锁的次数 writeSpinCount 减 1，当 writeSpinCount 耗尽，那么 doWrite 操作将会被暂时中断。doWriteInternal 的源码涉及 JDK NIO 底层，在这里我不再深入展开，它的主要作用在于删除 ChannelOutboundBuffer 缓存中的链表节点以及调用底层 API 发送数据；
3.  调用`incompleteWrite`方法确保数据能够全部发送出去，因为自旋锁次数的限制，可能数据并没有写完，所以需要继续监听 `OP_WRITE`事件；如果数据已经写完，清除`OP_WRITE`事件即可。

* * *

至此，整个 writeAndFlush 的工作原理已经全部分析完了，整个过程的调用层次比较深，我整理了 writeAndFlush 的时序图，如下所示，帮助大家梳理 writeAndFlush 的调用流程，加深对上述知识点的理解：

![](/img/network-program/netty/ChannelPipeline-writeAndFlush.png)

二、ChannelHandler
----------------

ChannelHandler 是负责业务处理的处理器，我们使用 Netty 进行开发时，主要的工作就是 ChannelHandler 的开发。前面已经讲了，ChannelHandler 分为 **入站（Inbound）** 和 **出站（Outbound）** 两种类型，分别对应`ChannelInboundHandler`和`ChannelOutboundHandler`这两个接口。

### 2.1 入站处理器

ChannelInboundHandler 接口定义了很多与入站事件相关的回调方法，每一个回调方法的触发时机如下：

<table><thead><tr><th>事件回调方法</th><th>触发时机</th></tr></thead><tbody><tr><td>channelRegistered</td><td>Channel 被注册到 EventLoop</td></tr><tr><td>channelUnregistered</td><td>Channel 从 EventLoop 中取消注册</td></tr><tr><td>channelActive</td><td>Channel 处于就绪状态，可以被读写</td></tr><tr><td>channelInactive</td><td>Channel 处于非就绪状态</td></tr><tr><td>channelRead</td><td>Channel 可以从远端读取到数据</td></tr><tr><td>channelReadComplete</td><td>Channel 读取数据完成</td></tr><tr><td>userEventTriggered</td><td>用户事件触发时</td></tr><tr><td>channelWritabilityChanged</td><td>Channel 的写状态发生变化</td></tr></tbody></table>

ChannelInboundHandler 的默认实现为`ChannelInboundHandlerAdapter`，我们自己开发的入站 ChannelHandler 一般只要继承该类即可。

### 2.2 出站处理器

ChannelOutboundHandler 接口定义了很多与出站事件相关的回调方法：

<table><thead><tr><th>事件回调方法</th><th>触发时机</th></tr></thead><tbody><tr><td>bind</td><td>监听地址（IP + 端口）绑定：完成底层 JavaIO 通道的地址绑定</td></tr><tr><td>connect</td><td>连接服务端：完成底层 JavaIO 通道的服务器端的连接操作</td></tr><tr><td>disconnect</td><td>断开服务器连接：断开底层 JavaIO 通道的服务器端连接</td></tr><tr><td>close</td><td>主动关闭通道：关闭底层的通道，例如服务器端的新连接监听通道</td></tr><tr><td>write</td><td>写数据到底层：完成 Netty 通道向底层 JavaIO 通道的数据写入操作。此方法仅仅是触发一下操作而已，并不是完成实际的数据写入操作。</td></tr><tr><td>flush</td><td>清空缓冲区数据，将数据写到对端</td></tr></tbody></table>

ChannelOutboundHandler 的默认实现为`ChannelOutboundHandlerAdapter`，我们自己开发的出站 ChannelHandler 一般只要继承该类即可。

> ChannelOutboundHandler 中绝大部分接口都包含`ChannelPromise`参数，便于在操作完成时能够及时获得通知。

### 2.3 生命周期

为了弄清上面的 Handler 业务处理器的各个方法的执行顺序和生命周期，我这里定义一个简单的入站 Handler 处理器——InHandlerDemo。这个类继承于 ChannelInboundHandlerAdapter 适配器，它实现了基类的大部分入站处理方法，并在每一个方法的实现代码中都加上必要的输出信息，以便观察方法是否被执行到：

```
    public class ChannelHandlerDemo {
    
        static class InHandlerDemo extends ChannelInboundHandlerAdapter {
            @Override
            public void channelRegistered(ChannelHandlerContext ctx) throws Exception {
                System.out.println("调用方法：channelRegistered");
                super.channelRegistered(ctx);
            }
    
            @Override
            public void channelUnregistered(ChannelHandlerContext ctx) throws Exception {
                System.out.println("调用方法：channelUnregistered");
                super.channelUnregistered(ctx);
            }
    
            @Override
            public void channelActive(ChannelHandlerContext ctx) throws Exception {
                System.out.println("调用方法：channelActive");
                super.channelActive(ctx);
            }
    
            @Override
            public void channelInactive(ChannelHandlerContext ctx) throws Exception {
                System.out.println("调用方法：channelInactive");
                super.channelInactive(ctx);
            }
    
            @Override
            public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
                System.out.println("调用方法：channelRead");
                super.channelRead(ctx, msg);
            }
    
            @Override
            public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
                System.out.println("调用方法：channelReadComplete");
                super.channelReadComplete(ctx);
            }
    
            @Override
            public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
                System.out.println("调用方法：handlerAdded");
                super.handlerAdded(ctx);
            }
    
            @Override
            public void handlerRemoved(ChannelHandlerContext ctx) throws Exception {
                System.out.println("调用方法：handlerRemoved");
                super.handlerRemoved(ctx);
            }
        }
    
        public static void main(String[] args) {
            ChannelInitializer channelInitializer = new ChannelInitializer<EmbeddedChannel>() {
                @Override
                protected void initChannel(EmbeddedChannel ch) throws Exception {
                    ch.pipeline().addLast(new InHandlerDemo());
                }
            };
            EmbeddedChannel channel = new EmbeddedChannel(channelInitializer);
            ByteBuf buf = Unpooled.buffer();
            buf.writeInt(90);
            //模拟入站，写一个入站数据包
            channel.writeInbound(buf);
            channel.flush();
            //模拟入站，再写一个入站数据包
            channel.writeInbound(buf);
            channel.flush();
            //通道关闭
            channel.close();
        }
    }
```

输出结果如下，通过执行结果可以清晰的看出 ChannelInboundHandler 处理 I/O 事件的过程：

```
    调用方法：handlerAdded
    调用方法：channelRegistered
    调用方法：channelActive
    调用方法：channelRead
    调用方法：channelReadComplete
    调用方法：channelRead
    调用方法：channelReadComplete
    调用方法：channelInactive
    调用方法：channelUnregistered
    调用方法：handlerRemoved
```

上述的方法中，`channelRead()`和`channelReadComplete()`方法每次有入站请求时都会调用，其余方法和 ChannelHandler 的生命周期有关：

*   handlerAdded：当 ChannelHandler 被加入到 Pipeline 后，此方法被回调。也就是执行完`ch.pipeline().addLast(handler)`语句之后回调；
*   channelRegistered：当 Channel 成功注册到一个 NioEventLoop 上之后，会通过 Pipeline 回调所有 ChannelHandler 的 channelRegistered 方法；
*   channelUnregistered：当 Channel 和 NioEventLoop 线程解除绑定，移除掉对这条通道的事件处理之后，会通过 Pipeline 回调所有 ChannelHandler 的 channelUnregistered 方法；
*   channelActive：当 Channel 处于就绪状态，可以被读写时，会通过 Pipeline 回调所有 ChannelHandler 的 channelActive 方法；
*   channelInactive：当 Channel 的底层连接已经不是 ESTABLISH 状态，或者底层连接已经关闭时，会首先通过 Pipeline 回调所有 ChannelHandler 的 channelInactive 方法；
*   handlerRemoved：当 Channel 关闭后，Netty 会移除掉通道上所有 ChannelHandler，并通过 Pipeline 回调所有 ChannelHandler 的 handlerRemoved() 方法。

> 对于出站处理器 ChannelOutboundHandler 的生命周期以及回调的顺序，与入站处理器是大致相同的，我这里不再赘述。

### 2.4 ChannelInitializer

ChannelInitializer 是一种特殊的入站处理器 ，负责向 Pipieline 流水线中装配业务处理器。我们在使用 Bootstrap 引导器时，就需要使用到它。ChannelInitializer 的源码比较简单，我们直接看源码理解下：

```
    // ChannelInitializer.java
    
    public abstract class ChannelInitializer<C extends Channel> extends ChannelInboundHandlerAdapter {
    
        private final Set<ChannelHandlerContext> initMap = Collections.newSetFromMap(
                new ConcurrentHashMap<ChannelHandlerContext, Boolean>());
    
        /**
         * 当Channel注册完成后，会调用该方法
         */
        protected abstract void initChannel(C ch) throws Exception;
    
        @Override
        public final void channelRegistered(ChannelHandlerContext ctx) throws Exception {
            // 初始化Channel
            if (initChannel(ctx)) {
                // 传播register事件
                ctx.pipeline().fireChannelRegistered();
                removeState(ctx);
            } else {
                ctx.fireChannelRegistered();
            }
        }
    
        @Override
        public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
            // 关闭Channel
            ctx.close();
        }
    
        /**
         * {@inheritDoc} If override this method ensure you call super!
         */
        @Override
        public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
            if (ctx.channel().isRegistered()) {
                if (initChannel(ctx)) {
                    removeState(ctx);
                }
            }
        }
    
        @Override
        public void handlerRemoved(ChannelHandlerContext ctx) throws Exception {
            initMap.remove(ctx);
        }
    
        @SuppressWarnings("unchecked")
        private boolean initChannel(ChannelHandlerContext ctx) throws Exception {
            if (initMap.add(ctx)) {
                try {
                    initChannel((C) ctx.channel());
                } catch (Throwable cause) {
                    exceptionCaught(ctx, cause);
                } finally {
                    // 将自己从pipeline移除
                    ChannelPipeline pipeline = ctx.pipeline();
                    if (pipeline.context(this) != null) {
                        pipeline.remove(this);
                    }
                }
                return true;
            }
            return false;
        }
    
        private void removeState(final ChannelHandlerContext ctx) {
            if (ctx.isRemoved()) {
                initMap.remove(ctx);
            } else {
                ctx.executor().execute(new Runnable() {
                    @Override
                    public void run() {
                        initMap.remove(ctx);
                    }
                });
            }
        }
    }
```

### 2.5 异常处理

我之前在讲解出 / 入站处理器时，提到 Inbound 事件和 Outbound 事件在 Pipeline 中的传播方向相反，Inbound 事件的传播方向为 Head -> Tail，而 Outbound 事件传播方向是 Tail -> Head。

但有一种情况是例外的，那就是异常传播： **异常事件的传播顺序与 ChannelHandler 的添加顺序相同，会依次向后传播，与 Inbound 事件和 Outbound 事件无关** 。我们通过一个例子看下：

```
    public class ExceptionDemo {
        static class ExceptionInHandler extends ChannelInboundHandlerAdapter {
            private String name;
    
            ExceptionInHandler(String name) {
                this.name = name;
            }
    
            @Override
            public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
                System.out.println("ExceptionInHandler[" + name + "]：channelRead");
                throw new RuntimeException("ExceptionInHandler[" + name + "] Error");
            }
    
            @Override
            public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
                System.out.println("ExceptionInHandler[" + name + "] ERROR!!!!!");
                super.exceptionCaught(ctx, cause);
            }
        }
    
        static class ExceptionOutHandler extends ChannelOutboundHandlerAdapter {
            private String name;
            ExceptionOutHandler(String name) {
                this.name = name;
            }
    
            @Override
            public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
                System.out.println("ExceptionOutHandler[" + name + "] ERROR!!!!!");
                super.exceptionCaught(ctx, cause);
            }
        }
    
        public static void main(String[] args) {
            ChannelInitializer channelInitializer = new ChannelInitializer<EmbeddedChannel>() {
                @Override
                protected void initChannel(EmbeddedChannel ch) throws Exception {
                    ch.pipeline()
                            .addLast(new ExceptionInHandler("IN_A"))
                            .addLast(new ExceptionInHandler("IN_B"))
                            .addLast(new ExceptionInHandler("IN_C"));
    
                    ch.pipeline()
                            .addLast(new ExceptionOutHandler("OUT_A"))
                            .addLast(new ExceptionOutHandler("OUT_B"))
                            .addLast(new ExceptionOutHandler("OUT_C"));
                }
            };
            EmbeddedChannel channel = new EmbeddedChannel(channelInitializer);
            ByteBuf buf = Unpooled.buffer();
            buf.writeInt(90);
            channel.writeInbound(buf);
        }
    }
```

上面代码中，我定义了三个入站处理器和三个出站处理器，并按照 `` 的顺序组装到 Pipeline 中，输出结果如下，说明异常会在整个 Pipeline 中按照 Handler 添加的顺序传播：

```
    ExceptionInHandler[IN_A]：channelRead
    ExceptionInHandler[IN_A] ERROR!!!!!
    ExceptionInHandler[IN_B] ERROR!!!!!
    ExceptionInHandler[IN_C] ERROR!!!!!
    ExceptionOutHandler[OUT_A] ERROR!!!!!
    ExceptionOutHandler[OUT_B] ERROR!!!!!
    ExceptionOutHandler[OUT_C] ERROR!!!!!
    Exception in thread "main" java.lang.RuntimeException: ExceptionInHandler[IN_A] Error
        at com.tpvlog.netty.ExceptionDemo$ExceptionInHandler.channelRead(ExceptionDemo.java:22)
```

### 2.6 共享 ChannelHandler

我们经常使用以下 `new HandlerXXX()` 的方式进行 Channel 初始化，在每建立一个新连接的时候会初始化新的 HandlerA 和 HandlerB，如果系统承载了 1w 个连接，那么就会初始化 2w 个处理器，造成非常大的内存浪费：

```
    ServerBootstrap b = new ServerBootstrap();
    b.group(bossGroup, workerGroup)
        .channel(NioServerSocketChannel.class)
        .localAddress(new InetSocketAddress(port))
        .childHandler(new ChannelInitializer<SocketChannel>() {
            @Override
            public void initChannel(SocketChannel ch) {
                ch.pipeline()
                    .addLast(new HandlerA())
                    .addLast(new HandlerB());
            }
        });
```

为了解决上述问题，Netty 提供了 `@Sharable` 注解用于修饰 ChannelHandler，标识该 ChannelHandler 全局只有一个实例，而且会被多个 ChannelPipeline 共享。所以我们必须要注意，`@Sharable` 修饰的 ChannelHandler 必须都是无状态的，这样才能保证线程安全。

三、总结
----

本章，我对 ChannelPipeline 和 ChannelHandler 的原理进行了分析，最后总结一下：

*   ChannelPipeline 是双向链表结构，包含 ChannelInboundHandler 和 ChannelOutboundHandler 两种处理器。
*   ChannelHandlerContext 是对 ChannelHandler 的封装，每个 ChannelHandler 都对应一个 ChannelHandlerContext，实际上 ChannelPipeline 维护的是与 ChannelHandlerContext 的关系；
*   Inbound 事件和 Outbound 事件的传播方向相反，Inbound 事件的传播方向为 Head -> Tail，而 Outbound 事件传播方向是 Tail -> Head；
*   异常事件的处理顺序与 ChannelHandler 的添加顺序相同，会依次向后传播，与 Inbound 事件和 Outbound 事件无关。