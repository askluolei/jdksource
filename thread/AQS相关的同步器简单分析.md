[TOC]   

# 基于 AQS 的同步器简单分析 

大部分同步器的实现都是基于 AQS 的。     
在了解了 AQS 的实现后，理解其他同步器的实现就比较轻松了。   
这里只是简单介绍一下。  
可重入锁 ReentrantLock 已经单独看过了，就不在分析了 

## Semaphore    
信号量（不知道是不是这样翻译）  
里面有一堆许可，每次可以请求一个或多个许可，当请求成功，内部许可数量减少相应许可    
如果内部许可不够，就请求失败（看调用的方法，是阻塞还是返回false）   
使用完要归还许可    

### acquire 
先从常用方法 acquire 开始分析   
```java
public void acquire() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}
``` 

这个 sync 是静态内部类, 基础自 AQS ，又公平和非公平实现（区别就是最新一个请求的线程是否有资格立即获取许可）
acquireSharedInterruptibly 这个方法是 AQS 里面的方法，获取共享资源，回调子类的 tryAcquireShared，来判断是否获取成功
又公平实现和非公平实现， Semaphore 只使用了共享相关的方法
```java
/**
 * 非公平实现，直接调用 Sync 的 nonfairTryAcquireShared
 */
protected int tryAcquireShared(int acquires) {
    return nonfairTryAcquireShared(acquires);
}

/**
* 公平实现获取
* 如果有节点排在前面，返回失败
* 再看许可数量，尝试获取
*/
protected int tryAcquireShared(int acquires) {
    for (;;) {
        if (hasQueuedPredecessors())
            return -1;
        int available = getState();
        int remaining = available - acquires;
        if (remaining < 0 ||
            compareAndSetState(available, remaining))
            return remaining;
    }
}
``` 

默认是非公平实现的，因此，关注一下非公平的  
```java
/**
 * 这个是非公平的尝试获取共享节点实现
 */
final int nonfairTryAcquireShared(int acquires) {
    for (;;) {
        int available = getState();
        int remaining = available - acquires;
        // 当前剩余的许可数量 < 请求的许可数量 返回 （负数就是失败）
        // 否则尝试修改state值（获取），成功就返回（非负数），失败就继续循环
        if (remaining < 0 ||
            compareAndSetState(available, remaining))
            return remaining;
    }
}
``` 

跟公平的实现就是少了前面的判断，公平就是得排队获取，    
AQS 的 state 存储的就是 Semaphore 当前拥有的许可数量    

然后简单过一下 acquire 的其他几个方法，基本差不多   
```java
public void acquireUninterruptibly() {
    sync.acquireShared(1);
}

public void acquire(int permits) throws InterruptedException {
    if (permits < 0) throw new IllegalArgumentException();
    sync.acquireSharedInterruptibly(permits);
}

public void acquireUninterruptibly(int permits) {
    if (permits < 0) throw new IllegalArgumentException();
    sync.acquireShared(permits);
}
``` 

这几个方法都是调用 AQS 里面的方法，而 AQS 会调用子类的实现 tryAcquireShared     

## tryAcquire   
这个方法就是尝试获取许可，获取失败返回false，成功返回true，不阻塞   
来看一下类似的方法  
```java
public boolean tryAcquire() {
    return sync.nonfairTryAcquireShared(1) >= 0;
}

public boolean tryAcquire(long timeout, TimeUnit unit)
    throws InterruptedException {
    return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
}

public boolean tryAcquire(int permits) {
    if (permits < 0) throw new IllegalArgumentException();
    return sync.nonfairTryAcquireShared(permits) >= 0;
}

public boolean tryAcquire(int permits, long timeout, TimeUnit unit)
    throws InterruptedException {
    if (permits < 0) throw new IllegalArgumentException();
    return sync.tryAcquireSharedNanos(permits, unit.toNanos(timeout));
}
``` 

感觉没啥好说的 调用 AQS ，然后 AQS 调用 tryAcquireShared 判断是否获取了，如果没有获取，阻塞，限时阻塞，中断判断都由 AQS 来维护  

## release  
释放许可    
```java
public void release() {
    sync.releaseShared(1);
}

public void release(int permits) {
    if (permits < 0) throw new IllegalArgumentException();
    sync.releaseShared(permits);
}
``` 

就是调用共享节点的 releaseShared ， 注意一点，就是共享节点，释放会激活下一个节点，获取会激活下一个共享节点。    
所以释放的许可不只一个的时候，首先获取到许可的是下一个等待节点（假设没有竞争），下一个节点获取到后，继续激活下一个共享节点(只是激活，能否获取许可还是看子类的 tryAcquireShared 实现)    

# CountDownLatch    
这个用法也比较简单，就是设置一个门闩，当有足够多的线程到达的时候（countDown），就激活所有在等待的线程（说的可能有问题） 
重点方法有构造，countDown， 和 await    
构造指定 门闩值，count，减少门闩，await 在门闩上等待，等门闩值减到0，激活所有 await 线程    

## 构造 
```java
public CountDownLatch(int count) {
    if (count < 0) throw new IllegalArgumentException("count < 0");
    this.sync = new Sync(count);
}

private static final class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = 4982264981922014374L;

    /**
        * 传进来的count 就是 state 的值
        */
    Sync(int count) {
        setState(count);
    }

    int getCount() {
        return getState();
    }

    /**
        * 下面的await 直接调用 AQS 的方法，AQS 的方法会调用这个方法来判断是否获取成功
        */
    protected int tryAcquireShared(int acquires) {
        /**
            * 当 state 的值为0 的时候是成功的，也就是 await 不会阻塞，直接放行
            */
        return (getState() == 0) ? 1 : -1;
    }

    /**
        * 下面的countDown 直接调用 AQS 的方法，AQS 的方法会调用这个方法来判断是否释放成功
        */
    protected boolean tryReleaseShared(int releases) {
        // Decrement count; signal when transition to zero
        // 这个是 countDown 方法调用后 AQS 回调子类的方法，判断是否要解除阻塞
        // 下面的逻辑很简单 如果 state 为 0 返回false，不是是话，state - 1，返回 state 是否为 0
        // 如果为0了，就代表 AQS 要做释放共享节点操作了（通知后续节点）
        for (;;) {
            int c = getState();
            if (c == 0)
                return false;
            int nextc = c-1;
            if (compareAndSetState(c, nextc))
                return nextc == 0;
        }
    }
}
```     

可以看到 AQS 的 state 里面存的就是构造指定的值  

## await    
看下 await 相关的方法   
```java
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}

public boolean await(long timeout, TimeUnit unit)
    throws InterruptedException {
    return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
}
``` 

可以看到，如果是直接阻塞，还是限时阻塞都是获取的共享节点，共享节点调用的是  
```java
protected int tryAcquireShared(int acquires) {
    /**
     * 当 state 的值为0 的时候是成功的，也就是 await 不会阻塞，直接放行
     */
    return (getState() == 0) ? 1 : -1;
}
``` 

可以看到，子类实现的，获取到共享节点的条件就是 state 的值为 0，所以 await 就是阻塞了    

## countDown    
```java
public void countDown() {
    sync.releaseShared(1);
}
``` 

这个方法调用的是 释放共享节点，能否释放看子类实现的 tryReleaseShared    
```java
protected boolean tryReleaseShared(int releases) {
    // Decrement count; signal when transition to zero
    // 这个是 countDown 方法调用后 AQS 回调子类的方法，判断是否要解除阻塞
    // 下面的逻辑很简单 如果 state 为 0 返回false，不是是话，state - 1，返回 state 是否为 0
    // 如果为0了，就代表 AQS 要做释放共享节点操作了（通知后续节点）
    for (;;) {
        int c = getState();
        if (c == 0)
            return false;
        int nextc = c-1;
        if (compareAndSetState(c, nextc))
            return nextc == 0;
    }
}
``` 
可以看到，调用一次 state 就 -1，等到 state = 0 的时候，就可以释放了，当释放的时候，会激活后面等待的共享节点 

# CyclicBarrier 
这个场景就是，指定一个值，当等待的线程达到这个值的时候，所有等待的线程激活。    
这个不是直接基于 AQS 的，内部使用了重入锁和条件 
重要的方法有构造，await，reset  

## 构造 
```java
public CyclicBarrier(int parties) {
    this(parties, null);
}

public CyclicBarrier(int parties, Runnable barrierAction) {
    if (parties <= 0) throw new IllegalArgumentException();
    this.parties = parties;
    this.count = parties;
    this.barrierCommand = barrierAction;
}
``` 

barrierAction 这个是用来指定当满足条件，激活的时候回调的一个任务    

## await    
```java
public int await() throws InterruptedException, BrokenBarrierException {
    try {
        return dowait(false, 0L);
    } catch (TimeoutException toe) {
        throw new Error(toe); // cannot happen
    }
}

public int await(long timeout, TimeUnit unit)
    throws InterruptedException,
            BrokenBarrierException,
            TimeoutException {
    return dowait(true, unit.toNanos(timeout));
}
``` 

await 方法是调用内部的 dowait 方法  
```java
/**
* 本质上很简单
* 使用重入锁和条件机制
* 线程在条件上等待，当等待的线程达到指定数量，就激活所有线程，然后进行下一轮
* 重点在于异常处理，当等待的线程中，有中断，或者使用限时等待，超时了，那么本同步器就处于一个中断（不可用状态）
* 需要调用 reset 来重置一下状态
*/
/**
* Main barrier code, covering the various policies.
*/
private int dowait(boolean timed, long nanos)
    throws InterruptedException, BrokenBarrierException,
            TimeoutException {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        final Generation g = generation;

        if (g.broken)
            throw new BrokenBarrierException();

        if (Thread.interrupted()) {
            // 如果调用 await 的线程中断了 那中断这个同步器
            breakBarrier();
            throw new InterruptedException();
        }

        int index = --count;
        if (index == 0) {  // tripped
            boolean ranAction = false;
            try {
                final Runnable command = barrierCommand;
                // 注意这里直接调用 run，也就是说再最后一个调用 await 的线程中调用的
                if (command != null)
                    command.run();
                // 如果不发生异常，就是true了，重置这个同步器
                ranAction = true;
                nextGeneration();
                return 0;
            } finally {
                // 如果上面 command 异常了，那中断这个同步器
                if (!ranAction)
                    breakBarrier();
            }
        }

        // 如果到这里，那就代表还没到激活线程的时候，本线程也要等待，直到激活，同步器中断，线程中断，超时
        // loop until tripped, broken, interrupted, or timed out
        for (;;) {
            try {
                // 没有超时条件，那就直接在条件上等待
                if (!timed)
                    trip.await();
                // 否则 限时等待
                else if (nanos > 0L)
                    nanos = trip.awaitNanos(nanos);
            } catch (InterruptedException ie) {
                // 这里处理线程中断
                if (g == generation && ! g.broken) {
                    // 如果同步器没有中断，就中断同步器，抛中断异常
                    breakBarrier();
                    throw ie;
                } else {
                    // 如果同步器已经中断了，设置线程中断，后面抛同步器中断异常
                    // We're about to finish waiting even if we had not
                    // been interrupted, so this interrupt is deemed to
                    // "belong" to subsequent execution.
                    Thread.currentThread().interrupt();
                }
            }

            // 同步器中断
            if (g.broken)
                throw new BrokenBarrierException();

            // 当调用了 nextGeneration 才为true，正常返回
            if (g != generation)
                return index;

            // 如果有超时标记，并且超时了
            if (timed && nanos <= 0L) {
                // 中断同步器，抛超时异常
                breakBarrier();
                throw new TimeoutException();
            }
        }
    } finally {
        lock.unlock();
    }
}
``` 

代码上有注释了，就不细讲了  
当本同步器因为一次，中断导致不可用后，可以通过 reset 方法恢复

# ReentrantReadWriteLock    
读写锁  
实现的逻辑是 读读不互斥，读写互斥，写写互斥 
源码里面的内容很多，看懂不复杂，还是基于 AQS， 内部有读锁 和 写锁   
读锁获取共享节点，写锁获取独占节点  

读写锁接口就两个方法，返回读锁和写锁    
```java
public interface ReadWriteLock {
    /**
     * Returns the lock used for reading.
     *
     * @return the lock used for reading
     */
    Lock readLock();

    /**
     * Returns the lock used for writing.
     *
     * @return the lock used for writing
     */
    Lock writeLock();
}
``` 

内部就是    
```java
/** 内部类实现的读锁 */
/** Inner class providing readlock */
private final ReentrantReadWriteLock.ReadLock readerLock;
/** 内部类实现的写锁 */
/** Inner class providing writelock */
private final ReentrantReadWriteLock.WriteLock writerLock;

/** 接口实现，返回内部的读写锁实现 */
public ReentrantReadWriteLock.WriteLock writeLock() { return writerLock; }
public ReentrantReadWriteLock.ReadLock  readLock()  { return readerLock; }
``` 

一般同步器大部分方法都是实现在一个继承 AQS 的静态内部类上，通常叫 Sync，如果还要区分公平，非公平的话，下面再有 FairSync 和 UnfairSync   
先看看 Sync 
```java
abstract static class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = 6317671515068378041L;

    /**
    * 下面这些变量，主要用来计算读锁，写锁的持有数
    * AQS 内部是一个 state int变量，使用这个变量记录
    * 一个int 32位，其中高 16位 为读锁数量，低16位为写锁数量
    */
    static final int SHARED_SHIFT   = 16;
    static final int SHARED_UNIT    = (1 << SHARED_SHIFT);// 2 的16次方
    static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;// 上面 -1，就是低16位全1
    static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;// 同上

    /** 返回读锁数量 */
    /** Returns the number of shared holds represented in count  */
    static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }

    /** 返回写锁数量 */
    /** Returns the number of exclusive holds represented in count  */
    static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }

    /**
        * A counter for per-thread read hold counts.
        * Maintained as a ThreadLocal; cached in cachedHoldCounter
        */
    static final class HoldCounter {
        int count = 0;
        // Use id, not reference, to avoid garbage retention
        final long tid = getThreadId(Thread.currentThread());
    }

    /**
    * ThreadLocal subclass. Easiest to explicitly define for sake
    * of deserialization mechanics.
    */
    static final class ThreadLocalHoldCounter
        extends ThreadLocal<HoldCounter> {
        public HoldCounter initialValue() {
            return new HoldCounter();
        }
    }

    /**
    * 线程变量，保存当前线程持有的读锁数量
    */
    private transient ThreadLocalHoldCounter readHolds;

    /**
    * 线程对象的缓存，减少线程对象的get
    */
    private transient HoldCounter cachedHoldCounter;

    /**
    * 第一个读锁线程，和第一个读锁线程的持有读锁的数量
    */
    private transient Thread firstReader = null;
    private transient int firstReaderHoldCount;

    Sync() {
        readHolds = new ThreadLocalHoldCounter();
        setState(getState()); // ensures visibility of readHolds
    }

    /**
        * 两个子类实现的方法
        */
    abstract boolean readerShouldBlock();

    abstract boolean writerShouldBlock();

    /**
        * 这个是写锁的释放方法
        * 当前线程持有写锁的数量为0的时候返回true
        */
    protected final boolean tryRelease(int releases) {
        if (!isHeldExclusively())
            throw new IllegalMonitorStateException();
        int nextc = getState() - releases;
        boolean free = exclusiveCount(nextc) == 0;
        if (free)
            setExclusiveOwnerThread(null);
        setState(nextc);
        return free;
    }

    /**
    * 这个是写锁获取的方法
    */
    protected final boolean tryAcquire(int acquires) {
        Thread current = Thread.currentThread();
        int c = getState();
        int w = exclusiveCount(c);
        // 如果当前有读锁或者写锁被持有了
        if (c != 0) {
            // (Note: if c != 0 and w == 0 then shared count != 0)
            // w == 0 代表有读锁，如果 != 0 ，代表有线程持有读锁，继续判断持有的线程是否为当前线程，如果还不是，返回false
            if (w == 0 || current != getExclusiveOwnerThread())
                return false;
            
            // 到这里来， w != 0 ，并且持有的线程就是自己，判断重入次数
            if (w + exclusiveCount(acquires) > MAX_COUNT)
                throw new Error("Maximum lock count exceeded");
            // Reentrant acquire
            // 增加锁持有数量
            setState(c + acquires);
            return true;
        }
        // 如果当前没有人持有，判断写锁是否需要阻塞，非公平不需要，公平则判断前面有没有等待的节点
        // 如果不需要阻塞，直接尝试获取，成功返回true，是否返回false
        // 这里的结果不是最终结果，父类上面还有尝试的机会，尝试失败，则阻塞
        if (writerShouldBlock() ||
            !compareAndSetState(c, c + acquires))
            return false;
        setExclusiveOwnerThread(current);
        return true;
    }

    /**
    * 尝试释放共享资源
    */
    protected final boolean tryReleaseShared(int unused) {
        Thread current = Thread.currentThread();
        // 建少线程持有的读锁数量
        if (firstReader == current) {
            // assert firstReaderHoldCount > 0;
            if (firstReaderHoldCount == 1)
                firstReader = null;
            else
                firstReaderHoldCount--;
        } else {
            HoldCounter rh = cachedHoldCounter;
            if (rh == null || rh.tid != getThreadId(current))
                rh = readHolds.get();
            int count = rh.count;
            if (count <= 1) {
                readHolds.remove();
                if (count <= 0)
                    throw unmatchedUnlockException();
            }
            --rh.count;
        }
        // 循环保证成功
        for (;;) {
            int c = getState();
            int nextc = c - SHARED_UNIT;
            // 这里才是真正修改读锁数量的地方，修改成功，判断读锁的数量是否为0，如果是，那就可以释放共享资源了
            if (compareAndSetState(c, nextc))
                // Releasing the read lock has no effect on readers,
                // but it may allow waiting writers to proceed if
                // both read and write locks are now free.
                return nextc == 0;
        }
    }

    private IllegalMonitorStateException unmatchedUnlockException() {
        return new IllegalMonitorStateException(
            "attempt to unlock read lock, not locked by current thread");
    }

    /**
        * 尝试获取共享锁，返回 < 0 ，获取失败
        */
    protected final int tryAcquireShared(int unused) {
        Thread current = Thread.currentThread();
        int c = getState();
        // 如果写锁被其他线程持有，获取失败
        if (exclusiveCount(c) != 0 &&
            getExclusiveOwnerThread() != current)
            return -1;
        // 获取当前读锁持有数量
        int r = sharedCount(c);
        // 非公平下，如果有写锁在第一个等待位置（第二个节点），读锁就应该阻塞
        // 这里如果不阻塞，并且读锁持有数量不超过最大值，尝试读锁加1，如果成功,那就是获取成功了，否则继续 fullTryAcquireShared
        if (!readerShouldBlock() &&
            r < MAX_COUNT &&
            // 尝试读锁加1 ，高16位标识读锁数量，SHARED_UNIT，就是高16位是 1，低16位是0
            compareAndSetState(c, c + SHARED_UNIT)) {
            if (r == 0) {
                // 代表本线程持有第一个读锁
                firstReader = current;
                firstReaderHoldCount = 1;
            } else if (firstReader == current) {
                // 如果持有数量 > 0 ，第一个持有线程还是本线程，则第一个线程持有锁的数量 ++
                firstReaderHoldCount++;
            } else {
                // 当前线程不为第一个持有读锁的线程
                // 线程变量记录当前线程持有的读锁的数量
                // cachedHoldCounter 字段用来减少 readHolds.get() 调用
                HoldCounter rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                    cachedHoldCounter = rh = readHolds.get();
                else if (rh.count == 0)
                    // 当前线程第一个获取读锁，设置线程变量
                    readHolds.set(rh);
                rh.count++;
            }
            // 获取成功
            return 1;
        }
        // 上面如果需要阻塞，或者获取失败
        return fullTryAcquireShared(current);
    }

    final int fullTryAcquireShared(Thread current) {
        HoldCounter rh = null;
        // 循环尝试
        for (;;) {
            int c = getState();
            // 如果有其他线程获取写锁，返回失败
            if (exclusiveCount(c) != 0) {
                if (getExclusiveOwnerThread() != current)
                    return -1;
                // else we hold the exclusive lock; blocking here
                // would cause deadlock.
            } else if (readerShouldBlock()) {
                // 当获取读锁需要被阻塞
                // Make sure we're not acquiring read lock reentrantly
                if (firstReader == current) {
                    // assert firstReaderHoldCount > 0;
                    // 第一个获取读锁的线程不管
                } else {
                    // 重点在这里。readerShouldBlock 影响
                    // 其他线程，如果该线程已经持有读锁了，继续后面的逻辑，没有持有，那就返回失败
                    if (rh == null) {
                        rh = cachedHoldCounter;
                        if (rh == null || rh.tid != getThreadId(current)) {
                            rh = readHolds.get();
                            if (rh.count == 0)
                                readHolds.remove();
                        }
                    }
                    // 其他新线程想要获取读锁，返回失败
                    if (rh.count == 0)
                        return -1;
                }
            }
            // 读锁最大数量检查
            if (sharedCount(c) == MAX_COUNT)
                throw new Error("Maximum lock count exceeded");
            // 读锁总数 + 1，如果失败，可能其他读锁操作了，外层循环，继续重新尝试
            if (compareAndSetState(c, c + SHARED_UNIT)) {
                // 如果读锁刚开始为0，则是第一个获取读锁的
                if (sharedCount(c) == 0) {
                    firstReader = current;
                    firstReaderHoldCount = 1;
                } else if (firstReader == current) {
                    // 不为0，如果还是第一个获取读锁的线程
                    firstReaderHoldCount++;
                } else {
                    // 否则就是其他线程获取锁了
                    if (rh == null)
                        rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current))
                        rh = readHolds.get();
                    else if (rh.count == 0)
                        readHolds.set(rh);
                    rh.count++;
                    cachedHoldCounter = rh; // cache for release
                }
                return 1;
            }
        }
    }

    /**
    * 尝试获取写锁
    */
    final boolean tryWriteLock() {
        Thread current = Thread.currentThread();
        int c = getState();
        if (c != 0) {
            // 有锁被获取
            int w = exclusiveCount(c);
            // 有读锁被获取，获取有其他线程持有写锁返回false
            if (w == 0 || current != getExclusiveOwnerThread())
                return false;
            if (w == MAX_COUNT)
                throw new Error("Maximum lock count exceeded");
        }
        // 尝试获取写锁，成功返回true，失败返回false
        if (!compareAndSetState(c, c + 1))
            return false;
        setExclusiveOwnerThread(current);
        return true;
    }

    /**
    * 尝试获取读锁
    */
    final boolean tryReadLock() {
        Thread current = Thread.currentThread();
        // 循环中保证有结果
        for (;;) {
            int c = getState();
            // 如果有写锁了，返回 false
            if (exclusiveCount(c) != 0 &&
                getExclusiveOwnerThread() != current)
                return false;
            int r = sharedCount(c);
            // 读锁达到最大数量
            if (r == MAX_COUNT)
                throw new Error("Maximum lock count exceeded");
            // 尝试获取读锁，如果 CAS 操作失败，可能是同时有其他读锁在修改，继续循环
            if (compareAndSetState(c, c + SHARED_UNIT)) {
                // 下面就是设置 是否是读锁的第一个线程，或者本线程是否是第一次获取读锁，设置数量
                if (r == 0) {
                    firstReader = current;
                    firstReaderHoldCount = 1;
                } else if (firstReader == current) {
                    firstReaderHoldCount++;
                } else {
                    HoldCounter rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current))
                        cachedHoldCounter = rh = readHolds.get();
                    else if (rh.count == 0)
                        readHolds.set(rh);
                    rh.count++;
                }
                return true;
            }
        }
    }

    /**
    * 本线程是否为写锁持有线程
    */
    protected final boolean isHeldExclusively() {
        // While we must in general read state before owner,
        // we don't need to do so to check if current thread is owner
        return getExclusiveOwnerThread() == Thread.currentThread();
    }

    // Methods relayed to outer class

    final ConditionObject newCondition() {
        return new ConditionObject();
    }

    final Thread getOwner() {
        // Must read state before owner to ensure memory consistency
        return ((exclusiveCount(getState()) == 0) ?
                null :
                getExclusiveOwnerThread());
    }

    final int getReadLockCount() {
        return sharedCount(getState());
    }

    final boolean isWriteLocked() {
        return exclusiveCount(getState()) != 0;
    }

    final int getWriteHoldCount() {
        return isHeldExclusively() ? exclusiveCount(getState()) : 0;
    }

    final int getReadHoldCount() {
        if (getReadLockCount() == 0)
            return 0;

        Thread current = Thread.currentThread();
        if (firstReader == current)
            return firstReaderHoldCount;

        HoldCounter rh = cachedHoldCounter;
        if (rh != null && rh.tid == getThreadId(current))
            return rh.count;

        int count = readHolds.get().count;
        if (count == 0) readHolds.remove();
        return count;
    }

    /**
        * Reconstitutes the instance from a stream (that is, deserializes it).
        */
    private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
        s.defaultReadObject();
        readHolds = new ThreadLocalHoldCounter();
        setState(0); // reset to unlocked state
    }

    final int getCount() { return getState(); }
}
```

上面有注释，写下自己的理解把
如果当前是写锁持有，那么后面不管是读还是写全部阻塞，这个很简单，独占    
如果当前是读锁持有，情况就复杂点，如果还是请求读锁，直接获取了，如果写锁请求，阻塞（正常情况下，他就在第二个节点上了，等待唤醒）    
但是，后面如果还有请求读锁，这个读锁是否可以获取到，就区分公平和非公平了
在非公平条件下，是写锁优先的，如果有写锁在等待了，读锁应该要等待（不代表要阻塞，这里持有资源的还是读锁），这个时候如果请求的线程已经获取读锁了，那么可以继续获取，如果之前
没有获取过读锁，那么是获取不到的，阻塞  
在公平条件下，先到先得，写锁一直要等待没有读锁被持有，才能获取锁。这可能会导致写锁长时间阻塞    
