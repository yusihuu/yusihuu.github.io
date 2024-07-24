---
layout:     post
title:      透彻理解 java 网络编程(十九)
subtitle:   netty 原理：Mpsc Queue 无锁队列
date:       2024-07-19 14:00:00
author:     yusihu
header-style: text
catalog: true
tags:
    - 网络编程
    - netty
    - 开源框架
---
Netty 中大量使用了一种名为 Mpsc Queue 的无锁队列。比如，上一章我在分析 HashedWheelTimer 时间轮时，就提到过它内部就是通过 Mpsc Queue 保存待执行的任务。

为什么 Netty 不使用 JDK 原生并发包的队列呢？Mpsc Queue 有什么特点？适用于什么样的场景？本章，我就对 Mpsc Queue 的底层原理进行剖析。

一、并发队列
------

我曾经在[《透彻理解 Java 并发编程》](https://www.tpvlog.com/article/17)专栏中，对 J.U.C 包中的所有并发队列进行过源码分析，本节我先带大家回顾下，作为后续讲解 Mpsc Queue 的铺垫。

### 1.1 JDK 并发队列

J.U.C 包中的并发队列按照阻塞方式归类可以分为 **阻塞队列** 和 **非阻塞队列** 两种类型：

![](/img/network-program/netty/ConcurrentQueue.png)

#### 阻塞队列

阻塞队列在队列为空或者队列满时，都会发生阻塞。阻塞队列自身是线程安全的，使用者无需关心线程安全问题，降低了多线程开发难度。阻塞队列主要分为以下几种：

*   **ArrayBlockingQueue** ：底层采用数组实现的有界队列，初始化需要指定队列的容量。ArrayBlockingQueue 是如何保证线程安全的呢？它内部是使用了一个重入锁 ReentrantLock，并搭配 notEmpty、notFull 两个条件变量 Condition 来控制并发访问。从队列读取数据时，如果队列为空，那么会阻塞等待，直到队列有数据了才会被唤醒。如果队列已经满了，也同样会进入阻塞状态，直到队列有空闲才会被唤醒。
*   **LinkedBlockingQueue** ：底层采用的数据结构是链表，队列的长度可以是有界或者无界的，初始化不需要指定队列长度，默认是 Integer.MAX_VALUE。LinkedBlockingQueue 内部使用了 takeLock、putLock 两个重入锁 ReentrantLock，以及 notEmpty、notFull 两个条件变量 Condition 来控制并发访问。采用读锁和写锁的好处是可以避免读写时相互竞争锁的现象，所以相比于 ArrayBlockingQueue，LinkedBlockingQueue 的性能要更好。
*   **PriorityBlockingQueue** ：底层最小堆实现的优先级队列，队列中的元素按照优先级进行排列，每次出队都是返回优先级最高的元素。PriorityBlockingQueue 内部是使用了一个 ReentrantLock 以及一个条件变量 Condition notEmpty 来控制并发访问，不需要 notFull 是因为 PriorityBlockingQueue 是无界队列，所以每次 put 都不会发生阻塞。PriorityBlockingQueue 底层的最小堆是采用数组实现的，当元素个数大于等于最大容量时会触发扩容，在扩容时会先释放锁，保证其他元素可以正常出队，然后使用 CAS 操作确保只有一个线程可以执行扩容逻辑。
*   **DelayQueue** ：一种支持延迟获取元素的阻塞队列，常用于缓存、定时任务调度等场景。DelayQueue 内部是采用优先级队列 PriorityQueue 存储对象。DelayQueue 中的每个对象都必须实现 Delayed 接口，并重写 compareTo 和 getDelay 方法。向队列中存放元素的时候必须指定延迟时间，只有延迟时间已满的元素才能从队列中取出。
*   **SynchronizedQueue** ：又称无缓冲队列。比较特别的是 SynchronizedQueue 内部不会存储元素。与 ArrayBlockingQueue、LinkedBlockingQueue 不同，SynchronizedQueue 直接使用 CAS 操作控制线程的安全访问。其中 put 和 take 操作都是阻塞的，每一个 put 操作都必须阻塞等待一个 take 操作，反之亦然。所以 SynchronizedQueue 可以理解为生产者和消费者配对的场景，双方必须互相等待，直至配对成功。在 JDK 的线程池 Executors.newCachedThreadPool 中就存在 SynchronousQueue 的运用，对于新提交的任务，如果有空闲线程，将重复利用空闲线程处理任务，否则将新建线程进行处理。
*   **LinkedTransferQueue** ：一种特殊的无界阻塞队列，可以看作 LinkedBlockingQueues、SynchronousQueue（公平模式）、ConcurrentLinkedQueue 的合体。与 SynchronousQueue 不同的是，LinkedTransferQueue 内部可以存储实际的数据，当执行 put 操作时，如果有等待线程，那么直接将数据交给对方，否则放入队列中。与 LinkedBlockingQueues 相比，LinkedTransferQueue 使用 CAS 无锁操作进一步提升了性能。

#### 非阻塞队列

非阻塞队列不需要通过加锁的方式对线程阻塞，并发性能更好。JDK 中常用的非阻塞队列有以下几种：

*   **ConcurrentLinkedQueue** ：采用双向链表实现的无界并发非阻塞队列，它属于 LinkedQueue 的安全版本。ConcurrentLinkedQueue 内部采用 CAS 操作保证线程安全，这是非阻塞队列实现的基础，相比 ArrayBlockingQueue、LinkedBlockingQueue 具备较高的性能。
*   **ConcurrentLinkedDeque** ：也是一种采用双向链表结构的无界并发非阻塞队列。与 ConcurrentLinkedQueue 不同的是，ConcurrentLinkedDeque 属于双端队列，它同时支持 FIFO 和 FILO 两种模式，可以从队列的头部插入和删除数据，也可以从队列尾部插入和删除数据，适用于多生产者和多消费者的场景。

### 1.2 第三方并发队列

JDK 提供的并发队列已经能够满足我们大部分的需求，但是在大规模流量的高并发系统中，如果你对性能要求严苛，JDK 的并发队列可能并不能满足你的需求。因此，一些第三方框架提供了解决方案，非常出名的有 Disruptor 和 JCTools。

#### Disruptorn

Disruptor 是 LMAX 公司开发的一款高性能无锁队列，我们平时常称它为 RingBuffer，其设计初衷是为了解决内存队列的延迟问题。Disruptor 内部采用环形数组和 CAS 操作实现，性能非常优越。为什么 Disruptor 的性能会比 JDK 原生的无锁队列要好呢？环形数组可以复用内存，减少分配内存和释放内存带来的性能损耗。而且数组可以设置长度为 2 的次幂，直接通过位运算加快数组下标的定位速度。此外，Disruptor 还解决了伪共享问题，对 CPU Cache 更加友好。Disruptor 已经开源，详细可查阅 Github 地址 [https://github.com/LMAX-Exchange/disruptor。](https://github.com/LMAX-Exchange/disruptor%E3%80%82)

#### JCTools

JCTools 也是一个开源项目，Github 地址为 [https://github.com/JCTools/JCTools。JCTools](https://github.com/JCTools/JCTools) 是适用于 JVM 并发开发的工具，主要提供了一些 JDK 缺失的并发数据结构。我们主要看它提供的并发队列，一共可分为四种类型：

*   Spsc 单生产者单消费者；
*   Mpsc 多生产者单消费者；
*   Spmc 单生产者多消费者；
*   Mpmc 多生产者多消费者。

Netty 直接引入了 JCTools 的 Mpsc Queue。

二、Mpsc Queue
------------

Mpsc 的全称是 Multi Producer Single Consumer，多生产者单消费者。Mpsc Queue 可以保证多个生产者同时访问队列是线程安全的，而且同一时刻只允许一个消费者从队列中读取数据。

Netty Reactor 线程中的任务队列 taskQueue 必须满足多个生产者可以同时提交任务，所以 JCTools 提供的 Mpsc Queue 非常适合 Netty Reactor 线程模型。

Mpsc Queue 有多种的实现类，例如 MpscArrayQueue、MpscUnboundedArrayQueue、MpscChunkedArrayQueue 等。 **本章， 我只介绍 MpscArrayQueue** ，通过对它的分析，基本可以让大家对 Mpsc Queue 有一个比较深入的认识，其余队列感兴趣的童鞋可以自行阅读源码。

### 2.1 使用示例

我们先来看下 MpscArrayQueue 的基本使用，和普通的阻塞队列没有什么区别：

```
    public class MpscArrayQueueTest {
    
        public static final MpscArrayQueue<String> MPSC_ARRAY_QUEUE = new MpscArrayQueue<>(2);
    
        public static void main(String[] args) {
            for (int i = 1; i <= 2; i++) {
                final int index = i;
                new Thread(new Runnable() {
                    @Override
                    public void run() {
                        // offer入队操作，队列满则阻塞
                        MPSC_ARRAY_QUEUE.offer("data" + index);
                    }
                }, "Thread-" + i).start();
            }
    
            try {
                Thread.sleep(1000L);
                // add入队操作，队列满则抛出异常
                MPSC_ARRAY_QUEUE.add("data3");
            } catch (Exception e) {
                e.printStackTrace();
            }
    
            System.out.println("队列大小：" + MPSC_ARRAY_QUEUE.size() + ", 队列容量：" + MPSC_ARRAY_QUEUE.capacity());
            // remove出队操作，队列为空则抛出异常
            System.out.println("出队：" + MPSC_ARRAY_QUEUE.remove());
            // poll出队操作，队列为空则返回 NULL
            System.out.println("出队：" + MPSC_ARRAY_QUEUE.poll());
        }
    }
```

输出结果如下：

```
    java.lang.IllegalStateException: Queue full
        at java.util.AbstractQueue.add(AbstractQueue.java:98)
        at com.tpvlog.im.gateway.MpscArrayQueueTest.main(MpscArrayQueueTest.java:24)
    队列大小：2, 队列容量：2
    出队：data1
    出队：data2
```

MpscArrayQueue 提供了以下入队 / 出队的操作：

<table><thead><tr><th>接口</th><th>行为</th><th>是否抛出异常</th></tr></thead><tbody><tr><td>offer</td><td>入队元素</td><td>否，队列满时线程阻塞等待</td></tr><tr><td>add</td><td>入队元素</td><td>是，队列满时抛出异常</td></tr><tr><td>poll</td><td>出队元素</td><td>否，队列空时返回 null</td></tr><tr><td>remove</td><td>出队元素</td><td>是，队列空时抛出异常</td></tr></tbody></table>

### 2.2 继承体系

来看下 MpscArrayQueue 的继承关系：

![](/img/network-program/netty/MpscArrayQueue.png)

可以看到，整个继承体系还是很复杂的，MpscArrayQueue 除了继承 JDK 原生的 AbstractCollection、AbstractQueue 外，还继承了很多类似于 MpscXxxPad 以及 MpscXxxField 的类。MpscXxxPad 起到什么作用呢？我们自顶向下，将所有类的字段合并在一起，看下 MpscArrayQueue 的整体结构：

```
    // ConcurrentCircularArrayQueueL0Pad.java
    
    long p01, p02, p03, p04, p05, p06, p07;
    long p10, p11, p12, p13, p14, p15, p16, p17;
    
    // ConcurrentCircularArrayQueue.java
    
    protected final long mask;
    protected final E[] buffer;
    
    // MpmcArrayQueueL1Pad.java
    
    long p00, p01, p02, p03, p04, p05, p06, p07;
    long p10, p11, p12, p13, p14, p15, p16;
    
    // MpmcArrayQueueProducerIndexField.java
    
    private volatile long producerIndex;
    
    // MpscArrayQueueMidPad.java
    
    long p01, p02, p03, p04, p05, p06, p07;
    long p10, p11, p12, p13, p14, p15, p16, p17;
    
    // MpscArrayQueueProducerLimitField.java
    
    private volatile long producerLimit;
    
    // MpscArrayQueueL2Pad.java
    
    long p00, p01, p02, p03, p04, p05, p06, p07;
    long p10, p11, p12, p13, p14, p15, p16;
    
    // MpscArrayQueueConsumerIndexField.java
    
    protected long consumerIndex;
    
    // MpscArrayQueueL3Pad.java
    
    long p01, p02, p03, p04, p05, p06, p07;
    long p10, p11, p12, p13, p14, p15, p16, p17;
```

可以看出，MpscXxxPad 类中使用了大量 long 类型的变量，其命名没有什么特殊的含义，只是起到填充的作用。如果你读过 Disruptor 的源码，会发现 Disruptor 也使用了类似的填充方法。Mpsc Queue 和 Disruptor 之所以填充这些无意义的变量，是为了解决 **伪共享（false sharing）** 问题。

### 2.3 伪共享问题

什么是伪共享呢？这要从 CPU 的缓存架构说起。

#### CPU 缓存架构

在计算机组成中，CPU 的运算速度比内存高出几个数量级，为了 CPU 能够更高效地与内存进行交互，在 CPU 和内存之间设计了多层缓存机制。一般来说，CPU 分为三级缓存，分别为 **L1 一级缓存** 、 **L2 二级缓存** 和 **L3 三级缓存** ，越靠近 CPU 的缓存，速度越快，但是缓存的容量也越小，L3 三级缓存一般被所有 CPU 核共享。所以从性能上来说，L1 > L2 > L3，容量方面 L1 < L2 < L3。

![](/img/network-program/netty/CPU-cache.png)

CPU 读取数据时，首先会从 L1 查找，如果未命中则继续查找 L2，如果还未能命中则继续查找 L3，最后还没命中的话只能从内存中查找，读取完成后再将数据逐级放入缓存中。此外，多线程之间共享一份数据的时候，需要其中一个线程将数据写回主存，其他线程访问主存数据。

#### 缓存行

CPU 缓存由若干个缓存行（Cache Line） 组成， **缓存行是 CPU 缓存可操作的最小单位** 。Cache Line 的大小与 CPU 架构有关，在目前主流的 64 位架构下，Cache Line 的大小通常为 64 Byte。而 Java 中一个 long 类型是 8 Byte，所以一个 Cache Line 可以存储 8 个 long 类型变量。

CPU 在加载内存数据时，会将相邻的数据一同读取到 Cache Line 中（一次加载连续的 64 个字节），这样就可以避免 CPU 频繁与内存进行交互了。

举个两个例子来理解：

1.  如果访问一个 long 型数组，当数组中的一个值被加载到缓存中时，另外 7 个元素也会被加载到缓存中；
2.  如果访问一个 long 型的单独变量 a，并且还有另外一个 long 型变量 b 紧挨着它，那么当加载 a 时候将免费加载 b。

理解了上述概念，我们来看伪共享是如何发生的：

1.  假设有 A、B、C、D 四个变量，线程 1 尝试修改变量 A，于是将 A 和 B、C、D 一起都加载到了 CPU1 的一个 Cache Line；
2.  此时，线程 2 读取变量 B，也将 A、C、D 加载到了 CPU2 的同一 Cache Line；
3.  线程 1 对变量 A 进行修改，修改完成后将变量 A 值写回主存，然后 CPU1 会通知 CPU2 该缓存行已经失效；
4.  线程 2 在 CPU Core2 中对变量 C 进行修改时，发现 Cache line 已经失效，所以需要再从主存中读取数据加载到当前 Cache line 中。

![](/img/network-program/netty/CPU-cache-line.png)

上述这个现象就是伪共享： **当多个线程同时修改互相独立的变量时，如果这些变量共享同一个缓存行，就会出现写竞争，导致频繁从主存加载数据，影响性能** 。

#### 解决方案

针对伪共享问题，常见的解决思路就是： **以空间换时间，让不同线程操作的不相干变量加载到不同缓存行，避免相互影响** 。

举个例子：

```
    public class FalseSharingPadding {
        protected long p1, p2, p3, p4, p5, p6, p7;
        protected volatile long value = 0L;
        protected long p9, p10, p11, p12, p13, p14, p15;
    }
```

上述代码中，变量 value 前后分别填充了 7 个 long 类型的变量。这样不论在什么情况下，都可以保证在多线程访问 value 变量时，value 与其他不相关的变量处于不同的 Cache Line，如下图所示：

![](/img/network-program/netty/CPU-cache-line1.png)

此外，我们还可以利用 Java 8 中的`@sun.misc.Contended`注解，对某字段加上该注解表示该字段会单独占用一个缓存行：

```
    @sun.misc.Contended
    class MyLong {
        volatile long value;
    }
```

> 注：JVM 添加 `-XX:-RestrictContended` 参数后 `@sun.misc.Contended` 注解才会有效。

### 2.4 源码分析

本节，我将对 MpscArrayQueue 的底层源码进行剖析，先回顾下 MpscArrayQueue 的重要属性，大部分都继承自父类：

```
    // ConcurrentCircularArrayQueue.java
    
    // 计算数组下标的掩码
    protected final long mask; 
    // 存放队列数据的数组
    protected final E[] buffer; 
    
    // MpmcArrayQueueProducerIndexField.java
    
    // 生产者索引
    private volatile long producerIndex; 
    
    // MpscArrayQueueProducerLimitField.java
    
    // 生产者索引的最大值
    private volatile long producerLimit; 
    
    // MpscArrayQueueConsumerIndexField.java
    
    // 消费者索引
    protected long consumerIndex;
```

#### offer()

offer 方法用于向队列入队元素，虽然比较简短，但是需要具备一些底层知识才能看得懂：

```
    // MpscArrayQueue.java
    
    public boolean offer(final E e) {
        if (null == e)
        {
            throw new NullPointerException();
        }
    
        final long mask = this.mask;
    
        // 获取生产者索引最大限制
        long producerLimit = lvProducerLimit();
        long pIndex;
        do
        {
            // 获取生产者索引
            pIndex = lvProducerIndex();
            if (pIndex >= producerLimit)
            {
                // 获取消费者索引
                final long cIndex = lvConsumerIndex();
                producerLimit = cIndex + mask + 1;
    
                if (pIndex >= producerLimit)
                {
                    return false; // 队列已满
                }
                else
                {
                    soProducerLimit(producerLimit);        // 更新 producerLimit
                }
            }
        } while (!casProducerIndex(pIndex, pIndex + 1));// CAS 更新生产者索引，更新成功则退出，说明当前生产者已经占领索引值
        // 计算生产者索引在数组中下标
        final long offset = calcCircularRefElementOffset(pIndex, mask);
        // 向数组中放入数据
        soRefElement(buffer, offset, e);
        return true; 
    }
```

首先需要搞懂 `producerIndex`、`producerLimit` 以及 `consumerIndex` 之间的关系，这也是 MpscArrayQueue 中设计比较独特的地方。来看下 `lvProducerIndex()` 方法的源码，该方法继承自`MpscArrayQueueProducerLimitField`：

```
    // MpscArrayQueueProducerIndexField.java
    
    private volatile long producerIndex;
    
    public final long lvProducerIndex()
    {
        return producerIndex;
    }
```

初始化时，producerLimit 与队列的容量相等，producerIndex = consumerIndex = 0。假设 Thread1 和 Thread2 并发向 MpscArrayQueue 中存放数据，如下图所示：

![](/img/network-program/netty/MpscArrayQueue-offer.png)

1.  初始时，两个线程拿到的 `pIndex` 都等于 producerIndex 为 0，小于 producerLimit ；
2.  接着，两个线程都会尝试 CAS 更新 producerIndex + 1 ，必然只有一个线程能更新成功，另一个失败；
3.  假设 Thread1 CAS 操作成功，那么它拿到的 `pIndex` 为 0，Thread2 失败后就会重新更新 producerIndex ，然后更新成功，拿到 `pIndex` 为 1；
4.  最后，根据 `pIndex` 进行位运算，得到数组对应的下标，然后通过 `UNSAFE.putOrderedObject()` 方法将数据写入到数组中。

```
    // MpscArrayQueueProducerIndexField.java
    
    public static <E> void soElement(E[] buffer, long offset, E e) {
        UnsafeAccess.UNSAFE.putOrderedObject(buffer, offset, e);
    }
```

putOrderedObject() 和 putObject() 都可以用于更新对象的值，但是 putOrderedObject() 使用的是 LazySet 延迟更新机制，不会立刻更新数据到内存中，并且该方法会把其它 Cache Line 置为失效。所以， putOrderedObject() 要比 putObject() 性能高很多。

Java 中有四种类型的内存屏障，分别为 `LoadLoad`、`StoreStore`、`LoadStore` 和 `StoreLoad`。putOrderedObject() 使用了 StoreStore 内存屏障，对于 `Store1，StoreStore，Store2` 这样的操作序列，在 Store2 进行写入之前，会保证 Store1 的写操作对其他处理器可见。

> LazySet 机制是有代价的，就是写操作结果有纳秒级的延迟，不会立刻被其他线程以及自身线程可见。因为在 Mpsc Queue 的使用场景中，多个生产者只负责写入数据，并没有写入之后立刻读取的需求，所以使用 LazySet 机制是没有问题的，只要 StoreStore 内存屏障保证多线程写入的顺序即可。

因为生产者有多个线程，所以 MpscArrayQueue 采用了`UNSAFE.getLongVolatile()`方法保证获取消费者索引 consumerIndex 的准确性。getLongVolatile() 使用了 StoreLoad 内存屏障，对于 `Store1，StoreLoad，Load2` 的操作序列，在 Load2 以及后续的读取操作之前，都会保证 Store1 的写入操作对其他处理器可见。

#### poll()

poll 方法用于从队列出队一个元素，当队列为空时返回 null：

```
    // MpscArrayQueue.java
    
    public E poll()
    {
        // 获取消费者索引
        final long cIndex = lpConsumerIndex();
    
        // 计算数组对应的偏移量
        final long offset = calcCircularRefElementOffset(cIndex, mask);
    
        // 取出数组中 offset 对应的元素
        final E[] buffer = this.buffer;
        E e = lvRefElement(buffer, offset);
        if (null == e)
        {
            if (cIndex != lvProducerIndex())
            {
                // 等待生产者填充元素
                do
                {
                    e = lvRefElement(buffer, offset);
                }
                while (e == null);
            }
            else    // 队列为空
            {
                return null;
            }
        }
        // 消费成功后将当前位置置为 NULL
        spRefElement(buffer, offset, null);
        // 更新 consumerIndex 到下一个位置
        soConsumerIndex(cIndex + 1);
        return e;
    }
```

因为只有一个消费者线程，所以整个 poll() 的过程没有 CAS 操作。poll() 方法核心思路是获取消费者索引 consumerIndex，然后根据 consumerIndex 计算得出数组对应的偏移量，然后将数组对应位置的元素取出并返回，最后将 consumerIndex 移动到环形数组下一个位置。

获取消费者索引以及计算数组对应的偏移量的逻辑与 offer() 类似，在这里就不赘述了。下面直接看下如何取出数组中 offset 对应的元素，跟进 lvElement() 方法的源码：

```
    public static <E> E lvRefElement(E[] buffer, long offset)
    {
        return (E) UNSAFE.getObjectVolatile(buffer, offset);
    }
```

获取数组元素的时候同样使用了 UNSAFE 系列方法，`getObjectVolatile()` 方法则使用的是 `LoadLoad` 内存屏障，对于`Load1，LoadLoad，Load2` 操作序列，在 Load2 以及后续读取操作之前，会保证 Load1 的读取操作执行完毕，所以 getObjectVolatile() 方法可以保证每次读取数据都可以从内存中拿到最新值。

poll() 比较关注队列为空的情况。当调用 lvRefElement() 方法获取到的元素为 NULL 时，有两种可能的情况：队列为空或者生产者填充的元素还没有对消费者可见。如果消费者索引 consumerIndex 等于生产者 producerIndex，说明队列为空。只要两者不相等，消费者需要等待生产者填充数据完毕。

当成功消费数组中的元素之后，需要把当前消费者索引 consumerIndex 的位置置为 NULL，然后把 consumerIndex 移动到数组下一个位置。逻辑比较简单，下面我们把 spRefElement() 和 soConsumerIndex() 方法放在一起看：

```
    public static <E> void spRefElement(E[] buffer, long offset, E e)
    {
        UNSAFE.putObject(buffer, offset, e);
    }
    
    final void soConsumerIndex(long newValue)
    {
        UNSAFE.putOrderedLong(this, C_INDEX_OFFSET, newValue);
    }
```

最后的更新操作我们又看到了 UNSAFE put 系列方法的运用，其中 `putObject()`不会使用任何内存屏障，它会直接更新对象对应偏移量的值。而 putOrderedLong 与 putOrderedObject() 是一样的，都使用了 `StoreStore` 内存屏障，也是延迟更新 LazySet 机制，我就不再赘述了。

到此为止，MpscArrayQueue 入队和出队的核心源码已经分析完了。JCTools 还提供了 MpscUnboundedArrayQueue、MpscChunkedArrayQueue 等其他具有特色功能的队列，有兴趣的童鞋可以自行研究。

三、总结
----

本章，我对 Jctools 中的 Mpsc Queue 进行了剖析，重点分析了 MpscArrayQueue 的源码。最后，做一个总结：

*   JCTools 是服务于 JVM 的并发工具类，其中包含了很多技巧，例如填充法解决伪共享问题、Unsafe 直接操作内存等，以此提升性能；
*   MpscArrayQueue 一种多生产单消费者的队列，多个生产者线程通过 CAS 无锁操作提升性能，单个消费者不需要加锁；
*   MpscArrayQueue 内部的环形数组容量为 2 的次幂，可以通过位运算快速定位到数组对应下标。