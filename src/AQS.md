# AQS(AbstractQueuedSynchronizer)

> 这是一个有关于锁的模板类, 需要子类实现一些方法来使用, 通过一个由volatile修饰的state变量来代表阻塞非阻塞, 并维护一个Sync Queue(并发安全的CLH Queue)放置等待的线程, 和一个Condition Queue放置阻塞的线程.
>
> AQS 内部定义获取锁(acquire), 释放锁(release)的主逻辑, 子类实现响应的模版方法即可
>
> 支持共享和独占两种模式(共享模式时只用 Sync Queue, 独占模式有时只用 Sync Queue, 但若涉及 Condition, 则还有 Condition Queue); 独占是排他的.
>
> 支持 不响应中断获取独占锁(acquire), 响应中断获取独占锁(acquireInterruptibly), 超时获取独占锁(tryAcquireNanos); 不响应中断获取共享锁(acquireShared), 响应中断获取共享锁(acquireSharedInterruptibly), 超时获取共享锁(tryAcquireSharedNanos);
>
> 在子类的 tryAcquire, tryAcquireShared 中实现公平与非公平的区分
>
> AQS 为一个基础模板, 具体使用由各个子类继承使用
>
> 独占模式: ReentrantLock, ReentrantReadWriteLock 中的 WriteLock等.
>
> 共享模式: ReentrantReadWriteLock 中的 ReadLock, Semaphore, CountDownLatch等.

## Node

> ![](\Note\src\images\node.jpg)
>
> AQS 中有一个静态内部类Node, 他是 Sync Queue 和 Condition Queue 内保存的类, 每个线程对应一个Node.
>
> 锁类型
>
> ```java
> /** 表明该Node想要共享锁 */
> static final Node SHARED = new Node();
> /** 表明该Node想要独占锁 */
> static final Node EXCLUSIVE = null;
> ```
>
> waitStatus表示该Node 等待的状态:
>
> ```java
> /** 表明该Node所对应的Thread已经取消掉了 */
> static final int CANCELLED =  1;
> /** 表明当该Node结束时需要唤醒(unpack)下一个Node */
> static final int SIGNAL    = -1;
> /** 表明该Node目前正在Condition Queue中等待 */
> static final int CONDITION = -2;
> /** 表明该Node的下一个Node也可以使用锁(如何该锁是Shard 模式下也就是共享锁) */
> static final int PROPAGATE = -3;
> volatile int waitStatus;
> ```
>
> Link to predecessor node and successor node: 链接上一个和下一个node
>
> ```java
> volatile Node prev;
> volatile Node next;
> ```
>
> node代表的节点:
>
> ```java
> volatile Thread thread;
> ```
>
> 在Condition Queue中才会使用的用来获取下一个在Condition中等待的node
>
> 在Sync Queue中主要用来判断锁的模式为null 表示独占锁
>
> ```java
> Node nextWaiter;
> ```

## Sync Queue

![](\Note\src\images\SyncQueue.jpg)

> 新增的node将至于尾部tail, 获取锁的node将至于头部head
>
> prev 通常用来应对某个node的waitStatus为取消时, 假设Sync Queue中由 a, b, c 三个node, 当b被标记为取消时, 需将a.next 指向c, c的prev 指向a, 此时就需要用到b的prev 去找到a.
>
> next 主要用来, 当当前Node 被标记为Signal时, 当他准备放弃锁时, 需通知他的后继Node去获取锁.

## 来看看方法

### Acquire 获取锁流程

```java
/**
* 独占模式获取锁,且忽视interruptException 
*/
public final void acquire(int arg) {
    if (!tryAcquire(arg) && // 尝试获取锁,成功true,失败false
        // 获取锁失败 将线程封装为独占模式的Node, 并再次尝试获取锁,失败则pack自己
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        // acquireQueued如果中途报interruptException,因为是忽视该异常的,所以这里再次打断自己
        selfInterrupt();
}
```

```java
/**
* 该方法需要子类自我实现.具体内容为子类判断是否获取到锁的逻辑
* 获取到锁返回true, 失败则false
*/
protected boolean tryAcquire(int arg) {
        throw new UnsupportedOperationException();
}
```

```java
/**
* 当tryAcquire返回false,获取锁失败时,此方法运行将线程封装为Node,如果Sync Queue未初始化,则初始化Queue 并将Node置于尾部
*/
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    Node pred = tail;
    if (pred != null) { // 将node置于尾部
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node); // 队列为空初始化队列并将node置于尾部
    return node;
}
```

```java
    
/**
* 将node 插入队尾, 当队列未初始化时 初始化队列.
* @param node the node to insert
* @return node's predecessor
*/
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node())) // 设置head为new Node()
                tail = head; // 当队列中只有一个node时 head和tail为相同node
        } else {
            node.prev = t; // 将新 node 的prev 指向老tail
            if (compareAndSetTail(t, node)) { // 将新node 赋值给老tail
                t.next = node; //将新node 赋值给老tail.next
                return t;
            }
        }
    }
}
```

```java
/**
* 为node再次获取锁失败则park 阻塞自己
*/
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor(); // 获取前一个Node
            // 判断前一个Node是否是head, 如果是 则再次尝试获取锁(因为前一个是head 说明下一个轮到就是传入的node 有很大几率能马上获取锁)
            if (p == head && tryAcquire(arg)) { 
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted; //获取成功返回 是否中途被打断过
            }
            // park当前node,并将前node的waitStatus设置为Signal,这样前面的Node释放锁时会唤醒后node, 此方法会进入2次,来保证前node被设置为Signal
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            // 如果异常出现则将此node的waitStatus设置为Cancelled
            cancelAcquire(node);
    }
}
```

```java
    /**
     * 将前nodewaitStatus设置为Signal, 如果前node waitStatus为cancelled,则跳过他找寻前面最近的一个节点,将node挂在他后面并将他设置为signal.
     */
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL)
            /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
            return true;
        if (ws > 0) {
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
```

```java

    /**
     * Convenience method to park and then check if interrupted
     *
     * @return {@code true} if interrupted
     */
    private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this); // 阻塞线程
        return Thread.interrupted(); // 被唤醒之后检查是否被打断过,此唤醒也有可能是因为被打断所以才唤醒
    }
```

#### 流程图

![](\Note\src\images\20190426161550.png)