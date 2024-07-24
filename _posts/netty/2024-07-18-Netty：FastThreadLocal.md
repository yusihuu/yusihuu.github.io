---
layout:     post
title:      透彻理解 java 网络编程(十七)
subtitle:   netty 原理：FastThreadLocal 线程本地变量
date:       2024-07-18 12:00:00
author:     yusihu
header-style: text
catalog: true
tags:
    - 网络编程
    - netty
    - 开源框架
---

本章，我将对 Netty 中的 FastThreadLocal 这个线程本地工具类进行讲解。我曾介绍过 JDK 中的 ThreadLocal，Netty 官方表示 FastThreadLocal 是比 JDK 的 ThreadLocal 性能更高。

那么，FastThreadLocal 到底比 ThreadLocal 快在哪里呢？

一、ThreadLocal
-------------

我先来带大家回顾下 JDK 中的 ThreadLocal。ThreadLocal 可以理解为线程本地变量，它是 Java 并发编程中非常重要的一个工具类。ThreadLocal 为变量在每个线程中都创建了一个副本，该副本只能被当前线程访问，多线程之间是隔离的，变量不能在多线程之间共享。这样每个线程修改变量副本时，不会对其他线程产生影响。

### 1.1 使用示例

通过一个例子看下 ThreadLocal 如何使用：

```
    public class ThreadLocalTest {
        private static final ThreadLocal<String> THREAD_NAME_LOCAL = new ThreadLocal<>();
        private static final ThreadLocal<TradeOrder> TRADE_THREAD_LOCAL = new ThreadLocal<>();
    
        public static void main(String[] args) {
            for (int i = 0; i < 2; i++) {
                int tradeId = i;
                new Thread(new Runnable() {
                    @Override
                    public void run() {
                        TradeOrder tradeOrder = new TradeOrder(tradeId, tradeId % 2 == 0 ? "已支付" : "未支付");
                        TRADE_THREAD_LOCAL.set(tradeOrder);
                        THREAD_NAME_LOCAL.set(Thread.currentThread().getName());
                        System.out.println("threadName: " + THREAD_NAME_LOCAL.get());
                        System.out.println("tradeOrder info：" + TRADE_THREAD_LOCAL.get());
                    }
                }).start();
            }
        }
    
        static class TradeOrder {
            long id;
            String status;
    
            public TradeOrder(int id, String status) {
                this.id = id;
                this.status = status;
            }
    
            @Override
            public String toString() {
                return "id=" + id + ", status=" + status;
            }
        }
    }
```

上述示例中，我构造了`THREAD_NAME_LOCAL`和`TRADE_THREAD_LOCAL`两个 ThreadLocal 变量，分别用于记录当前线程名称和订单交易信息。

输出结果如下，可以看出 Thread-0 和 Thread-1 虽然操作的是相同的 ThreadLocal 对象，但是它们取到了不同的线程名称和订单交易信息：

```
    threadName: Thread-0
    tradeOrder info：id=0, status=已支付
    threadName: Thread-1
    tradeOrder info：id=1, status=未支付
```

### 1.2 数据结构

既然多线程访问 ThreadLocal 变量时都会有自己独立的实例副本，那么很容易想到的方案就是在 ThreadLocal 中维护一个 Map，记录线程与实例之间的映射关系。当新增线程和销毁线程时都需要更新 Map 中的映射关系，因为会存在多线程并发修改，所以需要保证 Map 是线程安全的。

那么 JDK 的 ThreadLocal 是这么实现的吗？答案是 NO。因为在高并发的场景并发修改 Map 需要加锁，势必会降低性能。JDK 为了避免加锁，采用了相反的设计思路： **以 Thread 入手，在每个 Thread 中维护 Map，记录 ThreadLocal 与本地变量之间的映射关系** ，这样在同一个线程内，Map 就不需要加锁了。

每个线程内部有一个 **ThreadLocalMap** ，它是一种使用线性探测法实现的哈希表，底层采用数组存储数据。数组（默认大小 16）的每一项是一个`Entry` 对象，用于保存 key-value 键值对，key 就是 ThreadLocal 对象，value 就是线程本地变量：

![](/img/network-program/netty/ThreadLocal.png)

当线程调用`ThreadLocal.set(XXX)`添加对象时，会执行以下操作：

1.  首先，根据 ThreadLocal 对象的`threadLocalHashCode`与内部数组长度进行取余，计算应该存放到哪个索引位置的 Entry 中；
2.  如果出现 Hash 冲突，也就是说这个索引位置已经其它 ThreadLocal 对象占用了，则依次向后查找第一个空的索引位置；
3.  最后，更新 Entry 中的变量值。

由此可见，`ThreadLocal.set()/get()`方法在数据密集时很容易出现 Hash 冲突，需要`O(n)`时间复杂度解决冲突问题，效率较低。

> 每个 ThreadLocal 在初始化时都会有一个 Hash 值`threadLocalHashCode`，每增加一个 ThreadLocal， Hash 值就会固定增加一个魔数`HASH_INCREMENT = 0x61c88647`。为什么取`0x61c88647` 这个魔数呢？因为实验证明，通过 0x61c88647 累加生成的`threadLocalHashCod` 与 2 的幂取模，得到的结果可以较为均匀地分布在长度为 2 的幂次的数组中。

下面我们再聊聊 ThreadLocalMap 中 Entry 的设计原理。

Entry 继承自弱引用类`WeakReference`，Entry 的 key 是弱引用，value 是强引用。在 JVM 垃圾回收时，只要发现了弱引用的对象，不管内存是否充足，都会进行回收。

那么 **为什么 Entry 的 key 要设计成弱引用呢？** 我们试想下，如果 key 都是强引用，当 ThreadLocal 不再使用时，然而 ThreadLocalMap 中还是存在对 ThreadLocal 的强引用，那么 GC 是无法回收的，从而造成内存泄漏。

虽然 Entry 的 key 设计成了弱引用，但是当 ThreadLocal 不再使用被 GC 回收后，ThreadLocalMap 中可能出现 Entry 的 key 为 NULL，而 Entry 的 value 则一直会强引用数据而得不到释放，只能等待线程销毁，最终造成内存泄漏。

那么应该 **如何避免 ThreadLocalMap 内存泄漏呢？**

1.  首先，ThreadLocal 已经帮助我们做了一定的保护措施，在执行`ThreadLocal.set()/get()`方法时，ThreadLocal 会清除 ThreadLocalMap 中 key 为 NULL 的 Entry 对象；
2.  对开发者而言，需要保持良好的编码意识：当线程中某个 ThreadLocal 对象不需要使用时，应立即调用 remove() 方法删除 Entry 对象。如果是在异常的场景中，应该在 finally 代码块中进行清理。

二、FastThreadLocal
-----------------

JDK 的 ThreadLocal 已经在实际开发中运用的非常成熟了，Netty 为什么还要自己实现一个 FastThreadLocal ？我们来一探究竟。

FastThreadLocal 的实现与 ThreadLocal 非常类似，Netty 为 FastThreadLocal 量身打造了 `FastThreadLocalThread` 和 `InternalThreadLocalMap` 两个重要的类。

FastThreadLocalThread 是对 Thread 类的一层包装，每个线程都有一个 InternalThreadLocalMap 实例。所以，只有 FastThreadLocal 和 FastThreadLocalThread 组合使用，才能发挥 FastThreadLocal 的性能优势。对应关系如下：

*   ThreadLocal <-> FastThreadLocal
*   Thread <-> FastThreadLocalThread
*   ThreadLocalMap <-> InternalThreadLocalMap

```
    // FastThreadLocalThread.java
    
    public class FastThreadLocalThread extends Thread {
    
        private InternalThreadLocalMap threadLocalMap;
    
        //...
    }
```

### 2.1 使用示例

我们先通过一个例子来看如何使用 FastThreadLocal 。可以看到，用法几乎和 JDK 中的 ThreadLocal 完全一样，只需要把代码中 Thread、ThreadLocal 分别替换为 FastThreadLocalThread 和 FastThreadLocal 即可：

```
    public class FastThreadLocalTest {
        private static final FastThreadLocal<String> THREAD_NAME_LOCAL = new FastThreadLocal<>();
        private static final FastThreadLocal<ThreadLocalTest.TradeOrder> TRADE_THREAD_LOCAL = new FastThreadLocal<>();
    
        public static void main(String[] args) {
            for (int i = 0; i < 2; i++) {
                int tradeId = i;
                new FastThreadLocalThread(new Runnable() {
                    @Override
                    public void run() {
                        ThreadLocalTest.TradeOrder tradeOrder = new ThreadLocalTest.TradeOrder(tradeId, tradeId % 2 == 0 ? "已支付" : "未支付");
                        TRADE_THREAD_LOCAL.set(tradeOrder);
                        THREAD_NAME_LOCAL.set(Thread.currentThread().getName());
                        System.out.println("threadName: " + THREAD_NAME_LOCAL.get());
                        System.out.println("tradeOrder info：" + TRADE_THREAD_LOCAL.get());
                    }
                }).start();
            }
        }
    
        static class TradeOrder {
            long id;
            String status;
    
            public TradeOrder(int id, String status) {
                this.id = id;
                this.status = status;
            }
    
            @Override
            public String toString() {
                return "id=" + id + ", status=" + status;
            }
        }
    }
```

输出结果如下：

```
    threadName: Thread-2
    threadName: Thread-1
    tradeOrder info：id=0, status=已支付
    tradeOrder info：id=1, status=未支付
```

### 2.2 性能优势

我之前讲到 ThreadLocal 的一个缺点：内部的 ThreadLocalMap 采用线性探测法解决 Hash 冲突，性能较差。我们看看 Netty 中的 InternalThreadLocalMap 是如何优化的：

1.  每个 FastThreadLocal 对象，在初始化时会分配一个数组索引`index`，索引值由 InternalThreadLocalMap 类采用 AtomicInteger 生成，递增且全局唯一；
2.  当线程（FastThreadLocalThread）通过 FastThreadLocal 读写数据时，通过索引下标，可以定位到 FastThreadLocal 在 InternalThreadLocalMap 中的位置，时间复杂度为 `O(1)`；

```
    // FastThreadLocal.java
    
    private final int index;
    
    public FastThreadLocal() {
        index = InternalThreadLocalMap.nextVariableIndex();
    }
```

```
    // InternalThreadLocalMap.java
    
    public final class InternalThreadLocalMap extends UnpaddedInternalThreadLocalMap {
    
        private static final AtomicInteger nextIndex = new AtomicInteger();
        private Object[] indexedVariables;
    
        public static int nextVariableIndex() {
            int index = nextIndex.getAndIncrement();
            if (index < 0) {
                nextIndex.decrementAndGet();
                throw new IllegalStateException("too many thread-local indexed variables");
            }
            return index;
        }
        //...
    }
```

上述这种方案可以有效解决 Hash 冲突问题，当然，如果 InternalThreadLocalMap 内部的数组下标递增到非常大，那么数组也会比较大，所以，FastThreadLocal 是通过以 **空间换时间** 的思想提升读写性能。

### 2.3 数据结构

我们继续来看 `InternalThreadLocalMap` 和 `FastThreadLocal` 的数据结构：

![](/img/network-program/netty/InternalThreadLocalMap.png)

InternalThreadLocalMap 使用 Object 数组替代了 Entry 数组，Object[0] 存储的是一个 Set > 集合，从数组下标 1 开始都是直接存储的 value 数据，不再采用 ThreadLocal 的键值对形式进行存储，数组默认大小 32，默认每个元素的值都是 `UNSET`这个缺省 Object 对象的引用。

> 最新版本的 Netty 中，不再是在索引 0 处存放 Set 集合，而是在`variablesToRemoveIndex`处，我这里为了便于后续讲解，沿用 0。

举个例子，假设现在我们有一批数据需要添加到数组中，分别为 value1、value2、value3、value4，对应的 FastThreadLocal 在初始化的时候生成的数组索引分别为 1、2、3、4。如下图所示：

![](/img/network-program/netty/InternalThreadLocalMap1.png)

### 2.4 源码分析

本节，我来分析 FastThreadLocal 的源码，帮助大家从底层理解 FastThreadLocal 的原理。

#### set 方法

先来看设置线程本地变量的`set`方法：

```
    // FastThreadLocal.java
    
    public final void set(V value) {
        // 如果value不是缺省值
        if (value != InternalThreadLocalMap.UNSET) {
            // 获取当前线程内部的InternalThreadLocalMap
            InternalThreadLocalMap threadLocalMap = InternalThreadLocalMap.get();
            // 将InternalThreadLocalMap中的数据替换为新的value
            setKnownNotUnset(threadLocalMap, value);
        } else {
            remove();
        }
    }
```

上述 set() 的过程主要分为三步：

1.  首先，判断 value 是否为缺省值，如果是则调用 remove() 直接清除（这个后面专门分析）；
2.  如果 value 不等于缺省值，则获取当前线程的 InternalThreadLocalMap；
3.  最后，将 InternalThreadLocalMap 中对应数据替换为新 value。

来看`InternalThreadLocalMap.get()`方法，目的就是获取线程内部的 InternalThreadLocalMap 对象，根据线程类型的不同，采取了不同逻辑：

```
    // InternalThreadLocalMap.java
    
    private static final ThreadLocal<InternalThreadLocalMap> slowThreadLocalMap =
                new ThreadLocal<InternalThreadLocalMap>();
    
    public static InternalThreadLocalMap get() {
        Thread thread = Thread.currentThread();
        // 1.如果当前线程为FastThreadLocalThread
        if (thread instanceof FastThreadLocalThread) {
            return fastGet((FastThreadLocalThread) thread);
        } 
        // 2.当前线程是普通Thread线程
        else {
            return slowGet();
        }
    }
    
    private static InternalThreadLocalMap fastGet(FastThreadLocalThread thread) {
        InternalThreadLocalMap threadLocalMap = thread.threadLocalMap();
        if (threadLocalMap == null) {
            thread.setThreadLocalMap(threadLocalMap = new InternalThreadLocalMap());
        }
        return threadLocalMap;
    }
    
    private static InternalThreadLocalMap slowGet() {
        InternalThreadLocalMap ret = slowThreadLocalMap.get();
        if (ret == null) {
            ret = new InternalThreadLocalMap();
            slowThreadLocalMap.set(ret);
        }
        return ret;
    }
```

*   对于 FastThreadLocalThread 类型的线程：直接取 ThreadLocalThread 线程对象内部的 InternalThreadLocalMap，如果不存在就创建并关联；
*   对于普通 Thread 线程：从 JDK 的 ThreadLocal 中获取 InternalThreadLocalMap。

下面通过一幅图描述两种不同线程获取 InternalThreadLocalMap 的方式，便于大家理解：

![](/img/network-program/netty/InternalThreadLocalMap2.png)

再来看如何将 InternalThreadLocalMap 中的数据替换为新的 value，也就是`setKnownNotUnset`方法：

1.  找到数组下标 index 位置，设置新的 value；
2.  将 FastThreadLocal 对象保存到待清理的 Set 中。

```
    // FastThreadLocal.java
    
    private void setKnownNotUnset(InternalThreadLocalMap threadLocalMap, V value) {
        if (threadLocalMap.setIndexedVariable(index, value)) {    // 找到数组下标 index 位置，设置新value
            // 将 FastThreadLocal 对象保存到待清理的 Set 中
            addToVariablesToRemove(threadLocalMap, this);
        }
    }
```

InternalThreadLocalMap 的 setIndexedVariable 方法，`index`就是 FastThreadLocal 对象创建时分配的唯一索引：

```
    // InternalThreadLocalMap.java
    
    public boolean setIndexedVariable(int index, Object value) {
        // indexedVariables就是InternalThreadLocalMap中用于存放数据的数组
        Object[] lookup = indexedVariables;
        if (index < lookup.length) {
            Object oldValue = lookup[index];    // 时间复杂度为 O(1)
            lookup[index] = value;
            return oldValue == UNSET;
        } else {
            // 数组扩容
            expandIndexedVariableTableAndSet(index, value);
            return true;
        }
    }
```

> InternalThreadLocalMap 以 index 为基准进行扩容，将数组扩容后的容量向上取整为 2 的次幂，然后将原数组内容拷贝到新的数组中，空余部分填充缺省对象 UNSET，最终把新数组赋值给 indexedVariables。

回到 setKnownNotUnset() 的主流程，向 InternalThreadLocalMap 添加完数据之后，接下就是将 FastThreadLocal 对象保存到待清理的 Set 中。我们继续看下 addToVariablesToRemove() 是如何实现的：

```
    // FastThreadLocal.java
    
    private static void addToVariablesToRemove(InternalThreadLocalMap threadLocalMap, FastThreadLocal<?> variable) {
        // 获取数组下标为 variablesToRemoveIndex 的元素
        Object v = threadLocalMap.indexedVariable(variablesToRemoveIndex);
        Set<FastThreadLocal<?>> variablesToRemove;
        if (v == InternalThreadLocalMap.UNSET || v == null) {
            // 创建 FastThreadLocal 类型的 Set 集合
            variablesToRemove = Collections.newSetFromMap(new IdentityHashMap<FastThreadLocal<?>, Boolean>());
            // 将 Set 集合填充到数组下标 0 的位置
            threadLocalMap.setIndexedVariable(variablesToRemoveIndex, variablesToRemove);
        } else {
            // 如果不是 UNSET，Set 集合已存在，直接强转获得 Set 集合
            variablesToRemove = (Set<FastThreadLocal<?>>) v;
        }
        // 将 FastThreadLocal 添加到 Set 集合中
        variablesToRemove.add(variable);
    }
```

为什么 InternalThreadLocalMap 要在数组下标为 `variablesToRemoveIndex` 的位置存放一个 FastThreadLocal 类型的 Set 集合呢？与 remove 方法有关。

#### remove 方法

我们回过头看下 remove() 方法：

```
    // FastThreadLocal.java
    
    public final void remove() {
        remove(InternalThreadLocalMap.getIfSet());
    }
    
    public final void remove(InternalThreadLocalMap threadLocalMap) {
        if (threadLocalMap == null) {
            return;
        }
        // 删除数组下标 index 位置对应的 value
        Object v = threadLocalMap.removeIndexedVariable(index);
        // 从数组下标 0 的位置取出 Set 集合，并删除当前 FastThreadLocal
        removeFromVariablesToRemove(threadLocalMap, this);
    
        if (v != InternalThreadLocalMap.UNSET) {
            try {
                // 空方法，用户可以继承实现
                onRemoval((V) v);
            } catch (Exception e) {
                PlatformDependent.throwException(e);
            }
        }
    }
```

在调用 remove 方法时，InternalThreadLocalMap 会从数组中定位到下标 index 位置的元素，并将 index 位置的元素覆盖为缺省对象 UNSET。接下来就需要清理当前的 FastThreadLocal 对象，此时 InternalThreadLocalMap 会取出数组下标 0 位置的 Set 集合，然后删除当前 FastThreadLocal：

```
    // FastThreadLocal.java
    
    private static void removeFromVariablesToRemove(
        InternalThreadLocalMap threadLocalMap, FastThreadLocal<?> variable) {
    
        Object v = threadLocalMap.indexedVariable(variablesToRemoveIndex);
        if (v == InternalThreadLocalMap.UNSET || v == null) {
            return;
        }
    
        Set<FastThreadLocal<?>> variablesToRemove = (Set<FastThreadLocal<?>>) v;
        variablesToRemove.remove(variable);
    }
```

最后 onRemoval() 方法起到什么作用呢？Netty 只是留了一处扩展，并没有实现，用户需要在删除的时候做一些后置操作，可以继承 FastThreadLocal 实现该方法。

#### get 方法

最后，我们来看下 FastThreadLocal 的 get 方法：

```
    // FastThreadLocal.java
    
    public final V get() {
        // 获取当前线程内部的InternalThreadLocalMap
        InternalThreadLocalMap threadLocalMap = InternalThreadLocalMap.get();
        // 获取本地变量
        Object v = threadLocalMap.indexedVariable(index);
        if (v != InternalThreadLocalMap.UNSET) {
            return (V) v;
        }
        // 为缺省值则初始化
        return initialize(threadLocalMap);
    }
    
    private V initialize(InternalThreadLocalMap threadLocalMap) {
        V v = null;
        try {
            // 调用用户重写的 initialValue 方法构造需要存储的对象数据
            v = initialValue();
        } catch (Exception e) {
            PlatformDependent.throwException(e);
        }
    
        threadLocalMap.setIndexedVariable(index, v);
        // 把当前 FastThreadLocal 对象保存到待清理的 Set 中
        addToVariablesToRemove(threadLocalMap, this);
        return v;
    }
```

```
    // InternalThreadLocalMap.java
    
    public Object indexedVariable(int index) {
        Object[] lookup = indexedVariables;
        return index < lookup.length? lookup[index] : UNSET;
    }
```

三、总结
----

本章，我对 Netty 中的线程本地工具类 FastThreadLocal 进行了深入讲解。FastThreadLocal 真的一定比 ThreadLocal 快吗？答案是不一定的，只有使用 FastThreadLocalThread 类型的线程才会更快，如果是普通线程反而会更慢。最后，我对 FastThreadLocal 进行一个总结：

*   **高效查找** 。FastThreadLocal 在定位数据的时候可以直接根据数组下标 index 获取，时间复杂度 O(1)。而 JDK 原生的 ThreadLocal 在数据较多时容易发生 Hash 冲突，线性探测法在解决 Hash 冲突时需要不停地向后寻找，效率较低。此外，FastThreadLocal 相比 ThreadLocal 数据扩容更加简单高效，FastThreadLocal 以 index 为基准向上取整到 2 的次幂作为扩容后容量，然后把原数据拷贝到新数组。而 ThreadLocal 由于采用的哈希表，所以在扩容后需要再做一轮 rehash。
*   **安全性更高** 。JDK 原生的 ThreadLocal 使用不当可能造成内存泄漏，只能等待线程销毁。在使用线程池的场景下，ThreadLocal 只能通过主动检测的方式防止内存泄漏，从而造成了一定的开销。而 FastThreadLocal 不仅提供了 remove() 主动清除对象的方法，而且在线程池场景中 Netty 还封装了 FastThreadLocalRunnable，FastThreadLocalRunnable 最后会执行 FastThreadLocal.removeAll() 将 Set 集合中所有 FastThreadLocal 对象都清理掉，