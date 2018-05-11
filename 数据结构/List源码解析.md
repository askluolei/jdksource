[TOC]   

# List 源码解析 
List 体系下的部分实现分析   

## ArrayList    
ArrayList 在平时用的很多，是基于数组的。可随机访问。里面值得注意的地方扩容 
```java
private void ensureCapacityInternal(int minCapacity) {
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
    }

    ensureExplicitCapacity(minCapacity);
}

private void ensureExplicitCapacity(int minCapacity) {
    modCount++;

    // overflow-conscious code
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}

private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually close to size, so this is a win:
    elementData = Arrays.copyOf(elementData, newCapacity);
}

private static int hugeCapacity(int minCapacity) {
    if (minCapacity < 0) // overflow
        throw new OutOfMemoryError();
    return (minCapacity > MAX_ARRAY_SIZE) ?
        Integer.MAX_VALUE :
        MAX_ARRAY_SIZE;
}
``` 
数组扩容    
从上面可以看到默认是 1.5 倍扩容 `int newCapacity = oldCapacity + (oldCapacity >> 1)`    
如果指定的最小容量比 1.5 倍大，那就用指定的值。数组的容量上限是 `Integer.MAX_VALUE` 
可以看到，扩容后，将有一次复制操作 `Arrays.copyOf(elementData, newCapacity)`    
内部的实现是 `System.arrayCopy` 方法    
```java
public static native void arraycopy(Object src,  int  srcPos,
                                        Object dest, int destPos,
                                        int length);
``` 
这是一个 native 方法    

需要注意的是，在 List 中间插入或者删除一个元素操作，都会有一次复制操作    
```java
public void add(int index, E element) {
    rangeCheckForAdd(index);

    ensureCapacityInternal(size + 1);  // Increments modCount!!
    System.arraycopy(elementData, index, elementData, index + 1,
                        size - index);
    elementData[index] = element;
    size++;
}

public E remove(int index) {
    rangeCheck(index);

    modCount++;
    E oldValue = elementData(index);

    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                            numMoved);
    elementData[--size] = null; // clear to let GC do its work

    return oldValue;
}
``` 

因此，ArrayList 的优缺点就很清楚了

另外 `ArrayList` 是非线程安全的  



## Vector
基本跟 ArrayList 一样，不同的是，几乎所有方法都用 `synchronized` 关键字了，所以是线程安全的。   
虽说是线程安全的，但是如果要在多线程情况下使用的话，效率非常差，就是串行访问了。    

不同的增长方式，在 `Vecotr` 中，在构造的时候可以指定每次增长的长度，如果不指定，那就双倍（最小）    
```java
public Vector(int initialCapacity, int capacityIncrement) {
    super();
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal Capacity: "+
                                            initialCapacity);
    this.elementData = new Object[initialCapacity];
    this.capacityIncrement = capacityIncrement;
}

public synchronized void ensureCapacity(int minCapacity) {
    if (minCapacity > 0) {
        modCount++;
        ensureCapacityHelper(minCapacity);
    }
}

private void ensureCapacityHelper(int minCapacity) {
    // overflow-conscious code
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}

private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + ((capacityIncrement > 0) ?
                                        capacityIncrement : oldCapacity);
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    elementData = Arrays.copyOf(elementData, newCapacity);
}
``` 

## LinkedList   
内部的实现是双向链表    
```java
private static class Node<E> {
    E item;
    Node<E> next;
    Node<E> prev;

    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
``` 

不止是 List 的实现，还是 Deque 的实现   
由于内部是双向链表实现的 List，所以就不存在扩容了。 
双向列表，向链表中插入，重点在于找到位置，这个操作最耗时，LinkedList 是不支持随机访问的。   
看看重点方法就知道了    
内部根据 index 找到节点的方式 node
```java
Node<E> node(int index) {
    // assert isElementIndex(index);

    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else {
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}
``` 
这里做了一个简单的优化，根据 index 更靠近 first 还是 last 来决定从 first 开始遍历，还是从 last 遍历 
然后插入，删除操作，就是先用这个方法定位，然后使用下面的方法来操作  
```java
private void linkFirst(E e) {
    final Node<E> f = first;
    final Node<E> newNode = new Node<>(null, e, f);
    first = newNode;
    if (f == null)
        last = newNode;
    else
        f.prev = newNode;
    size++;
    modCount++;
}

void linkLast(E e) {
    final Node<E> l = last;
    final Node<E> newNode = new Node<>(l, e, null);
    last = newNode;
    if (l == null)
        first = newNode;
    else
        l.next = newNode;
    size++;
    modCount++;
}

void linkBefore(E e, Node<E> succ) {
    // assert succ != null;
    final Node<E> pred = succ.prev;
    final Node<E> newNode = new Node<>(pred, e, succ);
    succ.prev = newNode;
    if (pred == null)
        first = newNode;
    else
        pred.next = newNode;
    size++;
    modCount++;
}
``` 
上面 3 个方法是添加操作 
```java
private E unlinkFirst(Node<E> f) {
    // assert f == first && f != null;
    final E element = f.item;
    final Node<E> next = f.next;
    f.item = null;
    f.next = null; // help GC
    first = next;
    if (next == null)
        last = null;
    else
        next.prev = null;
    size--;
    modCount++;
    return element;
}

private E unlinkLast(Node<E> l) {
    // assert l == last && l != null;
    final E element = l.item;
    final Node<E> prev = l.prev;
    l.item = null;
    l.prev = null; // help GC
    last = prev;
    if (prev == null)
        first = null;
    else
        prev.next = null;
    size--;
    modCount++;
    return element;
}

E unlink(Node<E> x) {
    // assert x != null;
    final E element = x.item;
    final Node<E> next = x.next;
    final Node<E> prev = x.prev;

    if (prev == null) {
        first = next;
    } else {
        prev.next = next;
        x.prev = null;
    }

    if (next == null) {
        last = prev;
    } else {
        next.prev = prev;
        x.next = null;
    }

    x.item = null;
    size--;
    modCount++;
    return element;
}
``` 
上面3个操作是删除操作。基本上所有的公开方法就是基于上面的方法的。譬如   
```java
public void add(int index, E element) {
    checkPositionIndex(index);

    if (index == size)
        linkLast(element);
    else
        linkBefore(element, node(index));
}

public E remove(int index) {
    checkElementIndex(index);
    return unlink(node(index));
}

public E get(int index) {
    checkElementIndex(index);
    return node(index).item;
}
``` 

还有一个需要注意的地方是 iterator 方法， LinkedList 重写了 ListIterator 的实现，内部除了记录 index，还记录 节点，这个，就不用每次都用 index 来找上下节点了   
所以，一个代码的细节就是，如果使用了 LinkedList， 就不要使用 `for(int i = 0; i < list.size; i++)` 这样的循环了，应该使用 foreach 循环 `for(item : list)` 或者使用 iterator  
LinkedList 是非线程安全的   
