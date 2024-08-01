---
layout:     post
title:      透彻理解 java 网络编程(八)
subtitle:   异步回调模式
date:       2024-07-08
author:     yusihu
catalog: true
tags:
    - 网络编程
---
异步回调并非 Java 网络编程这块的内容，事实上，我曾在[《透彻理解 Java 并发编程》](https://www.tpvlog.com/article/17)专栏中详细剖析过 Future 模式，Java 中的异步回调实际上就是基于 Future 模式。

考虑到后续章节，我会剖析 Netty 源代码，而 Netty 源码中大量使用了异步回调技术，并且基于 Java 的异步回调，设计了自己的一整套异步回调接口和实现。所以，本章我先从 Java Future 异步回调技术入手，然后介绍比较常用的第三方异步回调技术——谷歌公司的 Guava Future 相关技术，最后介绍一下 Netty 的异步回调技术，为后续 Netty 源码剖析作铺垫。

一、Future 模式
-----------

Java 在 1.5 版本之后提供了一种新的多线程的创建方式——FutureTask 方式。FutureTask 方式包含了一系列的 Java 相关类，在`java.util.concurrent`包中。其中最为重要的是 **FutureTask** 类和 **Callable** 接口。

### 1.1 Callable 接口

我们都知道，Runnable 接口是在 Java 多线程中表示线程的业务代码的抽象接口。但是，Runnable 有一个重要的问题：它的 run 方法是没有返回值的。正因为如此，Runnable 不能用于需要有返回值的应用场景。

为了解决 Runnable 接口的问题，Java 定义了一个新的和 Runnable 类似的接口——Callable 接口，并且将其中的代表业务处理的方法命名为  
call，call 方法有返回值：

```
    package java.util.concurrent;
    
    @FunctionalInterface
    public interface Callable<V> {
        // call方法有返回值
        V call() throws Exception;
    }
```

Callable 接口是一个泛型接口，也声明为了 “函数式接口”。其唯一的抽象方法 call 有返回值，返回值的类型为泛型形参的实际类型。call 抽象方法还有一个 Exception 的异常声明，容许方法内部的异常不经过捕获。

Callable 接口与 Runnable 接口相比，还有一个很大的不同： **Callable 接口的实例不能作为 Thread 线程实例的 target 来使用** ；而 Runnable 接口实例可以作为 Thread 线程实例的 target 构造参数，开启一个 Thread 线程。

那么问题来了，Java 中的线程类型，只有一个 Thread 类，没有其他的类型，那 Callable 实例该如何异步执行呢？事实上，Java 提供了 Callable 实例和 Thread 的 target 成员之间一个搭桥的类——FutureTask 类。

### 1.2 FutureTask 类

FutureTask 类代表一个未来执行的任务，也位于 java.util.concurrent 包中。FutureTask 类的构造函数的参数为 Callable 类型，实际上是对 Callable 类型的二次封装，可以执行 Callable 的 call 方法。FutureTask 类间接地继承了 Runnable 接口，从而可以作为 Thread 实例的 target 执行目标：

```
    public FutureTask(Callable<V> callable) {
        if (callable == null)
            throw new NullPointerException();
        this.callable = callable;
        this.state = NEW; // ensure visibility of callable
    }
```

总体来说，FutureTask 类首先是一个搭桥类的角色，FutureTask 类能当作 Thread 线程去执行目标 target，被异步执行；其次，如果要获取异步执行的结果，需要通过 FutureTask 类的方法去获取，在 FutureTask 类的内部，会将 Callable 的 call 方法的真正结果保存起来，以供外部获取。

### 1.3 Future 接口

在 Java 语言中，将 FutureTask 类的一系列操作，抽象出来作为一个重要的接口——Future 接口。当然，FutureTask 类也实现了此接口。Future 接口不复杂，主要是对并发任务的执行及获取其结果的一些操作：

```
    package java.util.concurrent;
    public interface Future<V> {
        boolean cancel(boolean mayInterruptRunning);
        boolean isCancelled();
        boolean isDone();
        V get() throws InterruptedException，ExecutionException;
        V get(long timeout，TimeUnitunit) throws InterruptedException，ExecutionException，TimeoutException;
    }
```

Future 主要提供了 3 大功能：

1.  判断并发任务是否执行完成；
2.  获取并发任务执行的结果；
3.  取消并发执行中的任务。

关于 Future 接口的方法，详细说明如下：

*   **V get()** ：获取并发任务执行的结果。这个方法是阻塞性的，如果并发任务没有执行完成，调用此方法的线程会一直阻塞，直到并发任务执行完成；
*   **V get(Long timeout，TimeUnit unit)** ：获取并发任务执行的结果。这个方法也是阻塞性的，但是会有阻塞时间限制，如果阻塞时间超过设定的 timeout 时间，该方法将抛出异常；
*   **boolean isDone()：** 获取并发任务的执行状态。如果任务执行结束，则返回 true；
*   **boolean isCancelled()** ：获取并发任务的取消状态。如果任务完成前被取消，则返回 true；
*   **boolean cancel(boolean mayInterruptRunning)** ：取消并发任务的执行。

### 1.4 使用示例

我们来看一个使用 FutureTask 完成异步回调的示例。定义两个异步任务，主线程通过 FutureTask 获取异步任务的执行结果：

```
    /**
     * 烧水任务
     */
    class HotWarterJob implements Callable<Boolean>
    {
        @Override
        public Boolean call() throws Exception
        {
            try {
                Logger.info("洗好水壶");
                Logger.info("灌上凉水");
                Logger.info("放在火上");
                //线程睡眠一段时间，代表烧水中
                Thread.sleep(500);
                Logger.info("水开了");
            } catch (InterruptedException e) {
                Logger.info(" 发生异常被中断.");
                return false;
            }
            Logger.info(" 运行结束.");
            return true;
        }
    }
```

```
    /**
     * 洗茶具任务类
     */
    class WashJob implements Callable<Boolean> {
        @Override
        public Boolean call() throws Exception {
            try {
                Logger.info("洗茶壶");
                Logger.info("洗茶杯");
                Logger.info("拿茶叶");
                //线程睡眠一段时间，代表清洗中
                Thread.sleep(500);
                Logger.info("洗完了");
            } catch (InterruptedException e) {
                Logger.info(" 清洗工作发生异常被中断.");
                return false;
            }
            Logger.info(" 清洗工作运行结束.");
            return true;
        }
    }
```

```
    /**
     * 喝茶类
     */
    public class JavaFutureDemo {
        public static void drinkTea(boolean warterOk, boolean cupOk) {
            if (warterOk && cupOk) {
                Logger.info("泡茶喝");
            } else if (!warterOk) {
                Logger.info("烧水失败，没有茶喝了");
            } else if (!cupOk) {
                Logger.info("杯子洗不了，没有茶喝了");
            }
        }
    
        public static void main(String args[]) {
            // 1.创建烧水异步任务
            Callable<Boolean> hJob = new HotWarterJob();
            FutureTask<Boolean> hTask = new FutureTask<>(hJob);
            Thread hThread = new Thread(hTask, "** 烧水-Thread");
    
            // 2.创建洗茶具异步任务
            Callable<Boolean> wJob = new WashJob();
            FutureTask<Boolean> wTask = new FutureTask<>(wJob);
            Thread wThread = new Thread(wTask, "$$ 清洗-Thread");
    
            // 3.启动异步任务
            hThread.start();
            wThread.start();
    
            // 4.获取任务执行结果
            try {
                // 通过FutureTask类的get方法，获取异步结果时，主线程会被阻塞
                boolean warterOk = hTask.get();
                boolean cupOk = wTask.get();
                drinkTea(warterOk, cupOk);
            } catch (InterruptedException e) {
                Logger.info("发生异常被中断.");
            } catch (ExecutionException e) {
                e.printStackTrace();
            }
        }
    }
```

二、Guava 回调模式
------------

上述使用 Future 模式的过程中，主线程在获取异步任务的执行结果时，如果调用`Future.get()`方法，会被阻塞住，所以 Future 模式本质还是一种异步阻塞模式。被阻塞的主线程不能干任何事情，唯一能干的，就是在傻傻地等待。原生 Java API，除了阻塞的获取结果外，并没有实现非阻塞的异步结果获取方法。如果需要用到获取异步的结果，则需要引入一些额外的框架，我这里介绍 Google Guava 这个常用框架。

### 2.1 Google Guava

何为 Guava？它是谷歌公司提供的 Java 扩展包，提供了一种异步回调的解决方案。相关的源代码在`com.google.common.util.concurrent`包中。Guava 中的很多类，都是对 java.util.concurrent 能力的扩展和增强。例如，Guava 的异步任务接口`ListenableFuture`扩展了 Java 的 Future 接口，实现了 **非阻塞获取异步结果** 的功能。

Guava 对 Java 的异步回调机制，做了以下的增强：

1.  引入了一个新的接口`ListenableFuture`（继承自 Java 的 Future 接口），使得能以异步非阻塞的方式获取任务执行结果；
2.  引入了一个新的接口`FutureCallback`（这是一个独立接口），该接口可以在异步任务执行完成后，根据任务结果完成不同的回调处理。

### 2.2 FutureCallback

FutureCallback 是一个新增的接口，用来实现异步任务执行完后的回调逻辑：

```
    package com.google.common.util.concurrent;
    public interface FutureCallback<V> {
        void onSuccess(@Nullable V var1);
        void onFailure(Throwable var1);
    }
```

可以看到，FutureCallback 拥有两个回调方法：

*   **onSuccess：** 在异步任务执行成功后被回调。调用时，异步任务的执行结果作为 onSuccess 方法的参数被传入；
*   **onFailure：** 在异步任务执行抛出异常时被回调。调用时，异步任务所抛出的异常作为 onFailure 方法的参数被传入。

### 2.3 ListenableFuture

Guava 的 ListenableFuture 接口是对 Java 的 Future 接口的扩展，可以理解为异步任务的实例：

```
    package com.google.common.util.concurrent;
    import java.util.concurrent.Executor;
    import java.util.concurrent.Future;
    
    public interface ListenableFuture<V> extends Future<V> {
    // 此方法由Guava内部调用
    void addListener(Runnable r, Executor e);
    }
```

可以看到，ListenableFuture 仅仅增加了一个方法——addListener 方法。它的作用就是将 FutureCallback 封装成一个内部的 Runnable 异步回调任务，在 Callable 异步任务完成后，回调 FutureCallback 进行善后处理。

> 注意，ListenableFuture 的 addListener 方法只在 Guava 内部调用，如果对它感兴趣，可以查看 Guava 源代码。在实际编程中，我们不会调用 addListener。

在实际编程中，我们可以使用 Guava 的 **Futures** 工具类，将 FutureCallback 回调逻辑绑定到异步的 ListenableFuture 任务上：

```
    Futures.addCallback(listenableFuture, new FutureCallback<Boolean>() {
        public void onSuccess(Boolean r) {
            // listenableFuture内部的Callable 成功时回调此方法
        }
        public void onFailure(Throwable t) {
            // listenableFuture内部的Callable异常时回调此方法
        }
    });
```

### 2.4 Guava 线程池

还剩下一个问题，我们如何创建 ListenableFuture 实例呢？事实上，Google Guava 提供了自己的线程池，往里面提交 Callable 任务，就会返回 ListenableFuture 实例。Guava 线程池，本质是对 Java 线程池的一种装饰，创建 Guava 线程池的方法如下：

```
    // 1.首先创建Java线程池
    ExecutorService jPool= Executors.newFixedThreadPool(10);
    // 2.构造一个Guava线程池
    ListeningExecutorService gPool= MoreExecutors.listeningDecorator(jPool);
```

有了 Guava 的线程池之后，就可以通过 submit 方法来提交任务了。任务提交之后的返回结果，就是我们所要的 ListenableFuture 异步任务实例了：

```
    // 调用submit方法来提交任务，返回异步任务实例
    ListenableFuture<Boolean> hFuture = gPool.submit(hJob);
```

### 2.5 使用示例

最后，我们通过 Google Guava 来完成第一节中的异步任务示例：

```
    public class GuavaFutureDemo {
    
        public static void main(String args[]) {
            // 创建一个泡茶主线程
            MainJob mainJob = new MainJob();
            Thread mainThread = new Thread(mainJob);
            mainThread.setName("主线程");
            mainThread.start();
    
            // 烧水的业务逻辑实例
            Callable<Boolean> hotJob = new HotWarterJob();
            Callable<Boolean> washJob = new WashJob();
            ExecutorService jPool = Executors.newFixedThreadPool(10);
            ListeningExecutorService gPool = MoreExecutors.listeningDecorator(jPool);
    
            // 烧水异步任务
            ListenableFuture<Boolean> hotFuture = gPool.submit(hotJob);
            Futures.addCallback(hotFuture, new FutureCallback<Boolean>() {
                public void onSuccess(Boolean r) {
                    if (r) {
                        mainJob.warterOk = true;
                    }
                }
                public void onFailure(Throwable t) {
                    Logger.info("烧水失败，没有茶喝了");
                }
            });
    
            // 洗茶具异步任务
            ListenableFuture<Boolean> washFuture = gPool.submit(washJob);
            Futures.addCallback(washFuture, new FutureCallback<Boolean>() {
                public void onSuccess(Boolean r) {
                    if (r) {
                        mainJob.cupOk = true;
                    }
                }
                public void onFailure(Throwable t) {
                    Logger.info("杯子洗不了，没有茶喝了");
                }
            });
        }
    
        // 泡茶喝主线程类
        static class MainJob implements Runnable {
            boolean warterOk = false;
            boolean cupOk = false;
            @Override
            public void run() {
                while (true) {
                    try {
                        Thread.sleep(50);
                        Logger.info("读书中......");
                    } catch (InterruptedException e) {
                        Logger.info(getCurThreadName() + "发生异常被中断.");
                    }
                    if (warterOk && cupOk) {
                        drinkTea(warterOk, cupOk);
                    }
                }
            }
    
            public void drinkTea(Boolean wOk, Boolean cOK) {
                if (wOk && cOK) {
                    Logger.info("泡茶喝，茶喝完");
                } else if (!wOk) {
                    Logger.info("烧水失败，没有茶喝了");
                } else if (!cOK) {
                    Logger.info("杯子洗不了，没有茶喝了");
                }
            }
        }
    }
```

通过上述的示例，可以看到，Guava 异步回调和 Java 的 FutureTask 异步回调，本质的不同在于：

*   Guava 是 **非阻塞** 的异步回调，调用线程是不阻塞的，可以继续执行自己的业务逻辑；
*   FutureTask 是 **阻塞** 的异步回调，调用线程是阻塞的，在获取异步结果的过程中，一直阻塞，等待异步线程返回结果。

三、Netty 回调模式
------------

Netty 官方文档中指出 Netty 的网络操作都是异步的。在 Netty 源代码中，大量使用了异步回调处理模式。在 Netty 的业务开发层面，Netty 应用的 Handler 处理器中的业务处理代码，也都是异步执行的。所以，了解 Netty 的异步回调，无论是 Netty 应用级的开发还是源代码级的开发，都是十分重要的。

Netty 和 Guava 一样，实现了自己的异步回调体系：Netty 继承和扩展了 JDK Future 的 API，定义了自身的 Future 系列接口和类，实现了异步任务的监控、异步执行结果的获取。总体来说，Netty 对 Java Future 异步任务的扩展如下：

1.  继承 Java 的 Future 接口，得到了一个新的属于 Netty 自己的 Future 异步任务接口。该接口对原有的接口进行了增强，使得 Netty 异步任务能够以非阻塞的方式处理回调的结果；
2.  引入了一个新接口——GenericFutureListener，表示异步执行完成的监听器。这个接口和 Guava 的 FutureCallbak 回调接口不同。Netty 使用了监听器模式，将异步任务执行完成后的回调逻辑抽象成了 Listener 监听器接口；
3.  Netty 的 GenericFutureListener 监听器接口实现类，可以加入 Netty 异步任务 Future 中，实现对异步任务执行状态的事件监听。

总体上说，在异步非阻塞回调的设计思路上，Netty 和 Guava 的思路是一致的。对应关系为：

*   Netty 的 Future 接口，可以对应到 Guava 的 ListenableFuture 接口；
*   Netty 的 GenericFutureListener 接口，可以对应到 Guava 的 FutureCallback 接口。

### 3.1 GenericFutureListener

GenericFutureListener 接口，用来封装异步回调的逻辑：

```
    package io.netty.util.concurrent;
    import java.util.EventListener;
    
    public interface GenericFutureListener<F extends Future<?>> extends EventListener {
        //监听器的回调方法
        void operationComplete(F var1) throws Exception;
    }
```

在大多数情况下，Netty 的异步回调的代码编写在`GenericFutureListener.operationComplete()`方法中，`operationComplete`方法会在 Future 异步任务执行完成后回调。

### 3.2 Future 接口

Netty 自定义了自己的 Future 接口，位于`io.netty.util.concurrent`包中：

```
    public interface Future<V> extendsjava.util.concurrent.Future<V> {
        // 判断是否执行成功
        boolean isSuccess();
        // 判断是否取消
        boolean isCancellable(); 
        // 获取异步任务异常的原因
        Throwable cause();
        // 增加监听器Listener
        Future<V> addListener(GenericFutureListener<? extends Future<? super V>> listener);
        // 移除监听器Listener
        Future<V> removeListener(GenericFutureListener<? extends Future<? super V>> listener);
    }
```

实际使用时，一般使用 Future 的子接口，代表了不同类型的异步任务，比如常用的`ChannelFuture`接口。

### 3.3 ChannelFuture

在 Netty 的网络编程中，网络连接通道的输入和输出处理都是异步进行的，都会返回一个 ChannelFuture 接口的实例。通过返回的 Future 实例，可以为它增加异步回调的监听器，在异步任务真正完成后，执行回调：

```
    // connect是异步的，仅提交异步任务
    ChannelFuture future = bootstrap.connect(new InetSocketAddress("www.manning.com",80));
    // 添加监听器
    future.addListener(new ChannelFutureListener() {
        @Override
        public void operationComplete(ChannelFuture channelFuture) throws Exception {
            if(channelFuture.isSuccess()){
                System.out.println("Connection established");
            } else {
                System.err.println("Connection attempt failed");
                channelFuture.cause().printStackTrace();
            }
        }
    });
```

> 上述的`ChannelFutureListener`是 GenericFutureListener 的子接口。

### 3.4 使用实例

Netty 的出站和入站操作都是异步的。异步回调的方法，和上面 Netty 建立连接的异步回调是一样的。以最为经典的 NIO 出站操作——write 为例，说明一下 ChannelFuture 的使用。

在调用 write 操作后，Netty 并没有完成对 Java NIO 底层连接的写入操作，因为是异步执行的：

```
    // write方法，返回一个异步任务
    ChannelFuture future = ctx.channel().write(msg);
    // 为异步任务加上监听器
    future.addListener(new ChannelFutureListener() {
        @Override
        public void operationComplete(ChannelFuture future) {
            // write操作完成后的回调代码
        }
    });
```

四、总结
----

本章，我对 Java 编程中的异步回调模式进行了讲解，主要对 Java 原生异步模式，Guaua 异步模式，Netty 异步模式进行了介绍和比较，读者需要根据实际的业务场景来选择合适的模式。