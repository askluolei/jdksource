[TOC]   

# Queue 源码解析    
Queue 继承体系下，常用实现的分析    
Queue 接口下面，还有 Deque 和 BlockingQueue 和 BlockingDeque    

我们先弄清楚队列的方法的含义    
就说功能，方法太多，用的时候看说明就行了    
1. 从头获取并删除元素（如果为空抛异常）
2. 从头获取并删除元素（如果为空返回null）
3. 从头获取元素（如果为空抛异常）
4. 从头获取元素（如果为空返回null）
5. 从头添加元素（如果满了抛异常）
6. 从头添加元素（如果满了返回false）    

尾的操作跟上面类似。就不说了，具体用的时候看 api 的说明，完全记不住。

## Deque    
这个下面的实现类有 LinkedList ArrayDeque ConcurrentLinkedDeque  
其中 LinkedList 就不说了，我们看看其他两个  

### ArrayDeque 
看名字就知道，这是一个基于数组的双向队列实现    
ArrayDeque 的初始化容量一定是 2 的指数  
```java
public ArrayDeque(int numElements) {
    allocateElements(numElements);
}
private void allocateElements(int numElements) {
    int initialCapacity = MIN_INITIAL_CAPACITY;
    // Find the best power of two to hold elements.
    // Tests "<=" because arrays aren't kept full.
    if (numElements >= initialCapacity) {
        initialCapacity = numElements;
        initialCapacity |= (initialCapacity >>>  1);
        initialCapacity |= (initialCapacity >>>  2);
        initialCapacity |= (initialCapacity >>>  4);
        initialCapacity |= (initialCapacity >>>  8);
        initialCapacity |= (initialCapacity >>> 16);
        initialCapacity++;

        if (initialCapacity < 0)   // Too many elements, must back off
            initialCapacity >>>= 1;// Good luck allocating 2 ^ 30 elements
    }
    elements = new Object[initialCapacity];
}
```     

ArrayDeque 使用数组存储，但是，并不是 index = 0 的元素是头元素，如果这样设计，那么从队首插入的效率就非常低了（复制）    
使用    
```java
transient int head;

transient int tail;
```
这个标记，来记录队首和队尾的下标    

根据注释上面说的，本类作为 栈，比 `Stack` 要快（队尾添加，队尾获取，这样使用就是栈了）  
作为 队列，比 `LinkedList` 要快

ArrayDeque，每次扩容的时候都是双倍扩容  
```java
private void doubleCapacity() {
    assert head == tail;
    int p = head;
    int n = elements.length;
    int r = n - p; // number of elements to the right of p
    int newCapacity = n << 1;
    if (newCapacity < 0)
        throw new IllegalStateException("Sorry, deque too big");
    Object[] a = new Object[newCapacity];
    System.arraycopy(elements, p, a, 0, r);
    System.arraycopy(elements, 0, a, r, p);
    elements = a;
    head = 0;
    tail = n;
}
``` 

经过扩容后，head 将置为 0   
我们先看一下 addFirst   
```java
public void addFirst(E e) {
    if (e == null)
        throw new NullPointerException();
    elements[head = (head - 1) & (elements.length - 1)] = e;
    if (head == tail)
        doubleCapacity();
}
``` 
赋值语句比较复杂，我们拆开看    
由于初始容量为2 的指数，每次扩容都是2倍扩容，所以 `(elements.length - 1)` 的结果就是低位全1 
因为在队首插入，因此 `(head - 1)`,head 的新值是两个值 `&` ，其实就是取余。  
我们看一下，第一个调用 addFirst，那么 head 为 15 （默认容量为16）， 
如果 head == tail 了，那么代表数组满了，需要扩容了  
反正注意从队首添加就是 head - 1 取余，从队首删除就是 head + 1 取余  

从队首获取的方法    
```java
public E pollFirst() {
    int h = head;
    @SuppressWarnings("unchecked")
    E result = (E) elements[h];
    // Element is null if deque empty
    if (result == null)
        return null;
    elements[h] = null;     // Must null out slot
    head = (h + 1) & (elements.length - 1);
    return result;
}
``` 
这里结果为null，不抛异常，其他要抛异常的方法也是调用这个方法    

队尾的操作类似  
```java
public void addLast(E e) {
    if (e == null)
        throw new NullPointerException();
    elements[tail] = e;
    if ( (tail = (tail + 1) & (elements.length - 1)) == head)
        doubleCapacity();
}
```     

### ArrayBlockingQueue  
基于数组的阻塞队列，内部使用重入锁和两个条件实现阻塞
实现并不复杂，看一下代表性的方法 offer poll

```java

public boolean offer(E e) {
    // 元素不能为null
    checkNotNull(e);
    final ReentrantLock lock = this.lock;
    // 重入锁，所以阻塞队列是线程安全的
    lock.lock();
    try {
        // 队列已满
        if (count == items.length)
            return false;
        else {
            // 没满，就入队，肯定成功，因为上锁了
            enqueue(e);
            return true;
        }
    } finally {
        lock.unlock();
    }
}

/**
 * 调用这个方法前，先上锁
 */
private void enqueue(E x) {
    // assert lock.getHoldCount() == 1;
    // assert items[putIndex] == null;
    final Object[] items = this.items;
    // putIndex 记录最后入队索引
    items[putIndex] = x;
    if (++putIndex == items.length)
        putIndex = 0;
    count++;
    // 入队后，代表有数据了，通知 notEmpty 条件， 如果有在 notEmpty 上等待的，其实就是想要在非空的时候得到通知
    notEmpty.signal();
}
```     

太简单了，没啥说的。    

```java
/**
    出队
*/   
public E poll() {
    final ReentrantLock lock = this.lock;
    // 同样要上锁
    lock.lock();
    try {
        // 为空就返回null，否则出队操作
        return (count == 0) ? null : dequeue();
    } finally {
        lock.unlock();
    }
}

private E dequeue() {
    // assert lock.getHoldCount() == 1;
    // assert items[takeIndex] != null;
    final Object[] items = this.items;
    // takeIndex 记录出队索引
    @SuppressWarnings("unchecked")
    E x = (E) items[takeIndex];
    items[takeIndex] = null;
    if (++takeIndex == items.length)
        takeIndex = 0;
    count--;
    if (itrs != null)
        itrs.elementDequeued();
    // 出队了，肯定有空位，通知 notFull 条件，如果有在 notFull 上等待的，其实就是想要在非满的时候得到通知
    notFull.signal();
    return x;
}
``` 

基于数组，重入锁，两个条件，putIndex 和 takeIndex。

### LinkedBlockingDeque 
看名字就知道基于双向链表的阻塞队列
内部实现跟 LinkedList 差不多，就是多了重入锁，两个条件。。 


### ConcurrentLinkedDeque   
这个看名字，就能猜到线程安全的，基于链表实现的双端队列  
里面使用循环加上 cas 操作，无锁化编程   
