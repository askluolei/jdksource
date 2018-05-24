[TOC]

# AQS 分析  
juc 下面很多同步器都是基于 AQS 的。 
因此，掌握了 AQS ,其他同步器的实现原理就都明白了    

## 概述 
AQS 的阻塞使用的是 LockSupport 这个类来实现的   
阻塞可以被线程的中断标记唤醒    

内部使用双向链表（prev，next）来维持阻塞的线程（使用 Node 来代表）    
链表的头节点为当前获取资源的线程节点    
链表是延时加载的，如果没有竞争发生的时候，链表不会初始化    

节点分为独占节点 和 共享节点    
独占节点的概念是，如果独占节点持有资源，那么后面的节点都得等待，等待该节点释放资源后唤醒下一个节点  
共享节点的概念是，如果共享节点获取到了资源，那么会判断后面的节点是不是共享节点，如果是，那就唤醒后一个节点（是否获取到资源得看子类实现）    

AQS 内部有一个 volatile int 类型的变量 state。  
来表示 AQS 的状态，子类可以在 state 上面存自己设计的值

还有条件 Condition  
AQS 内部有 Condition 的内部类实现（就是说，可以访问外部类的变量，方法） ConditionObject 
条件等待是在一个单向链表的（nextWaiter），也是基于 Node 的  

外部通常可能会调用 AQS 的方法有 

```java
// 独占节点相关的方法
acquire(int arg)
acquireInterruptibly(int arg) throws InterruptedException
tryAcquireNanos(int arg, long nanosTimeout) throws InterruptedException 
release(int arg)

// 共享节点相关的方法   
acquireShared(int arg)
acquireSharedInterruptibly(int arg) throws InterruptedException
tryAcquireSharedNanos(int arg, long nanosTimeout) throws InterruptedException
releaseShared(int arg)
```     

当然，还有其他 public 的方法，不过，这几个是重点方法，功能也是基于这几个方法实现的  

acquire 代表获取    
release 是释放  

特别注意的是调用 acquire ，获取资源，具体是否获取的到，得看子类实现 
```java
// 独占节点相关的方法
tryAcquire(int arg)
tryRelease(int arg)
// 共享节点相关的方法   
tryAcquireShared(int arg)
tryReleaseShared(int arg)
``` 

这上面4个方法是留给子类实现的，用来实现自己的同步器功能 
而在 AQS 内部 则是通过调用这4个方法，来判断是否获取，释放了节点 
如果获取，或者释放了，那么内部的同步队列（双向链表）的状态，节点状态做相应变更  

这里只是一个概述，下面跟着代码，来看看 AQS 内部怎么维持双向链表和阻塞的逻辑 

## 代码分析 

### 独占    
独占节点相关的是 acquire,release    
后面不带 share 的，那就是独占节点的方法 
我们先看一下获取独占节点    

#### acquire(int arg)    
```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
``` 

可以看到，首先调用了 tryAcquire 方法，这个是由子类实现的。  
任何同步器，肯定有自己的阻塞条件和逻辑，因此这个只是一个模版方法，将是否获取到资源的条件留给子类实现    
如果子类说没获取到，那么 AQS 将保证，本线程阻塞，并且可以被取消和激活。 
JDK 源码大部分都写的比较精简，行数是减少了，但是也不太容易看    
首先看 addWaiter 方法   
这个方法是将节点添加到同步队列（双向链表）里面去，根据参数，我们可以看到是独占节点（Node.EXCLUSIVE）    
添加的节点是到链表尾部，这个时候，如果链表没有初始化，则先初始化    
```java    
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}
``` 
这里只是讲解，源码上注释可以看  
如果链表没有初始化（tail == null），调用 enq 方法，这个方法是初始化链表 + 加入节点到队尾    
如果已经初始化了，我们可以看到是直接设置节点到队尾（一次CAS 操作尝试），如果失败了（可能多个线程都要阻塞），还是调用 enq    

```java
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
``` 
enq 方法通过 循环 + CAS 来保证操作成功（为什么，因为即使在竞争情况下，肯定会有一个线程成功操作，其余的线程就继续再循环里面 CAS）    
可以看到，如果双向链表没有初始化的情况下，添加一个空节点当头节点，然后参数节点添加到队尾，如果有并发情况，则依次添加到队尾  
这里需要注意的一点是，返回的节点是给定节点的**前驱节点**


回到上面 acquire 方法
```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
``` 

添加到链表里面去了之后，调用 acquireQueued 方法，这个方法是处在队列里面的节点尝试获取资源，如果获取不到就阻塞   
```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
``` 
无论需不需要抛中断异常，中断条件都是要判断的。  
还是在循环里面，尝试获取资源和阻塞  
首先判断的是前一个节点是否是头节点，如果是，那就尝试去获取资源（tryAcquire），记住，是否获取到资源（获取到资源的线程不阻塞）由实现类来决定  
获取到的资源节点设置为头节点，并将前一个节点断开。返回是否有中断    
如果没有获取到，或者前一个节点不是头节点，那就可以开始判断是否需要阻塞了 shouldParkAfterFailedAcquire   
```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        return true;
    if (ws > 0) {
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
``` 
什么条件下会阻塞呢？当前一个节点的等待状态为 SIGNAL 的时候返回true，其他情况都返回 false    
因为 AQS 的实现，保证等待状态为 SIGNAL 的节点会至少唤醒一次下一个节点   
这里还删除了取消状态的节点，如果前一个节点的等待状态不为 SIGNAL，则 CAS 操作修改为 SIGNAL   
这里是有可能失败的，不过外层是循环。    

深入分析一下：  
为什么当前一个节点是头节点的时候，可以去尝试获取资源？  
因为，可能头节点将要释放资源了，这个时候有可能获取到。  
为什么一定要等待设置前一个节点为 SIGNAL 状态的时候才阻塞。parkAndCheckInterrupt 方法就是阻塞    
```java
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();
}
``` 
因为 AQS 保证的是 SIGNAL 的节点会通知下一个节点（实现后面分析）。   
如果没等到设置前一个节点状态为 SIGNAL，本节点可能得不到通知 

在最后还有一段代码  
```java
finally {
    if (failed)
        cancelAcquire(node);
}
```
在本方法中，不会走到这里的，这个是在现时等待，响应中断的时候可能到这里，用来取消节点    

```java
private void cancelAcquire(Node node) {
    if (node == null)
        return;
    node.thread = null;
    Node pred = node.prev;
    while (pred.waitStatus > 0)
        node.prev = pred = pred.prev;

    Node predNext = pred.next;

    node.waitStatus = Node.CANCELLED;
    if (node == tail && compareAndSetTail(node, pred)) {
        compareAndSetNext(pred, predNext, null);
    } else {
        int ws;
        if (pred != head &&
            ((ws = pred.waitStatus) == Node.SIGNAL ||
                (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
            pred.thread != null) {
            Node next = node.next;
            if (next != null && next.waitStatus <= 0)
                compareAndSetNext(pred, predNext, next);
        } else {
            unparkSuccessor(node);
        }
        node.next = node; // help GC
    }
}
``` 

先删除节点前面已经取消的节点，然后设置本节点的状态为 CANCELLED 取消 
如果本节点是尾节点，并且设置前一个节点为尾节点成功，删除本节点就行了    
否则 else 下面的逻辑就有点复杂了 
判断 前一个节点不是头节点 && 前一个节点的状态为 SIGNAL ，或者（非取消节点）成功设置为 SIGNAL && 代表的线程不为 null   

那么本节点就是可以获得前一个节点的通知的，把这个条件转移到下一个节点，然后删除本节点    
否则，就激活下一个节点（想想阻塞那里，会将前面的取消节点删除）。    
最后的结果就是删除给定节点，保证后一个节点可以得到通知      

再看一下激活后一个节点的方法 unparkSuccessor    
```java
private void unparkSuccessor(Node node) {
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);
}
``` 
当节点的状态 < 0 ，则设置为0，如果后一个节点为null，或者为取消状态，那么从尾节点开始向前寻找，离本节点最近的一个非取消节点  
如果找到了，就唤醒改线程。  

acquire 方法最后的就是 selfInterrupt 方法了，这个是设置线程的中断标记   
除了 acquire 方法，还有 acquireInterruptibly tryAcquireNanos 方法，分别是不响应中断异常，和限时等待，逻辑大体类似   

再看看 release  
```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
``` 
是释放独占节点，首先还是调用子类实现的 tryRelease 方法，子类说可以释放了，那么释放当前头节点（获取资源的节点为头节点）  
这里，只要当头节点的 waitStatus != 0 就通知后面的节点，这里需要知道 >0 是取消，<0 是需要获取通知（有几个小于0的值） 


### 共享    
共享的获取方法是 acquireShared  
```java
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}
``` 
还是先调用子类实现的 tryAcquireShared 方法，如果失败了(返回值小于0)，继续调用 doAcquireShared   
```java
private void doAcquireShared(int arg) {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```     
基本上跟获取独占节点的差不多，不同的是，这里添加的时一个共享节点。  
这里，在获取到资源后，调用了 setHeadAndPropagate 方法   
这里，将本节点设置为头节点。并唤醒下一个共享节点。  
```java
private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head; // Record old head for check below
    setHead(node);
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        if (s == null || s.isShared())
            doReleaseShared();
    }
}
``` 
上面经过条件判断后，如果后一个节点为共享节点，那么调用 doReleaseShared 方法。

```java
private void doReleaseShared() {
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }
            else if (ws == 0 &&
                        !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}
``` 
这里就是激活后一个节点，当节点的状态为 SIGNAL 的时候，唤醒后一个节点代表的线程。    
独占节点判断是否阻塞也是方法 shouldParkAfterFailedAcquire 来判断的，这里，如果节点要阻塞，那么会设置前一个节点的状态为 SIGNAL   
当它被唤醒后，如果有可能获取资源（前一个节点为 头节点），还是会调用 tryAcquireShared 方法尝试获取。 

可以看到，共享节点的逻辑是如果一个共享节点获取资源，那么会激活后一个共享节点，至于这个共享节点是否获取资源，得看子类实现的 tryAcquireShared 方法。  
AQS 只做到激活后一个共享节点，如果也能获取，那么继续往后激活。  

同样的，获取共享节点还有方法 acquireSharedInterruptibly tryAcquireSharedNanos   

释放共享节点 releaseShared  
```java
public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
``` 

也是先调用 tryReleaseShared 子类实现，判断是否释放，如果是否，还是调用 doReleaseShared 方法，这个方法上面看到过了   

AQS 的重点方法就这些，还有一部分是条件的实现    

### 条件 Condition   
条件就是通知机制，线程可以在条件上等待，其他线程可以激活一个条件上的线程，或者激活条件上的所有线程。    
实现方式是内部类，上面说过 Node 有个 nextWaiter 自动，就是维护单向链表用的，维护的就是条件上等待的节点，当被唤醒后，会将单向链表的节点，加入到双向链表中，等待获取资源  
所以 条件是跟锁一起工作的，必须先获取资源，才能调用条件上面的方法。 
我们看一个条件上的等待 await    
```java
public final void await() throws InterruptedException {
    // 如果中断了，就抛中断异常
    if (Thread.interrupted())
        throw new InterruptedException();
    // 添加一个节点到条件队列尾部
    Node node = addConditionWaiter();
    // 释放锁资源（如果没持有锁，会抛异常出来）
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    // 循环判断，如果当前节点不在同步队列中（当调用 signal 方法的时候，会将节点放入同步队列）
    while (!isOnSyncQueue(node)) {
        // 线程阻塞，正常流程应该是到这里
        LockSupport.park(this);
        // 阻塞取消，出来判断中断状态,如果 ！= 0，就是没有中断
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    // 从上面出来后，尝试获取锁，这里还是会触发等待锁的阻塞（如果要等的话），获取成功，处理了继续判断中断状态
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    // 如果 当前节点在条件队列上后后续节点，那么清除一遍无效节点
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    // 如果有中断状态，判断是否要抛异常
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
``` 
添加一个条件节点到 条件队列上面
```java
private Node addConditionWaiter() {
    Node t = lastWaiter;
    // 如果最后一个节点取消了，那么清理整个队列中取消的节点
    if (t != null && t.waitStatus != Node.CONDITION) {
        unlinkCancelledWaiters();
        t = lastWaiter;
    }
    // 新建一个mode 为 CONDITION 的节点
    Node node = new Node(Thread.currentThread(), Node.CONDITION);
    if (t == null)
        firstWaiter = node;
    else
        t.nextWaiter = node;
    lastWaiter = node;
    return node;
}
``` 
代码上又点注释，整个逻辑不是很负责。    
除了 await 外，还有几个限时等待的方法，和不响应中断异常的方法，在源码上已经加了注释了，就不一一细讲了，实现大体类似。   
再看一下 signal 方法    
```java
public final void signal() {
    // 如果锁的持有者不是当前线程，抛异常
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    // 如果等待队列不为空，激活第一个
    if (first != null)
        doSignal(first);
}
``` 
很简单的，直接调用 doSignal 方法。
```java
private void doSignal(Node first) {
    do {
        // 从first 开始向后遍历
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        // first 节点从链表中断开
        first.nextWaiter = null;
        // 转换first节点，如果成功，跳出循环，如果失败，继续下一个节点，获取节点为null，跳出循环
    } while (!transferForSignal(first) &&
                (first = firstWaiter) != null);
}
``` 
重点在 transferForSignal 方法，如果激活成功返回 true。这里的逻辑是一直向后，直到激活一个。  
```java
final boolean transferForSignal(Node node) {
    /*
    * 如果不能改变 等待状态值，那么这个节点就是取消状态(就是不通知)
    * If cannot change waitStatus, the node has been cancelled.
    */
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;

    /*
    * 将节点插入到同步队列尾部
    * 如果 前一个节点的等待状态是取消状态，那么设置为 SIGNAL，代表本节点需要收到前一个节点的通知
    * 如果设置失败，那么立马激活本节点代表的线程，返回true
    * Splice onto queue and try to set waitStatus of predecessor to
    * indicate that thread is (probably) waiting. If cancelled or
    * attempt to set waitStatus fails, wake up to resync (in which
    * case the waitStatus can be transiently and harmlessly wrong).
    */
    Node p = enq(node);
    int ws = p.waitStatus;
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    return true;
}
``` 
这里就是将单向链表的节点，添加的双向链表（等待之前释放的资源）  

signalAll 方法就是遍历等待队列，调用这个方法。  

## 总结 
了解了上面的一些内容，那么再分析 CountDownLatch, CyclicBarrier, ReentrantLock, ReentrantReadWriteLock, Semaphore 这里同步器就很轻松了。 
源码里面有注释。    
PS: StampedLock 没有分析，是新的读写锁，思路是，读不阻塞写，当写发生后，可以重读