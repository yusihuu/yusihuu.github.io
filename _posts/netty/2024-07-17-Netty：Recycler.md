---
layout:     post
title:      透彻理解 java 网络编程(十六)
subtitle:   netty 原理：Recycler 对象池
date:       2024-07-17
author:     yusihu
header-style: text
catalog: true
tags:
- 网络编程
- netty
- 开源框架
---
> 本章，我将对 Netty 中的 & nbsp;Recycler 对象池 & nbsp; 进行讲解。所谓对象池，顾名思义，就是程序对象（Java 对象）的一个缓存池。与内存池类似，对象池的目的也是

本章，我将对 Netty 中的 **Recycler 对象池** 进行讲解。所谓对象池，顾名思义，就是程序对象（Java 对象）的一个缓存池。与内存池类似，对象池的目的也是为了提升 Netty 的并发处理能力，避免频繁创建和销毁对象所带来的性能损耗。

那么，Netty 是如何实现对象池的？我们在实践中又该如何运用对象池呢？带着这两个问题，我们来看 Netty 中 Recycler 对象池的设计与实现。

一、Recycler 简介
-------------

Recycler 是 Netty 实现的轻量级对象回收站，借助 Recycler 可以完成对象的获取和回收。

### 1.1 基本使用

我们通过一个例子直观感受下 Recycler 如何使用。

假设我们有一个`User`类，需要实现`User`对象的复用。首先，我们定义一个 UserRecycler 对象池：

```
    // UserRecycler.java
    
    public class UserRecycler extends Recycler<User> {
        @Override
        protected User newObject(Handle handle) {
            return new User(handle);
        }
    }
```

User 对象：

```
    // User.java
    
    public class User {
    
        private Recycler.Handle<User> handle;
        private String name;
    
        public User(Recycler.Handle<User> handle) {
            this.handle = handle;
        }
    
        public String getName() {
            return name;
        }
    
        public void setName(String name) {
            this.name = name;
        }
    
        public void recycle() {
            handle.recycle(this);
        }
    }
```

测试用例：

```
    public class RecyclerTest {
    
        private static final UserRecycler userRecycler = new UserRecycler();
    
        public static void main(String[] args) {
    
            // 1.从对象池获取User对象
            User user1 = userRecycler.get();
            user1.setName("hello");
    
            // 2.回收对象
            user1.recycle();
    
            // 3.从对象池获取对象
            User user2 = userRecycler.get();
    
            System.out.println(user2.getName());
            System.out.println(user1 == user2);
        }
    }
```

打印结果如下：

```
    hello
    true
```

### 1.2 内部结构

Recycler 的内部结构，如下图所示：

![](/img/network-program/netty/Recycler.png)

可以看到，Recycler 一共包含四个核心组件： **Stack** 、 **WeakOrderQueue** 、 **Link** 、 **DefaultHandle** 。各个组件的关系，可以通过下图描述：

![](/img/network-program/netty/Recycler1.png)

#### Stack

Stack 是每个线程私有的，用于存储当前线程回收的对象。Netty 为了避免多线程场景下的锁竞争问题，每个线程都会持有自己的对象池，内部通过`FastThreadLocal`来实现线程私有化。

> FastThreadLocal 可以理解为 Netty 里的 ThreadLocal，后续我章节会专门对它进行讲解。

来看下 Stack 的源码：

```
    // Recycler.Stack.java
    
    private static final class Stack<T> {
           // 所属的Recycler
        final Recycler<T> parent;
    
        // 所属线程的弱引用
        final WeakReference<Thread> threadRef;
    
        // WeakOrderQueue最大个数
        private final int maxDelayedQueues;
    
        // 其它线程回收对象时，最多可以回收当前线程创建的对象个数
        final AtomicInteger availableSharedCapacity;
    
        // 对象池的最大大小，默认4k
        private final int maxCapacity;
    
        // 存储缓存数据的数组
        DefaultHandle<?>[] elements;
    
        // 缓存的DefaultHandle对象个数
        int size;
    
        // WeakOrderQueue链表的三个重要指针
        private WeakOrderQueue cursor, prev;
        private volatile WeakOrderQueue head;
    
        //...
    }
```

Stack 内部有一个 WeakOrderQueue 链表，链表的每个节点指向一个 WeakOrderQueue 队列，每个队列里保存着 **其它线程所释放的对象** 。

比如，ThreadA 表示当前线程，那么它 Stack 内部的 WeakOrderQueue 链表，就存储着 ThreadB、ThreadC 等其它线程释放的对象。

`availableSharedCapacity`字段的默认值为：`new AtomicInteger(max(maxCapacity / maxSharedCapacityFactor, LINK_CAPACITY)) = 16K`。它的含义是：其它线程在回收对象时，最多可以回收 ThreadA 创建的对象个数不能超过 availableSharedCapacity。

#### WeakOrderQueue

WeakOrderQueue 用于存储其它线程回收当前线程所分配的对象，并且在合适的时机，Stack 会从其它线程的 WeakOrderQueue 中收割对象。比如，ThreadB 从 ThreadA 回收对象时，会将回收到的对象放到 ThreadA 的 WeakOrderQueue 中。

#### Link

每个 WeakOrderQueue 中都包含一个 Link 链表，回收对象都会被存在 Link 链表中的节点上，每个 Link 节点默认存储 16 个对象，当每个 Link 节点存储满了会创建新的 Link 节点放入链表尾部。

#### DefaultHandle

DefaultHandle 实例中保存了实际回收的对象，Stack 和 WeakOrderQueue 都使用 DefaultHandle 存储回收的对象。

二、Recycler 原理
-------------

对 Recycler 有了初步认识后，我们再来看从 Recycler 获取对象和回收对象的原理。

### 2.1 获取对象

从对象池中获取对象的入口是在`Recycler.get()`方法，该方法的逻辑非常清晰：

1.  首先，通过 FastThreadLocal 获取当前线程的私有 Stack；
2.  尝试出栈一个 DefaultHandle 对象，如果结果为空说明 Stack 中没有可用 DefaultHandle 对象，则调用 newObject 生成一个新对象，并完成 handle 与对象和 Stack 的绑定。

```
    // Recycler.java
    
    public final T get() {
        if (maxCapacityPerThread == 0) {
            return newObject((Handle<T>) NOOP_HANDLE);
        }
    
        // 获取当前线程的私有Stack
        Stack<T> stack = threadLocal.get();
        // 出栈一个DefaultHandle对象
        DefaultHandle<T> handle = stack.pop();
        if (handle == null) {
            handle = stack.newHandle();
            // 创建对象，并保存到 DefaultHandle
            handle.value = newObject(handle);
        }
        return (T) handle.value;
    }
```

我们再来看 Stack 的出栈操作，跟进下`Stack.pop()`的源码：

```
    // Recycler.Stack.java
    
    DefaultHandle<T> pop() {
        int size = this.size;
        if (size == 0) {
            // 尝试从其它线程回收的对象中转移一些到自己的elements数组中
            if (!scavenge()) {
                return null;
            }
            size = this.size;
            if (size <= 0) {
                // double check, avoid races
                return null;
            }
        }
        // 从栈顶弹出元素
        size --;
        DefaultHandle ret = elements[size];
        elements[size] = null;
        this.size = size;
    
        if (ret.lastRecycledId != ret.recycleId) {
            throw new IllegalStateException("recycled multiple times");
        }
        ret.recycleId = 0;
        ret.lastRecycledId = 0;
        return ret;
    }
```

整个逻辑也很清晰：

1.  如果当前线程的私有 Stack 中有可用的对象，直接将对象出栈；
2.  如果 elements 数组中没有可用的对象，则调用 scavenge 方法。scavenge 的作用是从其它线程回收的对象中转移一些到 elements 数组当中。

`Stack.scavenge()`方法非常有意思，它会想办法从 WeakOrderQueue 链表中迁移部分对象。它的设计思想与 J.U.C 中的 ForkJoinPool 有些类似，ForkJoinPool 采用了 “工作窃取” 算法，也是从其它线程中 “窃取” 对象执行。

每个 Stack 都有一个 WeakOrderQueue 链表，链表中的每个 WeakOrderQueue 中保存了其它线程回收的对象：

```
    // Recycler.Stack.java
    
    private boolean scavenge() {
        // 尝试从 WeakOrderQueue 中转移对象到Stack中
        if (scavengeSome()) {
            return true;
        }
    
        // 如果迁移失败，就会重置cursor指针到head节点
        prev = null;
        cursor = head;
        return false;
    }
```

继续看`scavengeSome`方法：

```
    // Recycler.Stack.java
    
    private boolean scavengeSome() {
        WeakOrderQueue prev;
        // cursor指针指向当前WeakorderQueueu链表的读取位置
        WeakOrderQueue cursor = this.cursor;
        // 如果cursor指针为null, 则是第一次从WeakorderQueueu链表中获取对象
        if (cursor == null) {
            prev = null;
            cursor = head;
            if (cursor == null) {
                return false;
            }
        } else {
            prev = this.prev;
        }
    
        // 不断循环从WeakOrderQueue链表中找到一个可用的对象实例
        boolean success = false;
        do {
             // 尝试迁移WeakOrderQueue中部分对象实例到Stack中
            if (cursor.transfer(this)) {
                success = true;
                break;
            }
            WeakOrderQueue next = cursor.getNext();
            if (cursor.get() == null) {
                // 如果已退出的线程还有数据
                if (cursor.hasFinalData()) {
                    for (;;) {
                        if (cursor.transfer(this)) {
                            success = true;
                        } else {
                            break;
                        }
                    }
                }
                // 将已退出的线程从WeakOrderQueue链表中移除
                if (prev != null) {
                    cursor.reclaimAllSpaceAndUnlink();
                    prev.setNext(next);
                }
            } else {
                prev = cursor;
            }
            // 将cursor指针指向下一个WeakOrderQueue
            cursor = next;
        } while (cursor != null && !success);
        this.prev = prev;
        this.cursor = cursor;
        return success;
    }
```

scavenge 的源码中，首先会从 cursor 指针指向的 WeakOrderQueue 节点回收部分对象到 Stack 的 elements 数组中，如果没有回收到数据就会将 cursor 指针移到下一个 WeakOrderQueue，重复执行以上过程直至回收到对象为止，可以通过下图来理解：

![](/img/network-program/netty/Recycler3.png)

### 2.2 回收对象

理解了如何从 Recycler 获取对象之后，我们再来看 Recycler 是如何回收对象的，直接定位到对象回收的源码入口 `DefaultHandle.recycle()`：

```
    // Recycler.DefaultHandle.java
    
    public void recycle(Object object) {
        if (object != value) {
            throw new IllegalArgumentException("object does not belong to handle");
        }
    
        Stack<?> stack = this.stack;
        if (lastRecycledId != recycleId || stack == null) {
            throw new IllegalStateException("recycled already");
        }
        stack.push(this);
    }
```

在回收对象时，会向 Stack 中 push 对象，push 会分为 **当前线程回收** 和 **其它线程回收** 两种情况，分别对应`pushNow`和`pushLater`两个方法：

```
    // Recycler.DefaultHandle.java
    
    void push(DefaultHandle<?> item) {
        Thread currentThread = Thread.currentThread();
        if (threadRef.get() == currentThread) {
            // 当前线程回收
            pushNow(item);
        } else {
            // 其它线程回收
            pushLater(item, currentThread);
        }
    }
```

#### 当前线程回收

当前线程回收对象的逻辑非常简单，就是直接向 Stack 的 elements 数组中添加数据，对象会被存放在栈顶指针指向的位置。如果超过了 Stack 的最大容量，那么对象会被直接丢弃，这里使用了`dropHandle`方法控制对象的回收速率，每 8 个对象会有一个被回收到 Stack 中：

```
    // Recycler.DefaultHandle.java
    
    private void pushNow(DefaultHandle<?> item) {
        // 防止被多次回收
        if (item.recycleId != 0 || !item.compareAndSetLastRecycledId(0, OWN_THREAD_ID)) {
            throw new IllegalStateException("recycled already");
        }
        item.recycleId = OWN_THREAD_ID;
    
        int size = this.size;
        // 1. 超出最大容量 2. 控制回收速率
        if (size >= maxCapacity || dropHandle(item)) {
            return;
        }
        if (size == elements.length) {
            elements = Arrays.copyOf(elements, min(size << 1, maxCapacity));
        }
    
        elements[size] = item;
        this.size = size + 1;
    }
```

#### 其它线程回收

其它线程回收时，首先通过 FastThreadLocal 取出当前对象的`DELAYED_RECYCLED`缓存：DELAYED_RECYCLED 存放着当前线程帮助其它线程回收对象的映射关系。

举个例子，假如 item 是 ThreadA 分配的对象，当前线程是 ThreadB，此时 ThreadB 帮助 ThreadA 回收 item，那么 DELAYED_RECYCLED 放入的 key 是 StackA，然后从 delayedRecycled 中取出 StackA 对应的 WeakOrderQueue，如果 WeakOrderQueue 不存在，那么为 StackA 新创建一个 WeakOrderQueue，并将其加入 DELAYED_RECYCLED 缓存。

`WeakOrderQueue.allocate()` 会检查帮助 StackA 回收的对象总数是否超过 2K 个，如果没有超过 2K，会将 StackA 的 head 指针指向新创建的 WeakOrderQueue，否则不再为 StackA 回收对象。

当然 ThreadB 不会只帮助 ThreadA 回收对象，它可以帮助其它多个线程回收，所以 DELAYED_RECYCLED 使用的 Map 结构，为了防止 DELAYED_RECYCLED 内存膨胀，Netty 也采取了保护措施，从 `delayedRecycled.size() >= maxDelayedQueues` 可以看出，每个线程最多帮助 2 倍 CPU 核数的线程回收线程，如果超过了该阈值，假设当前对象绑定的为 StackX，那么将在 Map 中为 StackX 放入一种特殊的 WeakOrderQueue.DUMMY，表示当前线程无法帮助 StackX 回收对象。

```
    // Recycler.DefaultHandle.java
    
    private void pushLater(DefaultHandle<?> item, Thread thread) {
        if (maxDelayedQueues == 0) {
            return;
        }
        // 当前线程帮助其它线程回收对象的缓存映射
        Map<Stack<?>, WeakOrderQueue> delayedRecycled = DELAYED_RECYCLED.get();
        // 取出 Stack 对应的 WeakOrderQueue
        WeakOrderQueue queue = delayedRecycled.get(this);
        if (queue == null) {
            // 最多帮助2 * CPU核数的线程回收线程
            if (delayedRecycled.size() >= maxDelayedQueues) {
                // 存一个Dummy节点，表示当前线程无法再帮助该Stack回收对象
                delayedRecycled.put(this, WeakOrderQueue.DUMMY);
                return;
            }
            // 新建WeakOrderQueue
            if ((queue = newWeakOrderQueue(thread)) == null) {
                return;
            }
            delayedRecycled.put(this, queue);
        } else if (queue == WeakOrderQueue.DUMMY) {
            return;
        }
        // 添加对象到WeakOrderQueue的Link链表中
        queue.add(item);
    }
```

至此，Recycler 如何获取和回收对象的底层原理就全部分析完了，Recycler 回收对象时向 WeakOrderQueue 中存放对象，从 Recycler 获取对象时，WeakOrderQueue 中的对象会作为 Stack 的储备，而且有效地解决了跨线程回收的问题。

三、总结
----

Netty 中大量运用了 Recycler。 例如，我们在使用 PooledDirectByteBuf 时，并不是每次都去创建新的对象，而是从对象池中获取预先分配好的对象实例，不再使用 PooledDirectByteBuf 时，会被回收归还到对象池中。

最后，对 Recycler 对象池进行一个总结：

*   对象池有两个重要的组成部分：Stack 和 WeakOrderQueue；
*   Recycler 获取对象时，优先从 Stack 中查找，如果 Stack 没有可用对象，会尝试从 WeakOrderQueue 迁移部分对象到 Stack 中；
*   Recycler 回收对象时，分为当前线程对象回收和其它线程对象回收两种情况，当前线程回收直接向 Stack 中添加对象，其它线程回收向 WeakOrderQueue 中的 Link 添加对象；
*   对象回收时会控制回收速率，每 8 个对象会回收一个，其他的全部丢弃。