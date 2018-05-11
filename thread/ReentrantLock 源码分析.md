[TOC]    

# ReentrantLock 源码分析    
基本所有的同步器都是基于 `AbstractQueuedSynchronizer` 实现的，但是直接分析 `AbstractQueuedSynchronizer` 有太复杂，因此，从一个具体的实现 `ReentrantLock`，可重入锁开始分析    

## ReentrantLock 结构    
`ReentrantLock` 实现了 `Lock` 接口，在其内部有一个 静态抽象类 `Sync` 继承了 `AbstractQueuedSynchronizer`,这是大部分同步器的实现方式，在内部通常有一个自定义的 `Sync` 类继承 `AbstractQueuedSynchronizer`    
`ReentrantLock` 内部对 `Sync` 有两个实现 `NonfairSync`, `FairSync` ，看名字就知道是公平锁，和非公平锁。    
构造 `ReentrantLock`,默认是非公平锁，可以指定一个 `boolean` 设置为公平锁，这两个有啥区别，下面看源码分析。    

### lock 方法

首先分析的情况是默认的非公平锁的 `lock` 方法    
直接就是调用内部 `sync` 对象的 `lock` 方法 
```java
public void lock() {
        sync.lock();
    }
```    

我们先看非公平锁的实现    
```java
final void lock() {
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }
```    
首先调用 `compareAndSetState(0, 1)` 尝试获取，如果设置成功了，那么就代表锁被当前线程获得了 `setExclusiveOwnerThread(Thread.currentThread())`,这个方法设置当前获取锁的线程对象。    
这两个方法都是 `AbstractQueuedSynchronizer` 的方法，到这里我们就得去简单的分析一下  `AbstractQueuedSynchronizer` 内部的情况了    


`AbstractQueuedSynchronizer` 内部维护了一个双向链表结构. 节点对象就是静态内部类 `Node`,看其中的成员

```java
/** 表示节点正处在共享模式下等待的标记 **/
        static final Node SHARED = new Node();
        /**表示节点正在以独占模式等待的标记*/
        static final Node EXCLUSIVE = null;
        /**waitStatus值,表示线程已取消 */
        static final int CANCELLED = 1;
        /** waitStatus值,表示后继线程需要取消挂起 */
        static final int SIGNAL = -1;
        /** waitStatus值，表示线程正在等待条件 */
        static final int CONDITION = -2;
        /**waitStatus值指示下一个acquireShared应无条件传播*/
        static final int PROPAGATE = -3;
        /**
         * 状态字段，仅接受值:
         *
         * SIGNAL:值为-1 ，后继节点的线程处于等待状态，
         * 而当前节点的线程如果释放了同步状态或者被取消，
         * 将会通知后继节点，使后继节点的线程得以运行。
         *
         * CANCELLED:值为1，由于在同步队列中等待的
         * 线程等待超时或者被中断，需要从同步队列中取消等待，
         * 节点进入该状态将不会变化
         *
         * CONDITION: 值为-2，节点在等待队列中，
         * 节点线程等待在Condition上，当其他线程
         * 对Condition调用了singal方法后，该节点
         * 将会从等待队列中转移到同步队列中，加入到
         * 对同步状态的获取中
         *
         * PROPAGATE: 值为-3，表示下一次共享模式同步
         * 状态获取将会无条件地传播下去
         *
         * INITIAL: 初始状态值为0
         *
         * value 按数值排列以简化使用
         * 非负数 意味着这个节点不需要通知，所以，大部分代码里面不需要检查一些特殊值(只要判断是否 >0 就行)
         * 这个字段初始化为0，使用 CAS 进行更新
         * 或者，无条件写（就是赋值，无条件赋值，在赋值的时候不应该依赖当前值）
         */
        volatile int waitStatus;
        /**
        * 链接到前驱节点，当前节点/线程依赖它来检查 waitStatus 。
        * 在入同步队列时被设置(waitStatus)，并且仅在移除同步队列时才归零
        * （为了GC的目的）。 此外，在取消(出队列)状态为（CANCELLED）前驱节点时，我们使用简单的循环找到未取消的节点 
        * 这将始终存在，因为头节点从未被取消：节点仅作为成功获取的结果而变为头。
        * 被取消的线程永远不会成功获取，并且线程只取消自身，
        * 而不是任何其他节点。
        */
        volatile Node prev;
        /**
        * 链接到后续节点，当前节点/线程释放时释放。
        * 在入同步队列期间分配，在绕过取消的前驱节
        * 点时调整，并在出同步队列时取消（为了GC的目的）。
        * enq操作不会分配前驱节点的next字段，直到附加之后，
        * 因此看到一个为null的next字段不一定意味着该节点在
        * 队列的末尾。 但是，如果next字段显示为null,我们
        * 可以从尾部扫描prev，仔细检查。 被取消的节点的next字段
        * 被设置为指向节点本身而不是null，以使isOnSyncQueue更
        * 方便操作。调用isOnSyncQueue时，如果节点（始终
        * 是放置在条件队列上的节点）正等待在同步队列上重新获取，则返回true。
        **/
        volatile Node next;
        /**
         * 这个节点代表的线程，入队的时候初始化，出队的时候设置null
         * construction and nulled out after use.
         */
        volatile Thread thread;
         /**
        * 将此节点入列的线程。在构造方法里初始化，使用后清零。
        * 链接到下一个节点等待条件，或特殊值SHARED。
        * 因为条件队列只有在保持在独占模式时才被访问，
        * 所以我们只需要一个简单的链接队列来保存节点，
        * 同时等待条件。 然后将它们转移到队列中以重新获取。
        * 并且因为条件只能是排它的，我们通过使用特殊的
        * 值来指示共享模式来保存一个字段。
        */
        Node nextWaiter;
        /**
         * 如果节点在共享模式下等待，则返回true
         */
        final boolean isShared() {
            return nextWaiter == SHARED;
        }
        /**
         * 返回上一个节点，如果为null，
         * 则抛出NullPointerException。当前驱节点不为null时使用。
         **/
        final Node predecessor() throws NullPointerException {
            Node p = prev;
            if (p == null)
                throw new NullPointerException();
            else
                return p;
        }
```    

每个节点就是线程的代表，因此里面有 `Thread` 对象，`waitStatus` 这个字段代表当前线程所处的状态，只能是定义的几个静态常量值里面的。    

在来看 `AbstractQueuedSynchronizer` 里面有什么东西     
```java
    /**
     * Head of the wait queue, lazily initialized. Except for
     * initialization, it is modified only via method setHead. Note:
     * If head exists, its waitStatus is guaranteed not to be
     * CANCELLED.
     */
    private transient volatile Node head;
    /**
     * Tail of the wait queue, lazily initialized. Modified only via
     * method enq to add new wait node.
     */
    private transient volatile Node tail;
    /**
     * The synchronization state.
     */
    private volatile int state;
```    

维护了链表的头节点和尾节点，还有一个状态值 `state`, 注意使用了 `volatile`, 保证的该变量的可见性。    
还有几个用来使用 `CAS` 操作相关的成员，这里就不展示了    

好了，继续看之前的 `compareAndSetState(0, 1)`    
这个方法就是使用 `CAS` 操作，当 `state` 的值为0的时候，修改为1（在多线程的情况下，只会有一个成功，其他的失败），    
如果这个方法调用成功，那就是获取到锁了，调用 `setExclusiveOwnerThread` 方法，设置当前获取锁的线程对象，这里是 `AbstractQueuedSynchronizer` 的方法，`AbstractQueuedSynchronizer` 内部维护当前获取到临界资源的线程对象    
这就是获取到锁的情况，很简单，那么如果 `compareAndSetState` 不成功，代表锁获取失败。    
后面就调用 `acquire(1)`    
这个 `acquire` 方法也是 `AbstractQueuedSynchronizer` 里面的方法    
```java
public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```    
可以看到 调用了 `tryAcquire`, `acquireQueued`, `addWaiter`, `selfInterrupt`,这些方法。    
都是 `AbstractQueuedSynchronizer` 内部定义的。    
其中 `tryAcquire` 是定义好，专门给继承者重写的，`AbstractQueuedSynchronizer` 本身没有实现它，只是抛异常    
```java
protected boolean tryAcquire(int arg) {
        throw new UnsupportedOperationException();
    }
```    

我们可以找到，实现是在非公平锁那里实现的 
```java
protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
```    
它又调用了父类 `Sync` 的 `nonfairTryAcquire`    
跟进去看一下    
```java
final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```    

看逻辑，获取状态值，如果是0，代表还没有被获取，因此就再次调用 `compareAndSetState`,尝试获取锁，如果获取成功，那么返回 `true`，    
这也是获取到锁的情况，获取锁的逻辑到这里就结束了    
如果状态值不等于0，判断之前获取到锁的线程是不是当前线程，如果是，状态值增加，返回 `true` ，这也是获取到锁的情况，结束，否则，返回 `false`,进行后面的步骤（再次尝试，进入等待等。。）    
看到这里，就知道为什么叫可重入锁了，也看到可重入次数就是 `Integer.max`。    
那么继续，看如果返回了 `false` 会发生什么 

还是回到 `AbstractQueuedSynchronizer` 的 `acquire` 方法    
```java
public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```    

理一下逻辑，`tryAcquire` 是非公平锁实现的方法，调用了 `Sync` 里面的 `nonfairTryAcquire` 方法，获取成功了返回 `true`，否则失败`false`    
这里可以看到如果返回 `true` 后面就木有了，返回 `false` 的时候，继续 `acquireQueued(addWaiter(Node.EXCLUSIVE), arg))`    
我们一个一个看，先看 `addWaiter` 方法    
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
`new` 一个 `Node` 节点，使用独占模式    
```java
Node() { // Used to establish initial head or SHARED marker
        }
        Node(Thread thread, Node mode) { // Used by addWaiter
            this.nextWaiter = mode;
            this.thread = thread;
        }
        Node(Thread thread, int waitStatus) { // Used by Condition
            this.waitStatus = waitStatus;
            this.thread = thread;
        }
```    

如果 尾节点为不为 `null`，代表双向链表已经初始化过了，尝试设置当前新建的节点为尾节点 `compareAndSetTail`，(尾节点设置要用CAS操作)设置成功后，加入链表中返回这个节点    
如果失败（多个抢锁失败的线程同时过来了），或者尾节点为 `null` 了 调用 `enq` 方法，然后返回这个节点。    
看下 `enq`方法的实现    
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
这是一个私有方法。在一个for循环中，一直操作，知道成功    
如果尾节点为 `null` 了，代表链表还没有初始化，那么新建一个头节点，尾节点跟头节点指向同一个节点（正常情况下，头节点代表获取到锁的线程，设置头节点不需要CAS操作，但是这里链表还没初始化，还没有获取锁的线程。因此得使用CAS）    
链表初始化好了后继续循环    
设置当前节点为尾节点，加入链表，使用 `CAS` 加上 循环，保证成功，成功后返回当前节点    
为什么能一定成功？如果是多个线程同时操作，那么可能有一个成功，其他失败，成功的就返回了，其他失败的继续尝试，每次都有一个成功，到最后，每个都可以操作完，因此 `CAS` 操作很多时候都是跟死循环在一起使用的。    

所以简单的一句话来说明 `addWaiter` 方法的作用，那就是，新建一个独占节点代表当前线程，加入到双向链表的尾部，如果链表还没初始化，则先初始化    
这里注意，初始化的头节点是 waitStatus 值为0 ，不代表任何线程的节点，这里注意一下。
还发现一个问题没有 `addWaiter` 里面 
```java
Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }

```    
即便是没有这段，也是一样的，加上这段是为了应对大部分情况，因为链表只用初始化一次，大部分时候 `tail` 的指向不会为 `null` 的，这是题外话    
继续看    
```java

public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```    
后面就是 `acquireQueued` 方法了    
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

这里的 failed 代表的是失败标记，interrupted 代表的是中断标记    
可以看到，这里又是一个死循环，在循环里面处理    
获取当前线程的前一个节点，如果前一个节点就是头节点，那么就再次尝试获取锁 `tryAcquire`，这个方法前面已经说过了，arg 都是 1    
如果这里获取成功了，那么设置本节点为头节点（默认情况下，头节点代表已经获取锁的节点，但是也不是绝对，链表是延时加载的，如果每次获取锁都没有人竞争或者等到，这个链表压根就会初始化，只有有人需要等待的时候才会初始化）    
没有获取成功就是进入 `shouldParkAfterFailedAcquire` 方法了    
```java
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
             * waitStatus must be 0 or PROPAGATE. Indicate that we
             * need a signal, but don't park yet. Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
```    
这段逻辑就是当前一个节点的等待状态值`waitStatus` 为 `Node.SIGNAL` 的时候，本节点需要等待（等待前一个节点激活），如果是大于0（代表取消）,就从链表里面删除这些（while循环）节点，直到遇到一个不大于0的节点为止，
如果 等于0，那或者 `PROPAGATE`,代表，我们需要信号，但是，不要现在阻塞我，我阻塞之前还要最后再来尝试一次，设置前一个节点的等待状态为 `SIGNAL`.
可以看到，只有前一个节点的等待状态为 `SIGNAL` 的时候，才返回 true，其他情况 都是 false    
而这个方法外面，又是一个循环，通常如果这个方法返回false的时候，还是会再次进入到这里，可以看到最终，除非本线程拿到锁，否则还是得返回true，无非就是多有几次获取锁的机会（注意，这里不是尝试获取锁了，而是有尝试获取锁的机会，除非在这几次循环过程中，锁被释放掉了，而且本线程排在队列前面，并且没有插队的线程来抢锁，这个时候，本线程在多循环几次才可能抢到锁）    
这个方法返回true，代表本线程要阻塞了 ，后面的方法 `parkAndCheckInterrupt` 就是阻塞当前线程，并检查中断标记    
```java
private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }
```    
` LockSupport.park(this)` 这个就是阻塞当前线程    
`return Thread.interrupted()` 这行，就是，当本线程被激活后，返回本线程是否被设置了中断标记    

注意的是 ` LockSupport.park(this)` 阻塞线程，可以通过线程中断或者 `unpark` 唤醒    
返回是是否中断 `Thread.interrupted()` 判断中断标记，并且清除标记。    
所以在后面如果判断是中断，线程自己又设置了中断标记    
```java
static void selfInterrupt() {
        Thread.currentThread().interrupt();
    }
```    

`lock` 方法到这里就说完了，大部分时候，都是拿到锁，或者阻塞。    

### lockInterruptibly 方法    
这个方法是，响应中断标记的加锁，会抛中断异常    
```java
public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }
```    

直接调用 `AQS` 里面的方法    
```java
public final void acquireInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (!tryAcquire(arg))
            doAcquireInterruptibly(arg);
    }
```    

如果这个时候，线程已经被中断了，直接抛异常，这里清除中断标记    
如果没有中断，尝试获取锁，获取成功，返回，失败，就调用 `doAcquireInterruptibly` 方法    
```java
private void doAcquireInterruptibly(int arg)
        throws InterruptedException {
        final Node node = addWaiter(Node.EXCLUSIVE);
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```    
这个方法跟 `acquireQueued(addWaiter(Node.EXCLUSIVE), arg)`  类似。    
先在链表尾部添加独占节点。
然后尝试获取锁，阻塞，不同的是，这里遇到中断标记，抛出中断异常，而不是设置中断标记，这里就需要注意了，当抛异常出去的时候，`failed` 是为 `false` 的.。    
之前上面是没有说这个的，因为上面的方法，即使被中断了，也出不去循环的，只有获取锁的时候，也只是设置了一下中断标记，`failed` 始终是 `false` 的，但是这里，如果异常出去，就会有 `finally` 里面的逻辑了    
看一下 `cancelAcquire` 这个方法     
```java
private void cancelAcquire(Node node) {
        if (node == null)
            return;
        node.thread = null;
        // 跳过前置节点状态大于0的节点（出队列）
        Node pred = node.prev;
        while (pred.waitStatus > 0)
            node.prev = pred = pred.prev;
        // pred 是 node 的前置节点，predNext 是 pred.next ，那么显然是 node 节点
        Node predNext = pred.next;
        // 设置 node 节点的等待状态为 取消
        node.waitStatus = Node.CANCELLED;
        // 如果是尾节点，那就删掉，前置节点设置为尾节点
        if (node == tail && compareAndSetTail(node, pred)) {
            compareAndSetNext(pred, predNext, null);
        } else {
            // node 不是尾节点，或者设置pred为尾节点的时候失败了（有其他节点又加上来了）
            int ws;
            // 如果 pred 不是头节点 并且 ( pred的状态为 SIGNAL, 或者 状态小于等于0并且成功设置为 SIGNAL) 并且 pred 前置节点代表的线程不为null，那就删除当前节点，连接前置节点和后置节点
            // 这里的条件有点复杂，简单点说，就是看前置节点被设置为是否通知后面节点的状态没，如果设置了，或者如果没有设置，但是现在成功设置了，删除节点，
            // 否则，现在就尝试激活当前节点的后置节点,看 unparkSuccessor 方法
            if (pred != head &&
                ((ws = pred.waitStatus) == Node.SIGNAL ||
                 (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
                pred.thread != null) {
                Node next = node.next;
                if (next != null && next.waitStatus <= 0)
                    compareAndSetNext(pred, predNext, next);
            } else {
                // 如果pred为头节点，或者看上面的条件，就激活node的下一个节点
                unparkSuccessor(node);
            }
            node.next = node; // help GC
        }
    }
```    

上面已经有注释了，继续看看 `unparkSuccessor`    
```java
private void unparkSuccessor(Node node) {
        /*
         * 如果本节点的状态值 < 0 ，则设置为0，cas操作失败了也没关系，可能状态被等待的线程修改了
         * 修改的原因是，当前节点不需要等待锁资源，将要唤醒下一个节点
         */
        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);
        /*
         * 找到下一个节点，如果不为null，并且没有取消（状态值），则下一个节点就是继承人
         * 如果不满足上面的条件，就从尾节点开始，向前寻找(到当前节点的位置)未取消的节点(一直寻找，最后满足的那个)
         */
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        // 如果找到继承人，唤醒线程
        if (s != null)
            LockSupport.unpark(s.thread);
    }
```    

注释也已经有了，注意的是从`cancelAcquire` 方法过来的节点，等待状态值是取消，也就是 > 0 的    
参见上面获取锁的逻辑， 阻塞是在循环里面的，这里被激活了，也只是循环一次后，激活阻塞（大部分时候）    
其实，这里主要保证的是，如果当前节点被取消了，至少要保证它后面的节点会被它前面的节点激活，或者立即就激活一次    

### tryLock    
在来看看 `tryLock` ，尝试获取锁，获取成功返回true，获取失败返回false，直截了当    
```java
public boolean tryLock() {
        return sync.nonfairTryAcquire(1);
    }
```    
 
调用的时候 `Sync` 里面的方法    
```java
final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```    

这个方法就是 `lock` 里面调用过的方法，成功就返回true，失败就是返回 false

### tryLock(long timeout, TimeUnit unit)    
限时等待的 `tryLock`    
```java
public boolean tryLock(long timeout, TimeUnit unit)
            throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(timeout));
    }
```    
直接调用 `AQS` 的方法    
```java
public final boolean tryAcquireNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        return tryAcquire(arg) ||
            doAcquireNanos(arg, nanosTimeout);
    }
```    
同样是响应中断的    
`tryAcquire` 就不说了，调用子类的方法    
看看 `doAcquireNanos`    
```java
private boolean doAcquireNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        // 如果已经超时了，返回 false
        if (nanosTimeout <= 0L)
            return false;
        // 获取 最后期限
        final long deadline = System.nanoTime() + nanosTimeout;
        // 提交节点到链表最后
        final Node node = addWaiter(Node.EXCLUSIVE);
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                // 如果前置节点是头节点，尝试获取获取锁，获取成功返回true
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return true;
                }
                nanosTimeout = deadline - System.nanoTime();
                // 如果超时了，返回false
                if (nanosTimeout <= 0L)
                    return false;
                // 判断是否需要阻塞 如果离 最后期限只剩1秒了，就不需要阻塞了，一直循环就行了
                if (shouldParkAfterFailedAcquire(p, node) &&
                    nanosTimeout > spinForTimeoutThreshold)
                    // 限时阻塞, 阻塞可能是限时等待结束，或者前置节点激活
                    LockSupport.parkNanos(this, nanosTimeout);
                // 中断标识判断
                if (Thread.interrupted())
                    throw new InterruptedException();
            }
        } finally {
            // 中断出去的逻辑
            if (failed)
                cancelAcquire(node);
        }
    }
```    

可以看到，阻塞还是使用的 `LockSupport` 阻塞工具类    

### unlock    
看一下是否锁的逻辑    
```java
public void unlock() {
        sync.release(1);
    }
```    

调用 `AQS` 里面的逻辑    
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
尝试释放，释放成功，判断是否需要激活后面的节点，如果需要就激活    
`tryRelease` 由子类 `Sync` 实现

```java
protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }
```    
当 `AQS` 里面的 `state` 值为0，才算当前线程释放锁    

### newCondition    
最后看一个 `newCondition`    
```java
public Condition newCondition() {
        return sync.newCondition();
    }
```    

调用  `Sync` 里面的方法    
```java
final ConditionObject newCondition() {
            return new ConditionObject();
        }
```    

这个 `ConditionObject` 对象是 `AQS` 里面定义的,实现 `Condition` 接口    
```java
public interface Condition {
    /**
     * 等待信号量，响应中断
     */
    void await() throws InterruptedException;

    /**
     * 等待信号量，不响应中断
     */
    void awaitUninterruptibly();

    /**
     * 限时等待
     */
    long awaitNanos(long nanosTimeout) throws InterruptedException;

    /**
     * 限时等待
     */
    boolean await(long time, TimeUnit unit) throws InterruptedException;

    /**
     * 限时等待
     */
    boolean awaitUntil(Date deadline) throws InterruptedException;

    /**
     * 激活一个阻塞线程
     */
    void signal();

    /**
     * 激活所有阻塞线程
     */
    void signalAll();
}
```    

用法就是 `new` 一个 `Condition` 出来，获取锁后可以调用 `await` （注意，不要跟 wait 弄混了）。`signal` 方法也需要持有锁的线程才能调用。    
从 `await` 跟一下代码    
`ReentrantLock` 里面的 `newCondition` 方法，直接调用内部类 `Sync`
```java
public Condition newCondition() {
        return sync.newCondition();
    }
```    
`Sync`    
```java
final ConditionObject newCondition() {
            return new ConditionObject();
        }
```    
这个 `ConditionObject ` 是 `AQS` 内部类对 `Condition` 接口的实现    
然后具体跟一下 `await` 方法    
```java
/**
         * 在条件上等待
         * 1. 如果当前线程已经中断，则抛 InterruptedException 异常
         * 2. 保存当前 getState 状态
         * 3. 释放 state，如果失败，就抛 IllegalMonitorStateException。（当前线程必须持有锁）
         * 4. 阻塞，直到被激活（signal或者中断）
         * 5. 重新尝试获取之前持有的锁（可能继续等待）
         * 6. 如果在步骤4，阻塞的时候中断了，抛 InterruptedException
         * Implements interruptible condition wait.
         * <ol>
         * <li> If current thread is interrupted, throw InterruptedException.
         * <li> Save lock state returned by {@link #getState}.
         * <li> Invoke {@link #release} with saved state as argument,
         * throwing IllegalMonitorStateException if it fails.
         * <li> Block until signalled or interrupted.
         * <li> Reacquire by invoking specialized version of
         * {@link #acquire} with saved state as argument.
         * <li> If interrupted while blocked in step 4, throw InterruptedException.
         * </ol>
         */
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

代码上面已经有注释了，简单说一下，这里涉及到  节点 `Node` 的条件同步队列，再看一下 `Node` 的定义
```java
/**
        * 链接到前驱节点，当前节点/线程依赖它来检查 waitStatus 。
        * 在入同步队列时被设置(waitStatus)，并且仅在移除同步队列时才归零
        * （为了GC的目的）。 此外，在取消(出队列)状态为（CANCELLED）前驱节点时，我们使用简单的循环找到未取消的节点 
        * 这将始终存在，因为头节点从未被取消：节点仅作为成功获取的结果而变为头。
        * 被取消的线程永远不会成功获取，并且线程只取消自身，
        * 而不是任何其他节点。
        */
        volatile Node prev;
        /**
        * 链接到后续节点，当前节点/线程释放时释放。
        * 在入同步队列期间分配，在绕过取消的前驱节
        * 点时调整，并在出同步队列时取消（为了GC的目的）。
        * enq操作不会分配前驱节点的next字段，直到附加之后，
        * 因此看到一个为null的next字段不一定意味着该节点在
        * 队列的末尾。 但是，如果next字段显示为null,我们
        * 可以从尾部扫描prev，仔细检查。 被取消的节点的next字段
        * 被设置为指向节点本身而不是null，以使isOnSyncQueue更
        * 方便操作。调用isOnSyncQueue时，如果节点（始终
        * 是放置在条件队列上的节点）正等待在同步队列上重新获取，则返回true。
        **/
        volatile Node next;
        /**
         * 这个节点代表的线程，入队的时候初始化，出队的时候设置null
         * construction and nulled out after use.
         */
        volatile Thread thread;
         /**
        * 将此节点入列的线程。在构造方法里初始化，使用后清零。
        * 链接到下一个节点等待条件，或特殊值SHARED。
        * 因为条件队列只有在保持在独占模式时才被访问，
        * 所以我们只需要一个简单的链接队列来保存节点，
        * 同时等待条件。 然后将它们转移到队列中以重新获取。
        * 并且因为条件只能是排它的，我们通过使用特殊的
        * 值来指示共享模式来保存一个字段。
        */
        Node nextWaiter;
```    

在锁的实现里面，我们看到了同步队列，是一个双向链表，使用了 `prev` 和 `next` 。    
条件等待队列是一个单向链表,使用了 `nextWaiter` 字段    

看一下 `addConditionWaiter` 方法    
```java
/**
         * 添加一个新的等待节点到条件等待队列队尾
         * @return its new wait node
         */
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
 
`firstWaiter `, `lastWaiter` 是 `ConditionObject` 里面的字段    
```java
public class ConditionObject implements Condition, java.io.Serializable {
        private static final long serialVersionUID = 1173984872572414699L;
        /** First node of condition queue. */
        private transient Node firstWaiter;
        /** Last node of condition queue. */
        private transient Node lastWaiter;
```    

重点关注 `Node.CONDITION` 和 `nextWaiter`    
这里可以看到节点的等待状态是 `CONDITION`,进的队列是 消息等待队列    

进入队列后，释放锁    
```java
/**
     * 释放本节点持有的 state
     * 如果释放失败，就标记这个节点为取消状态
     */
    final int fullyRelease(Node node) {
        boolean failed = true;
        try {
            int savedState = getState();
            if (release(savedState)) {
                failed = false;
                return savedState;
            } else {
                throw new IllegalMonitorStateException();
            }
        } finally {
            if (failed)
                node.waitStatus = Node.CANCELLED;
        }
    }
```    

其中 `release` 方法就是正常的锁的释放流程，上面已经看过了    
然后就是一个循环，判断节点释放在同步队列（双向链表，正常情况是不在的，当其他线程调用了 signal 后才会放入到同步队列中）。循环内部就是阻塞了，阻塞可以被其他线程唤醒或者中断唤醒    
当出了循环后，就是重新尝试获取锁（这里正常的锁获取流程，也会阻塞等待），和中断处理    
`acquireQueued` 之前看过，就不分析了。    
`unlinkCancelledWaiters` 这个方法就是清理条件队列的无效节点（CANNELED 状态）    
```java
/**
         * 从条件队列清除取消的等待节点，这个方法只应该被持有锁的线程调用
         */
        private void unlinkCancelledWaiters() {
            Node t = firstWaiter;
            Node trail = null;
            while (t != null) {
                Node next = t.nextWaiter;
                if (t.waitStatus != Node.CONDITION) {
                    t.nextWaiter = null;
                    if (trail == null)
                        firstWaiter = next;
                    else
                        trail.nextWaiter = next;
                    if (next == null)
                        lastWaiter = trail;
                }
                else
                    trail = t;
                t = next;
            }
        }
```    

`reportInterruptAfterWait` 方法是中断处理
```java
/**
         * 根据中断标记，抛异常，或者设置中断状态
         * Throws InterruptedException, reinterrupts current thread, or
         * does nothing, depending on mode.
         */
        private void reportInterruptAfterWait(int interruptMode)
            throws InterruptedException {
            if (interruptMode == THROW_IE)
                throw new InterruptedException();
            else if (interruptMode == REINTERRUPT)
                selfInterrupt();
        }
```    

其他的几个 `await` 方法就不一一细说了。    
```java
/**
         * 在条件上等待 nanosTimeout 长时间
         * 基本跟上面一样
         * Implements timed condition wait.
         * <ol>
         * <li> If current thread is interrupted, throw InterruptedException.
         * <li> Save lock state returned by {@link #getState}.
         * <li> Invoke {@link #release} with saved state as argument,
         * throwing IllegalMonitorStateException if it fails.
         * <li> Block until signalled, interrupted, or timed out.
         * <li> Reacquire by invoking specialized version of
         * {@link #acquire} with saved state as argument.
         * <li> If interrupted while blocked in step 4, throw InterruptedException.
         * </ol>
         */
        public final long awaitNanos(long nanosTimeout)
                throws InterruptedException {
            // 如果中断了，就抛中断异常
            if (Thread.interrupted())
                throw new InterruptedException();
            // 添加一个节点到条件队列尾部
            Node node = addConditionWaiter();
            // 释放锁资源（如果没持有锁，会抛异常出来）
            int savedState = fullyRelease(node);
            // 阻塞的截至时间
            final long deadline = System.nanoTime() + nanosTimeout;
            int interruptMode = 0;
            // 循环判断，如果当前节点不在同步队列中（当调用 signal 方法的时候，会将节点放入同步队列）
            while (!isOnSyncQueue(node)) {
                // 超时，跳出循环
                if (nanosTimeout <= 0L) {
                    // 设置节点状态为取消，如果设置成功，那就是超时，如果设置失败（可能这个节点被激活了，超时就是 false）
                    transferAfterCancelledWait(node);
                    break;
                }
                // 如果要阻塞的时间大于 1s，继续限时阻塞（如果小于1s了，就不继续阻塞了，循环直到超时，获取被唤醒，很多地方都是这样的设计）
                if (nanosTimeout >= spinForTimeoutThreshold)
                    LockSupport.parkNanos(this, nanosTimeout);
                // 如果有中断，跳出循环
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
                // 计算离超时时间点的间隔
                nanosTimeout = deadline - System.nanoTime();
            }
            // 从上面出来后，尝试获取锁，这里还是会触发等待锁的阻塞（如果要等的话），获取成功，处理了继续判断中断状态
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            // 如果 当前节点在条件队列上后后续节点，那么清除一遍无效节点
            if (node.nextWaiter != null)
                unlinkCancelledWaiters();
            // 如果有中断状态，判断是否要抛异常
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
            return deadline - System.nanoTime();
        }

/**
         * 在条件上等待，直到给定时间点
         * 基本同上
         * Implements absolute timed condition wait.
         * <ol>
         * <li> If current thread is interrupted, throw InterruptedException.
         * <li> Save lock state returned by {@link #getState}.
         * <li> Invoke {@link #release} with saved state as argument,
         * throwing IllegalMonitorStateException if it fails.
         * <li> Block until signalled, interrupted, or timed out.
         * <li> Reacquire by invoking specialized version of
         * {@link #acquire} with saved state as argument.
         * <li> If interrupted while blocked in step 4, throw InterruptedException.
         * <li> If timed out while blocked in step 4, return false, else true.
         * </ol>
         */
        public final boolean awaitUntil(Date deadline)
                throws InterruptedException {
            // 阻塞的截至时间
            long abstime = deadline.getTime();
            // 如果中断了，就抛中断异常
            if (Thread.interrupted())
                throw new InterruptedException();
            // 添加一个节点到条件队列尾部
            Node node = addConditionWaiter();
            // 释放锁资源（如果没持有锁，会抛异常出来）
            int savedState = fullyRelease(node);
            boolean timedout = false;
            int interruptMode = 0;
            // 循环判断，如果当前节点不在同步队列中（当调用 signal 方法的时候，会将节点放入同步队列）
            while (!isOnSyncQueue(node)) {
                // 超时，跳出循环
                if (System.currentTimeMillis() > abstime) {
                    // 设置节点状态为取消，如果设置成功，那就是超时，如果设置失败（可能这个节点被激活了，超时就是 false）
                    timedout = transferAfterCancelledWait(node);
                    break;
                }
                // 阻塞
                LockSupport.parkUntil(this, abstime);
                // 判断中断，如果有，跳出循环
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            // 从上面出来后，尝试获取锁，这里还是会触发等待锁的阻塞（如果要等的话），获取成功，处理了继续判断中断状态
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            // 如果 当前节点在条件队列上后后续节点，那么清除一遍无效节点
            if (node.nextWaiter != null)
                unlinkCancelledWaiters();
            // 如果有中断状态，判断是否要抛异常
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
            // 如果超时，返回false
            return !timedout;
        }
        /**
         * 在条件上限时等待
         * 基本同上
         * Implements timed condition wait.
         * <ol>
         * <li> If current thread is interrupted, throw InterruptedException.
         * <li> Save lock state returned by {@link #getState}.
         * <li> Invoke {@link #release} with saved state as argument,
         * throwing IllegalMonitorStateException if it fails.
         * <li> Block until signalled, interrupted, or timed out.
         * <li> Reacquire by invoking specialized version of
         * {@link #acquire} with saved state as argument.
         * <li> If interrupted while blocked in step 4, throw InterruptedException.
         * <li> If timed out while blocked in step 4, return false, else true.
         * </ol>
         */
        public final boolean await(long time, TimeUnit unit)
                throws InterruptedException {
            long nanosTimeout = unit.toNanos(time);
            // 如果中断了，就抛中断异常
            if (Thread.interrupted())
                throw new InterruptedException();
            // 添加一个节点到条件队列尾部
            Node node = addConditionWaiter();
            // 释放锁资源（如果没持有锁，会抛异常出来）
            int savedState = fullyRelease(node);
            // 阻塞的截至时间
            final long deadline = System.nanoTime() + nanosTimeout;
            boolean timedout = false;
            int interruptMode = 0;
            // 循环判断，如果当前节点不在同步队列中（当调用 signal 方法的时候，会将节点放入同步队列）
            while (!isOnSyncQueue(node)) {
                if (nanosTimeout <= 0L) {
                    // 设置节点状态为取消，如果设置成功，那就是超时，如果设置失败（可能这个节点被激活了，超时就是 false）
                    timedout = transferAfterCancelledWait(node);
                    break;
                }
                // 如果要阻塞的时间大于 1s，继续限时阻塞（如果小于1s了，就不继续阻塞了，循环直到超时，获取被唤醒，很多地方都是这样的设计）
                if (nanosTimeout >= spinForTimeoutThreshold)
                    LockSupport.parkNanos(this, nanosTimeout);
                // 如果有中断，跳出循环
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
                // 计算离超时时间点的间隔
                nanosTimeout = deadline - System.nanoTime();
            }
            // 从上面出来后，尝试获取锁，这里还是会触发等待锁的阻塞（如果要等的话），获取成功，处理了继续判断中断状态
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            // 如果 当前节点在条件队列上后后续节点，那么清除一遍无效节点
            if (node.nextWaiter != null)
                unlinkCancelledWaiters();
            // 如果有中断状态，判断是否要抛异常
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
            // 如果超时，返回false
            return !timedout;
        }
```    

然后再看一下 `signal` 方法    
```java
/**
         * 激活一个等待节点
         * Moves the longest-waiting thread, if one exists, from the
         * wait queue for this condition to the wait queue for the
         * owning lock.
         *
         * @throws IllegalMonitorStateException if {@link #isHeldExclusively}
         * returns {@code false}
         */
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

可以看到当前线程必须持有锁    
```java
/**
         * 从参数node，开始，异常，并且转移直到一个没有取消的节点，或者nul
         * Removes and transfers nodes until hit non-cancelled one or
         * null. Split out from signal in part to encourage compilers
         * to inline the case of no waiters.
         * @param first (non-null) the first node on condition queue
         */
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

就是移除条件队列的第一个节点然后激活，如果激活失败，继续尝试下一个节点，直到激活成功，或者队列遍历完    
```java
/**
     * 这个方法的功能就是激活在条件上等待的线程（node里面的）
     * 将节点从条件队列，丢到同步队列中，成功返回true，失败返回false
     * Transfers a node from a condition queue onto sync queue.
     * Returns true if successful.
     * @param node the node
     * @return true if successfully transferred (else the node was
     * cancelled before signal)
     */
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

先尝试将等待状态修改为0（初始状态),然后入同步队列，告诉前一个前一个节点，本节点需要等待通知（获取锁）。如果设置通知失败，那先激活当前线程（执行到获取锁那里，阻塞或者获取）    

最后看 `signalAll`    
```java
/**
         * 激活所有节点
         * Moves all threads from the wait queue for this condition to
         * the wait queue for the owning lock.
         *
         * @throws IllegalMonitorStateException if {@link #isHeldExclusively}
         * returns {@code false}
         */
        public final void signalAll() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;
            if (first != null)
                doSignalAll(first);
        }
```    

```java
/**
         * 从first开始激活所有等待的节点
         * Removes and transfers all nodes.
         * @param first (non-null) the first node on condition queue
         */
        private void doSignalAll(Node first) {
            lastWaiter = firstWaiter = null;
            // 逻辑跟上面是一样的，不过就是激活所有节点，所有取消了对 transferForSignal 的结果判断
            do {
                Node next = first.nextWaiter;
                first.nextWaiter = null;
                transferForSignal(first);
                first = next;
            } while (first != null);
        }
```    
如果理解上激活单个节点，这里就好理解了，激活所有节点，所谓激活，就是将节点丢到同步队列里面去参与锁的竞争。    

上面还有几个限时等待，注意的是，如果超时后，就会将节点丢到同步队列参与锁的竞争，或者 `signal` 方法也可以触发    
非限时等待方法，只有 `signal` 才能激活    

## 总结    
通过分析，学到什么    
1. `AQS` 的内部结构，虽然还没完全了解，但是基本概念知道了，等后面再分析几个同步器，就可以对 `AQS` 进行总结了    
2. 可重入锁基本掌握，知道了锁并不是随机获取的，而是有顺序的，所谓非公平锁，那也只是在刚好释放锁的时候，下一个节点跟当前请求线程的竞争    
3. 条件，基本都是使用 `AQS` 内部的实现。知道了内部有同步队列和条件队列    

锁的顺序获取测试     
```java
public static final ReentrantLock lock = new ReentrantLock();
    public static void main(String[] args) {
        // 主线程获取锁
        lock.lock();
        new Thread(() -> {
            System.out.println("线程1 尝试获取锁");
            lock.lock();
            SleepUtil.sleepQuietly(2);//业务运行2s
            System.out.println("线程1 获取到锁");
            System.out.println("线程1 释放锁");
            lock.unlock();
        }).start();
        SleepUtil.sleepQuietly(2);//停2s，等线程运行到等锁那里
        new Thread(() -> {
            System.out.println("线程2 尝试获取锁");
            lock.lock();
            SleepUtil.sleepQuietly(2);//业务运行2s
            System.out.println("线程2 获取到锁");
            System.out.println("线程2 释放锁");
            lock.unlock();
        }).start();
        SleepUtil.sleepQuietly(2);//停2s，等线程运行到等锁那里
        new Thread(() -> {
            System.out.println("线程3 尝试获取锁");
            lock.lock();
            SleepUtil.sleepQuietly(2);//业务运行2s
            System.out.println("线程3 获取到锁");
            System.out.println("线程3 释放锁");
            lock.unlock();
        }).start();
        SleepUtil.sleepQuietly(2);//停2s，等线程运行到等锁那里
        // 主线程释放锁
        lock.unlock();
    }
```
运行结果（可以多次运行，但是输出是固定的）
```txt
线程1 尝试获取锁
线程2 尝试获取锁
线程3 尝试获取锁
线程1 获取到锁
线程1 释放锁
线程2 获取到锁
线程2 释放锁
线程3 获取到锁
线程3 释放锁
```    
