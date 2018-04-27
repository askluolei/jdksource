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

### 共享