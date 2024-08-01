---
layout:     post
title:      透彻理解 java 网络编程(十八)
subtitle:   HashedWheelTimer 时间轮
date:       2024-07-18 18:00:00
author:     yusihu
header-style: text
catalog: true
tags:
    - 网络编程
    - netty
    - 开源框架
---
本章，我将对 Netty 中的 **HashedWheelTimer** 这个延迟任务处理器进行讲解。我曾经在[《透彻理解 Kafka》](https://www.tpvlog.com/article/278)系列中介绍过 Kafka 的时间轮算法，为了实现高性能的定时任务调度，Netty 也引入了时间轮算法驱动定时任务的执行。

为什么 Netty 要用时间轮来处理定时任务？JDK 原生的实现方案不能满足要求吗？本章我将一步步深入剖析时间轮的原理以及 Netty 中是如何实现时间轮算法的。

一、JDK 定时器
---------

定时器一般有三种表现形式：按 **固定周期** 定时执行、 **延迟一定时间** 后执行、指定 **某个时刻** 执行。一般定时器都需要通过 **轮询** 的方式来实现，即每隔一个时间片去检查任务是否到期。

JDK 提供了三种常用的定时器，分别为 Timer、DelayedQueue 和 ScheduledThreadPoolExecutor，下面我来逐一介绍。

### 1.1 Timer

Timer 是 JDK 早期版本提供的一个定时器，用于固定周期任务以及延迟任务的执行。具体任务由 TimerTask 类定义，它是实现了 Runnable 接口的抽象类，而 Timer 负责调度和执行 TimerTask ：

```
    Timer timer = new Timer();
    timer.scheduleAtFixedRate(new TimerTask() {
        @Override
        public void run() {
            // do something
        }
    }, 10000, 1000);  // 10s后调度一个周期为1s的定时任务
```

看下 Timer 的内部构造：

```
    public class Timer {
        private final TaskQueue queue = new TaskQueue();
        private final TimerThread thread = new TimerThread(queue);
    
        public Timer(String name) {
            thread.setName(name);
            thread.start();
        }
    }
```

*   TaskQueue 是基于数组结构实现的小顶堆，deadline 最近的任务位于堆顶端，即`queue[1]`始终是最优先被执行的任务。由于是堆结构，所以 Run 操作时间复杂度 `O(1)`，新增 Schedule 和取消 Cancel 操作的时间复杂度都是 `O(logn)`；
    
*   Timer 内部启动了一个 TimerThread 线程，它会定时轮询 TaskQueue 中的任务，如果堆顶的任务 deadline 已到，那么执行任务；如果是周期性任务，则执行完成后再重新计算下一次任务的 deadline，然后再次放入堆；如果是单次任务，执行结束后会从 TaskQueue 中删除。
    
    Timer 目前并不推荐使用，它在设计上存在很多缺陷：
    
*   Timer 是单线程模式。如果某个 TimerTask 执行时间很久，会影响其他任务的调度；
    
*   Timer 的任务调度基于系统绝对时间，如果系统时间不正确，可能会出现问题；
    
*   TimerTask 如果执行出现异常，Timer 并不会捕获，会导致线程终止，其他任务永远不会执行。
    

### 1.2 DelayQueue

DelayQueue 是 JDK 中一种可以延迟获取对象的阻塞队列，其内部采用优先级队列 PriorityQueue 存储对象。DelayQueue 中的每个对象都必须实现 Delayed 接口，并重写 `compareTo` 和 `getDelay` 方法。

*   compareTo：对象根据 compareTo() 方法进行优先级排序；
*   getDelay：用于计算消息延迟的剩余时间，只有 `getDelay <=0`时，该对象才能从 DelayQueue 中取出。

DelayQueue 的使用方法如下：

```
    public class DelayQueueTest {
    
        public static void main(String[] args) throws Exception {
    
            BlockingQueue<SampleTask> delayQueue = new DelayQueue<>();
    
            long now = System.currentTimeMillis();
    
            delayQueue.put(new SampleTask(now + 1000));
            delayQueue.put(new SampleTask(now + 2000));
            delayQueue.put(new SampleTask(now + 3000));
    
            for (int i = 0; i < 3; i++) {
                System.out.println(new Date(delayQueue.take().getTime()));
            }
        }
    
        static class SampleTask implements Delayed {
            long time;
            public SampleTask(long time) {
                this.time = time;
            }
    
            public long getTime() {
                return time;
            }
    
            @Override
            public int compareTo(Delayed o) {
                return Long.compare(this.getDelay(TimeUnit.MILLISECONDS), o.getDelay(TimeUnit.MILLISECONDS));
            }
    
            @Override
            public long getDelay(TimeUnit unit) {
                // TimeUnit类的convert()方法，用于将给定单位的时间转换为该单位
                return unit.convert(time - System.currentTimeMillis(), TimeUnit.MILLISECONDS);
            }
        }
    }
```

DelayQueue 提供了 `put()`和 `take()` 阻塞方法，可以向队列中添加对象和取出对象，只有 getDelay 方法 <=0 时，对象才能从 DelayQueue 中取出。

DelayQueue 最常用的使用场景就是实现 **重试** 机制。例如，接口调用失败或者请求超时后，可以将请求对象放入 DelayQueue，然后通过一个异步线程 take() 取出对象后进行重试，如果还是请求失败，继续放回 DelayQueue。为了限制重试的频率，可以设置重试的最大次数以及采用 _**指数退避算法**_ 设置对象的 deadline，如 2s、4s、8s、16s …… 以此类推。

相比于 Timer，DelayQueue 只实现了任务管理的功能，需要与异步线程配合使用。DelayQueue 使用优先级队列实现任务的优先级排序，新增 Schedule 和取消 Cancel 操作的时间复杂度也是 `O(logn)`。

### 1.3 ScheduledThreadPoolExecutor

ScheduledThreadPoolExecutor 是 JDK 并发包中提供的一种可以周期或延迟调度任务的线程池。它继承于 ThreadPoolExecutor，并在 ThreadPoolExecutor 的基础上，重新设计了任务 ScheduledFutureTask 和阻塞队列 DelayedWorkQueue：

*   ScheduledFutureTask 继承于 FutureTask，重写了 run() 方法，使其具备周期执行任务的能力；
*   DelayedWorkQueue 内部是优先级队列，deadline 最近的任务在队列头部，对于周期执行的任务，在执行完会重新设置时间再次放入队列中。

ScheduledThreadPoolExecutor 的实现原理可以用下图表示。
![](/img/network-program/netty/ScheduledThreadPoolExecutor.png)

ScheduledThreadPoolExecutor 的使用可以参见下面的示例：

```
    public class ScheduledExecutorServiceTest {
        public static void main(String[] args) {
            ScheduledExecutorService executor = Executors.newScheduledThreadPool(5);
            // 1s延迟后开始执行任务，每2s重复执行一次
            executor.scheduleAtFixedRate(() -> System.out.println("Hello World"), 1000, 2000,
                                         TimeUnit.MILLISECONDS); 
        }
    }
```

JDK 中的三种定时器，实现思路都非常类似，都离不开 **任务** 、 **任务管理** 、 **任务调度** 三个角色。三种定时器的各类操作的时间复杂度如下：

*   查找待执行任务：O(1)；
*   新增任务：O(logn)；
*   取消任务：O(logn)。

因为内部都采用了堆结构，所以新增和删除任务的时间复杂度较高，在面对海量任务插入和删除的场景，会出现比较严重的性能瓶颈。因此，对于性能要求较高的场景，我们一般都会采用时间轮算法。

二、HashedWheelTimer
------------------

HashedWheelTimer 是 Netty 中时间轮算法的实现类。在分析 HashedWheelTimer 前，我们先来回顾时间轮算法是什么？它是如何解决海量任务插入和删除的？

### 2.1 算法思想

时间轮算法的设计思想就来源于钟表。它的基本特点如下：

1.  时间轮是一种环形结构，像钟表一样被分为多个 slot 槽位，每个 slot 代表一个时间段；
2.  每个 slot 中，使用链表保存该时间段到期的所有任务；
3.  时间轮通过一个时针随着时间一个个 slot 转动，并执行 slot 中的所有到期任务。

![](/img/network-program/netty/TimerWheel.png)

如上图所示，时间轮被划分为 8 个 slot，每个 slot 代表 1s，当前时针指向 2。我们通过一个示例来理解时间轮的执行过程：

1.  假如现在需要调度一个 3s 后执行的任务，应该加入 `2 + 3 = 5` 的 slot 中；
2.  假如现在需要调度一个 12s 后执行的任务，应该加入第 `( 2 + 12 ) % 8 = 6` 的 slot 中，此时时针完整走完一圈`round`加 4 个 slot。

这里有一个问题，对于上述的第二种情况，当时针走完 N 圈到达某个 slot 时，怎么区分这个 slot 内的任务哪些是当前需要立即执行，哪些是需要等待后面的 round 再执行呢？

所以，我们需要把 round 信息保存在任务中。例如，上图中第 6 个 slot 的链表中包含 3 个任务：第一个任务`round=0`，需要立即执行；第二个任务`round=1`，需要等待 8s 后执行；第三个任务`round=2`，需要等待`2*8=16s` 后执行。

当时针转动到对应 slot 时，只执行 round=0 的任务，slot 中其余任务的 round 应当减 1，等待下一个 round 之后执行。

上述就是时间轮算法的基本原理，该算法最大的优势是，任务的新增和取消都是 O(1) 时间复杂度，而且只需要一个线程就可以驱动时间轮进行工作。

### 2.2 接口定义

我们先来看与 HashedWheelTimer 相关的核心接口定义。

HashedWheelTimer 实现了接口`io.netty.util.Timer`，Timer 可以认为是上层的时间轮调度器，该接口提供了两个方法，分别是创建任务 `newTimeout()` 和停止所有未执行任务 `stop()`：

```
    // Timer.java
    
    public interface Timer {
    
        Timeout newTimeout(TimerTask task, long delay, TimeUnit unit);
    
        Set<Timeout> stop();
    }
```

通过`newTimeout()`方法可以提交一个 TimerTask 任务，并返回一个 Timeout。TimerTask 和 Timeout 是两个接口类：

```
    // TimerTask.java
    
    public interface TimerTask {
        void run(Timeout timeout) throws Exception;
    }
```

```
    // Timeout.java
    
    public interface Timeout {
        Timer timer();
    
        TimerTask task();
    
        boolean isExpired();
    
        boolean isCancelled();
    
        boolean cancel();
    }
```

Timeout 持有 Timer 和 TimerTask 的引用，而且通过 Timeout 接口可以取消任务。Timer、Timeout 和 TimerTask 之间的关系如下图所示：

![](/img/network-program/netty/Timer-Timeout-TimerTask.png)

### 2.3 使用示例

了解了 HashedWheelTimer 的接口定义以及相关组件的概念之后，我们通过一个示例来看看该如何使用 HashedWheelTimer：

```
    public class HashedWheelTimerTest {
    
        public static void main(String[] args) {
            Timer timer = new HashedWheelTimer();
    
            // 创建一个任务timeout1，10秒后执行
            Timeout timeout1 = timer.newTimeout(new TimerTask() {
                @Override
                public void run(Timeout timeout) {
                    System.out.println("timeout1: " + new Date());
                }
            }, 10, TimeUnit.SECONDS);
    
            // 取消任务timeout1
            if (!timeout1.isExpired()) {
                timeout1.cancel();
            }
    
            // 创建一个任务，1秒后执行
            timer.newTimeout(new TimerTask() {
                @Override
                public void run(Timeout timeout) throws InterruptedException {
                    System.out.println("timeout2: " + new Date());
                    Thread.sleep(5000);
                }
            }, 1, TimeUnit.SECONDS);
    
            // 创建一个任务，3秒后执行
            timer.newTimeout(new TimerTask() {
                @Override
                public void run(Timeout timeout) {
                    System.out.println("timeout3: " + new Date());
                }
            }, 3, TimeUnit.SECONDS);
        }
    }
```

执行结果如下：

```
    timeout2: Thu Aug 05 17:31:33 CST 2021
    timeout3: Thu Aug 05 17:31:38 CST 2021
```

上述示例中，我通过 newTimeout() 启动了三个 TimerTask：

*   timeout1 由于被取消了，所以并没有执行；
*   timeout2 和 timeout3 应该分别在 1s 和 3s 后执行，但是从打印时间看相差了 5s，因为 timeout2 阻塞了 5s 。

由此可以看出， **时间轮中的任务执行是串行的** ，当一个任务执行的时间过长，会影响后续任务的调度和执行，很可能产生任务堆积的情况。

### 2.4 源码分析

结合前面小节介绍的时间轮算法，我们来看下 HashedWheelTimer 的源码实现。

#### 构造函数

先来看 HashedWheelTimer 的构造函数：

```
    // HashedWheelTimer.java
    
    public HashedWheelTimer(
        ThreadFactory threadFactory, long tickDuration, TimeUnit unit, int ticksPerWheel, 
        boolean leakDetection, long maxPendingTimeouts) {
    
        // 创建时间轮的环形数组结构
        wheel = createWheel(ticksPerWheel);
        // 用于快速取模的掩码
        mask = wheel.length - 1;
        // 转换成纳秒
        long duration = unit.toNanos(tickDuration);
    
        //...
    
        // 创建工作线程
        workerThread = threadFactory.newThread(worker);
        // 是否开启内存泄漏检测
        leak = leakDetection || !workerThread.isDaemon() ? leakDetector.track(this) : null;
        // 最大允许等待任务数
        this.maxPendingTimeouts = maxPendingTimeouts;
        // 如果 HashedWheelTimer 的实例数超过 64，会打印错误日志
        if (INSTANCE_COUNTER.incrementAndGet() > INSTANCE_COUNT_LIMIT &&
            WARNED_TOO_MANY_INSTANCES.compareAndSet(false, true)) {
            reportTooManyInstances();
        }
    }
```

构造函数的几个入参是比较好理解的：

*   **threadFactory：** 线程池，用于创建一个与时间轮关联的 Woker 线程；
*   **tickDuration：** 时间轮的时针间隔；相当于时针间隔多久走到下一个 slot；
*   **unit：** tickDuration 的时间单位；
*   **ticksPerWheel：** 时间轮上一共有多少个 slot，默认 512 个（分配的 slot 越多，占用内存空间越大）；
*   **leakDetection：** 是否开启内存泄漏检测；
*   **maxPendingTimeouts：** 最大允许等待任务数。

#### 创建时间轮

来看下时间轮是如何创建的，其实就是创建了一个 HashedWheelBucket 数组，每个 HashedWheelBucket 表示时间轮中一个 slot：

```
    // HashedWheelTimer.java
    
    private static HashedWheelBucket[] createWheel(int ticksPerWheel) {
        checkInRange(ticksPerWheel, 1, 1073741824, "ticksPerWheel");
        // 数组长度整形成2的幂次
        ticksPerWheel = normalizeTicksPerWheel(ticksPerWheel);
        // 创建一个HashedWheelBucket数组
        HashedWheelBucket[] wheel = new HashedWheelBucket[ticksPerWheel];
        for (int i = 0; i < wheel.length; i ++) {
            wheel[i] = new HashedWheelBucket();
        }
        return wheel;
    }
    
    private static int normalizeTicksPerWheel(int ticksPerWheel) {
        int normalizedTicksPerWheel = 1;
        while (normalizedTicksPerWheel < ticksPerWheel) {
            normalizedTicksPerWheel <<= 1;
        }
        return normalizedTicksPerWheel;
    }

    private static final class HashedWheelBucket {
        private HashedWheelTimeout head;
        private HashedWheelTimeout tail;
        // 省略其他代码
    }
```

HashedWheelBucket 内部是一个双向链表结构，双向链表的每个节点是 HashedWheelTimeout 对象，HashedWheelTimeout 代表一个定时任务，每个 HashedWheelBucket 都包含双向链表 head 和 tail 两个 HashedWheelTimeout 节点，这样就可以实现不同方向进行链表遍历。

因为时间轮需要使用 & 做取模运算，所以数组的长度需要是 2 的次幂。normalizeTicksPerWheel() 方法的作用就是找到不小于 ticksPerWheel 的最小 2 次幂，这个方法实现的并不好，可以参考 JDK HashMap 扩容 tableSizeFor 的实现进行性能优化，如下所示。当然 normalizeTicksPerWheel() 只是在初始化的时候使用，所以并无影响。

```
    static final int MAXIMUM_CAPACITY = 1 << 30;
    private static int normalizeTicksPerWheel(int ticksPerWheel) {
        int n = ticksPerWheel - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
```

HashedWheelTimer 其内部结构与上文中介绍的时间轮算法类似
![](/img/network-program/netty/HashedWheelTimer.png)

#### 添加任务

HashedWheelTimer 初始化完成后，可以通过 `newTimeout()`方法添加任务，该方法主要做了三件事：

1.  启动内部的唯一工作线程；
2.  计算延时时间并创建定时任务；
3.  将任务添加到内部队列中。

HashedWheelTimer 的工作线程采用了懒启动的方式，不需要用户显示调用。这样做的好处是在时间轮中没有任务时，可以避免工作线程空转而造成性能损耗。

```
    // HashedWheelTimer.java
    
    public Timeout newTimeout(TimerTask task, long delay, TimeUnit unit) {
    
        long pendingTimeoutsCount = pendingTimeouts.incrementAndGet();
        if (maxPendingTimeouts > 0 && pendingTimeoutsCount > maxPendingTimeouts) {
            pendingTimeouts.decrementAndGet();
            throw new RejectedExecutionException("Number of pending timeouts ("
                                                 + pendingTimeoutsCount + ") is greater than or equal to maximum allowed pending "
                                                 + "timeouts (" + maxPendingTimeouts + ")");
        }
    
        // 1.如果工作线程没有启动，则启动它
        start();
    
        // 2.计算任务的deadline
        long deadline = System.nanoTime() + unit.toNanos(delay) - startTime;
        if (delay > 0 && deadline < 0) {
            deadline = Long.MAX_VALUE;
        }
    
        // 3.创建定时任务，添加到一个内部队列中
        HashedWheelTimeout timeout = new HashedWheelTimeout(this, task, deadline);
        timeouts.add(timeout);
        return timeout;
    }
```

先来看如何启动工作线程：

1.  首先，通过 CAS 操作获取工作线程的状态；
    
    *   如果已经启动：直接跳过；
    *   如果没有启动：通过 CAS 操作更改工作线程状态，然后启动工作线程；
2.  启动过程是直接调用`Thread.start()` 方法。
    

```
    // HashedWheelTimer.java
    
    public void start() {
        switch (WORKER_STATE_UPDATER.get(this)) {
            case WORKER_STATE_INIT:        // 未启动
                if (WORKER_STATE_UPDATER.compareAndSet(this, WORKER_STATE_INIT, WORKER_STATE_STARTED)) {
                    // 启动线程
                    workerThread.start();
                }
                break;
            case WORKER_STATE_STARTED:    // 已经启动
                break;
            case WORKER_STATE_SHUTDOWN:    // 已经关闭
                throw new IllegalStateException("cannot be started once stopped");
            default:
                throw new Error("Invalid WorkerState");
        }
    
        // Wait until the startTime is initialized by the worker.
        while (startTime == 0) {
            try {
                startTimeInitialized.await();
            } catch (InterruptedException ignore) {
                // Ignore - it will be ready very soon.
            }
        }
    }
```

再来看创建定时任务后，添加到内部队列这一步（这是一个 Mpsc Queue，我们暂且把它当作一个普通的并发阻塞队列，后续章节我会对它的底层原理进行剖析）。

这里思考下， **为什么不直接把 HashedWheelTimeout 任务添加到时间轮的 slot 中呢？**

因为 Mpsc Queue 可以理解为多生产者单消费者的线程安全队列，HashedWheelTimer 是想借助 Mpsc Queue 保证多线程向时间轮添加任务的线程安全性。

#### 工作线程

工作线程 Worker ，是时间轮的核心引擎，随着时针的转动，Worker 线程不断处理到期任务。下面我们直接看 Worker 的 run() 方法：

```
    // HashedWheelTimer.Worker.java
    
    private final class Worker implements Runnable {
        // 未处理任务列表
        private final Set<Timeout> unprocessedTimeouts = new HashSet<Timeout>();
    
        private long tick;
    
        @Override
        public void run() {
            // 初始化startTime，作为整个时间轮启动的基准时间
            startTime = System.nanoTime();
            if (startTime == 0) {
                startTime = 1;
            }
    
            startTimeInitialized.countDown();
    
            do {
                // 1.计算距下次tick的时间间隔, 然后sleep到下次tick
                final long deadline = waitForNextTick();
    
                // 可能因为溢出或者线程中断，造成 deadline <= 0
                if (deadline > 0) {
                    // 2.获取当前tick在HashedWheelBucket数组中对应的下标
                    int idx = (int) (tick & mask);
    
                    // 3.处理被取消的任务
                    processCancelledTasks();
    
                    // 4.从内部队列中取出任务加入对应的slot中
                    HashedWheelBucket bucket = wheel[idx];
                    transferTimeoutsToBuckets();
    
                    // 5.执行到期的任务
                    bucket.expireTimeouts(deadline);
    
                    tick++;
                }
            } while (WORKER_STATE_UPDATER.get(HashedWheelTimer.this) == WORKER_STATE_STARTED);
    
            // 时间轮退出后，取出slot中未执行且未被取消的任务，加入未处理任务列表，以便stop()方法返回
            for (HashedWheelBucket bucket: wheel) {
                bucket.clearTimeouts(unprocessedTimeouts);
            }
    
            // 将还没来得及添加到slot中的任务取出，如果任务未取消则加入未处理任务列表，以便stop()方法返回
            for (;;) {
                HashedWheelTimeout timeout = timeouts.poll();
                if (timeout == null) {
                    break;
                }
                if (!timeout.isCancelled()) {
                    unprocessedTimeouts.add(timeout);
                }
            }
            processCancelledTasks();
        }
    }
```

整个 run 方法的核心逻辑在一段 while 循环中执行，只要 Worker 是 STARTED 状态就会一直循环：

*   首先，通过 `waitForNextTick()`方法，计算出时针到下一次 tick 的时间间隔，然后 sleep 到下一次 tick；
*   通过位运算获取当前 tick 在 HashedWheelBucket 数组中对应的下标；
*   通过`processCancelledTasks`方法，处理被取消的任务；
*   通过`transferTimeoutsToBuckets`方法，从内部队列中取出任务加入对应的 HashedWheelBucket 中；
*   执行当前 HashedWheelBucket 中的到期任务。

##### waitForNextTick

首先看下 `waitForNextTick()` 方法是如何计算等待时间的，根据 tickDuration 可以推算出下一次 tick 的 deadline，deadline 减去当前时间就可以得到需要 sleep 的等待时间：

```
    // HashedWheelTimer.Worker.java
    
    private long waitForNextTick() {
        long deadline = tickDuration * (tick + 1);
    
        for (;;) {
            // 计算需要sleep的时间
            final long currentTime = System.nanoTime() - startTime;
            long sleepTimeMs = (deadline - currentTime + 999999) / 1000000;
    
            if (sleepTimeMs <= 0) {
                if (currentTime == Long.MIN_VALUE) {
                    return -Long.MAX_VALUE;
                } else {
                    return currentTime;
                }
            }
    
            if (PlatformDependent.isWindows()) {
                sleepTimeMs = sleepTimeMs / 10 * 10;
                if (sleepTimeMs == 0) {
                    sleepTimeMs = 1;
                }
            }
    
            try {
                Thread.sleep(sleepTimeMs);
            } catch (InterruptedException ignored) {
                if (WORKER_STATE_UPDATER.get(HashedWheelTimer.this) == WORKER_STATE_SHUTDOWN) {
                    return Long.MIN_VALUE;
                }
            }
        }
    }
```

所以 tickDuration 的值越小，时间的精准度也就越高，同时 Worker 的繁忙程度越高。

##### processCancelledTasks

再来看`processCancelledTasks`方法，用于处理被取消的任务。由于所有取消的任务都会加入`cancelledTimeouts` 队列中，所以该方法的逻辑很简单，就是会从队列中取出任务，将其从对应的 HashedWheelBucket 中移除：

```
    // HashedWheelTimer.Worker.java
    
    private void processCancelledTasks() {
        for (;;) {
            // 从队列中取出已取消的任务
            HashedWheelTimeout timeout = cancelledTimeouts.poll();
            if (timeout == null) {
                // all processed
                break;
            }
            try {
                // 移除任务
                timeout.remove();
            } catch (Throwable t) {
                if (logger.isWarnEnabled()) {
                    logger.warn("An exception was thrown while process a cancellation task", t);
                }
            }
        }
    }
```

##### transferTimeoutsToBuckets

再来看`transferTimeoutsToBuckets`方法，就是将内部队列中的任务添加到时间轮中的指定 slot，每次最多只处理 100000 个任务：

```
    // HashedWheelTimer.Worker.java
    
    private void transferTimeoutsToBuckets() {
        for (int i = 0; i < 100000; i++) {
            HashedWheelTimeout timeout = timeouts.poll();
            // 队列中没有任务
            if (timeout == null) {
                break;
            }
            // 已取消的任务忽略
            if (timeout.state() == HashedWheelTimeout.ST_CANCELLED) {
                continue;
            }
    
            // 计算任务需要经历多少个tick
            long calculated = timeout.deadline / tickDuration;
    
            // 计算任务需要在时间轮中经历的圈数remainingRounds
            timeout.remainingRounds = (calculated - tick) / wheel.length;
    
            // 取最大值，即如果任务在 timeouts 队列里已经过了执行时间, 那么会加入当前tick所在的HashedWheelBucket中
            final long ticks = Math.max(calculated, tick); 
            int stopIndex = (int) (ticks & mask);
    
            // 添加任务到slot的HashedWheelBucket中
            HashedWheelBucket bucket = wheel[stopIndex];
            bucket.addTimeout(timeout);
        }
    }
```

上述代码，根据任务的 deadline，可以计算出任务需要经过多少次 tick 才能开始执行，以及任务需要在时间轮中转动圈数 remainingRounds，remainingRounds 会记录在 HashedWheelTimeout 中，在执行任务时，remainingRounds 会被使用到。

因为时间轮中的任务并不能够保证及时执行，假如有一个任务执行的时间特别长，那么任务在 timeouts 队列里已经过了执行时间，也没有关系，Worker 会将这些任务直接加入当前的 HashedWheelBucket 中，所以过期的任务并不会被遗漏。

任务被添加到时间轮之后，重新再回到 Worker#run() 的主流程，接下来就是执行当前 HashedWheelBucket 中的到期任务，跟进 HashedWheelBucket#expireTimeouts() 方法的源码：

##### expireTimeouts

最后，我们来看 Workder 是如何执行当前 HashedWheelBucket 中的到期任务，跟进 `HashedWheelBucket.expireTimeouts()` 方法的源码：

```
    // HashedWheelTimer.HashedWheelBucket.java
    
    public void expireTimeouts(long deadline) {
        HashedWheelTimeout timeout = head;
    
        // 从链表的头节点开始遍历
        while (timeout != null) {
            HashedWheelTimeout next = timeout.next;
            if (timeout.remainingRounds <= 0) {
                next = remove(timeout);
    
                // 执行任务
                if (timeout.deadline <= deadline) {
                    timeout.expire();
                } else {
                    throw new IllegalStateException(String.format(
                        "timeout.deadline (%d) > deadline (%d)", timeout.deadline, deadline));
                }
            } else if (timeout.isCancelled()) {
                next = remove(timeout);
            } else {
                // 圈数-1
                timeout.remainingRounds --;
            }
            timeout = next;
        }
    }
```

执行任务的操作比较简单，就是从头开始遍历 HashedWheelBucket 中的双向链表：

*   如果 remainingRounds <=0，则调用 HashedWheelTimeout.expire() 方法执行任务；
*   如果任务已经被取消，直接从链表中移除；
*   否则，说明任务的执行时间还没到，remainingRounds 减 1，等待下一圈即可。

`HashedWheelTimeout.expire()` 内部就是调用了 TimerTask 的 run() 方法：

```
    // HashedWheelTimer.HashedWheelTimeout.java
    
    private final TimerTask task;
    public void expire() {
        if (!compareAndSetState(ST_INIT, ST_EXPIRED)) {
            return;
        }
    
        try {
            task.run(this);
        } catch (Throwable t) {
            if (logger.isWarnEnabled()) {
                logger.warn("An exception was thrown by " + TimerTask.class.getSimpleName() + '.', t);
            }
        }
    }
```

至此，工作线程 Worker 的核心逻辑就分析完了。当时间轮退出后，Worker 还会执行一些后置的收尾工作：  
Worker 会从每个 HashedWheelBucket 取出未执行且未取消的任务，以及还来得及添加到 HashedWheelBucket 中的任务，然后加入未处理任务列表，以便 stop() 方法统一处理。

#### 停止时间轮

Timer 接口的 newTimeout() 方法，上文已经分析完了，接下来分析`stop()` 方法，看下时间轮停止时做了哪些工作：

```
    // HashedWheelTimer.java
    
    public Set<Timeout> stop() {
        // 只能由非Worker线程停止时间轮
        if (Thread.currentThread() == workerThread) {
            throw new IllegalStateException(
                HashedWheelTimer.class.getSimpleName() + ".stop() cannot be called from " +
                TimerTask.class.getSimpleName());
        }
    
        // 尝试通过 CAS 操作将工作线程的状态更新为 SHUTDOWN 状态
        if (!WORKER_STATE_UPDATER.compareAndSet(this, WORKER_STATE_STARTED, WORKER_STATE_SHUTDOWN)) {
            // workerState can be 0 or 2 at this moment - let it always be 2.
            if (WORKER_STATE_UPDATER.getAndSet(this, WORKER_STATE_SHUTDOWN) != WORKER_STATE_SHUTDOWN) {
                INSTANCE_COUNTER.decrementAndGet();
                if (leak != null) {
                    boolean closed = leak.close(this);
                    assert closed;
                }
            }
    
            return Collections.emptySet();
        }
    
        try {
            boolean interrupted = false;
            while (workerThread.isAlive()) {
                // 中断 Worker 线程
                workerThread.interrupt();
                try {
                    workerThread.join(100);
                } catch (InterruptedException ignored) {
                    interrupted = true;
                }
            }
    
            if (interrupted) {
                Thread.currentThread().interrupt();
            }
        } finally {
            INSTANCE_COUNTER.decrementAndGet();
            if (leak != null) {
                boolean closed = leak.close(this);
                assert closed;
            }
        }
        // 返回未处理任务的列表
        return worker.unprocessedTimeouts();
    }
```

1.  如果当前线程是 Worker 线程，它是不能发起停止时间轮的操作的，是为了防止有定时任务发起停止时间轮的恶意操作；
    
2.  停止时间轮主要做了三件事：
    
    *   首先，尝试通过 CAS 操作将工作线程的状态更新为 SHUTDOWN 状态；
    *   然后，中断工作线程 Worker；
    *   最后，将未处理的任务列表返回。

到此为止，HashedWheelTimer 的核心源码就分析完了。再来回顾一下 HashedWheelTimer 的几个核心成员：

*   **HashedWheelTimeout** ：任务的封装类，包含任务的到期时间 deadline、需要经历的圈数 remainingRounds 等属性；
*   **HashedWheelBucket** ：相当于时间轮的每个 slot，内部采用双向链表保存了需要执行的 HashedWheelTimeout 列表；
*   **Worker** ：HashedWheelTimer 的核心工作引擎——内部的一个工作线程，负责处理定时任务。

### 2.5 优化思路

通过对 HashedWheelTimer 的源码分析，可以知道 Netty 中的时间轮是通过固定的时间间隔`tickDuration`进行推动的，如果长时间没有到期任务，那么会存在 **时间轮空推进** 的现象，从而造成一定的性能损耗。此外，如果任务的到期时间跨度很大，例如 A 任务 1s 后执行，B 任务 6 小时之后执行，也会造成空推进的问题。

所以，HashedWheelTimer 其实是有改进空间的。可以参考 Kafka 中时间轮的设计思想，Kafka 作为一个分布式消息中间件，面对海量高并发消息请求，对性能的要求更为苛刻，那么 Kafka 是如何解决上述时间跨度太大造成的时间轮空推进问题呢？

Kafka 时间轮的内部结构与 Netty 类似，如下图所示：

![](/img/network-program/netty/Kafka-HashedWheelTimer.png)

Kafka 的时间轮同样采用环形数组结构，数组中的每个 slot 代表一个 Bucket，每个 Bucket 保存了定时任务列表 TimerTaskList，TimerTaskList 同样采用双向链表的结构实现，链表的每个节点代表真正的定时任务 TimerTaskEntry。

#### DelayQueue

**空推进问题的本质是 Worker 线程不断循环执行，大量消耗 CPU** ，所以 Kafka 借助 JDK 的 DelayQueue 来负责推进时间轮：

1.  创建时间轮时，通过一个 DelayQueue 保存了时间轮中的每个 Bucket，由于 DelayQueue 底层是堆结构，最近到期时间的 Bucket 会在 DelayQueue 的队头；
2.  时间轮中会有一个线程负责出队 DelayQueue 中的 Bucket，如果时间没有到，线程会处于阻塞状态，从而解决空推进的问题。

这时候你可能会问，DelayQueue 插入和删除的性能不是并不好吗？没错，Bucket 在 DelayQueue 中的插入和删除性能确实是`O(logn)`，但是 Bucket 数量不多，且在时间轮初始化时，就已经构造好了，所以是可以解决任务本身的增删带来的性能问题的。

#### 层级时间轮

此外，为了解决任务时间跨度很大的问题，Kafka 引入了层级时间轮，当任务的 deadline 超出当前所在层的时间轮表示范围时，就会尝试将任务添加到上一层时间轮中，跟钟表的时针、分针、秒针的转动规则是同一个道理：

![](/img/network-program/netty/Kafka-HashedWheelTimer2.png)

从上图中可以看出：

*   第一层时间轮每个时间格为 1ms，整个时间轮的跨度为 20ms；
*   第二层时间轮每个时间格为 20ms，整个时间轮跨度为 400ms；
*   第三层时间轮每个时间格为 400ms，整个时间轮跨度为 8000ms。

在这种模型下，每一层时间轮都有自己的指针，每层时间轮走完一圈后，上层时间轮也会相应推进一格。以上图为例举个例子来理解，假设现在有一个任务，到期时间是 450ms 之后：

1.  首先，根据时间轮跨度，这个任务应该放到第三层时间轮的第一格；
2.  随着时间的流逝，当指针指向该时间格时，发现任务到期时间还有 50ms，这时需要进行时间轮降级，将任务重新提交到时间轮中；
3.  由于发现第一层时间轮整体跨度不够，所以放到第二层时间轮中的第三格；
4.  当时间再经历 40ms 之后，又会触发一次降级操作，将任务重新放入到第一层时间轮，最后等到 10ms 后执行任务。

由此可见，Kafka 的层级时间轮的时间粒度更好控制，可以应对更加复杂的定时任务处理场景，适用的范围更广。

三、总结
----

HashedWheelTimer 基于时间轮算法进行设计，设计思想值得我们借鉴。但是，HashedWheelTimer 并不是十全十美的，使用的时候需要清楚它存在的问题：

*   如果长时间没有到期任务，那么会存在时间轮空推进的现象。
*   由于 Worker 是单线程的，只适用于处理耗时较短的任务，如果一个任务执行的时间过长，会造成 Worker 线程阻塞。
*   相比传统定时器的实现方式，内存占用较大。

此时，我还介绍了 Kafka 中的时间轮算法，可以看到 Kafka 主要是针对时间轮的空推进问题进行了优化，引入了 DelayQueue 和多层级分层的手段，有效的提升了时间轮算法的性能。