[TOC]   

# Map   
常用 Map 的实现分析 

## HashMap  
实现原理就是数组，根据 hash 找到数组中对应的位置 （取余）, 如果已经有值了，但是 key不同，构建链表，链表长度 > 8 了，链表变成红黑树。    
key，value 允许 null    

重点方法注释    
putVal 这个方法是往 map 里面添加元素使用的内部方法
```java
/**
* 这里是 put 到 map 的逻辑
*/
/**
    * Implements Map.put and related methods
    *
    * @param hash hash for key
    * @param key the key
    * @param value the value to put
    * @param onlyIfAbsent if true, don't change existing value
    * @param evict if false, the table is in creation mode.
    * @return previous value, or null if none
    */
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // 第一个判断，如果内部的数组还没初始化，那么先初始化（可能说初始化不准确），resize 方法，是要扩容的时候调用的，如果未初始化，的确需要第一次扩容
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    
    // hash 值 对数组长度取余，如果对应 index 位置没有元素，那么直接就放在数组上
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        // 如果 index 上有元素了
        Node<K,V> e; K k;
        // 如果put 的 key 已经存在了（先比较 hash，再判断是否 equals）
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        // 如果已经成为 TreeNode 了
        // 默认情况下，map 内部是一个数组，里面存的是 Node 节点， 如果存在相同位置的元素，那么用 Node 组成单向链表，如果链表长度超过指定长度，进化成 红黑树
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            // 否则，遍历链表，找到相同的key，或者链表尾部插入一个节点，如果长度超过来阀值，变换成红黑树
            for (int binCount = 0; ; ++binCount) {
                // p.next == null 就是遍历到了尾部了
                if ((e = p.next) == null) {
                    // 新建节点，插入链表尾部
                    p.next = newNode(hash, key, value, null);
                    // 如果超过阀值了（8），换成红黑树
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                // 或者，存在 key 相同的节点，找到
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        // e != null 代表存在相同的 key
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            // 如果 onlyIfAbsent = true 代表不替换旧值，当然，如果旧值为 null，那也替换了
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            // 一个空方法，留待子类实现，当相同 key 的节点的时候
            afterNodeAccess(e);
            return oldValue;
        }
    }
    // 如果是新增了节点，修改数 加一
    ++modCount;
    // 如果 自增 size 后 超过阀值了，扩容
    if (++size > threshold)
        resize();
    // 这是一个空方法，插入节点后，留待子类实现自己的动作
    afterNodeInsertion(evict);
    // put 方法，返回的是原来的值，如果没有原来的值，返回 null
    return null;
}
``` 

resize 扩容 
当 size > threshold 的时候，扩容
```java
/**
 * 初始化或者双倍扩容
 */
/**
    * Initializes or doubles table size.  If null, allocates in
    * accord with initial capacity target held in field threshold.
    * Otherwise, because we are using power-of-two expansion, the
    * elements from each bin must either stay at same index, or move
    * with a power of two offset in the new table.
    *
    * @return the table
    */
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    // 拿到 旧的 最大容量 和 扩容门槛
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    // 如果 旧的最大容量 > 0 代表已经初始化过了
    if (oldCap > 0) {
        // 如果旧的容量已经超过 最大容量了，直接把门槛设为 最大 int 值
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // 新容量为旧容量的2倍，&& 旧容量 >= 默认容量（16）
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                    oldCap >= DEFAULT_INITIAL_CAPACITY)
            // 新门槛也2倍
            newThr = oldThr << 1; // double threshold
    }
    else if (oldThr > 0) // initial capacity was placed in threshold
        // 如果在构造的时候，指定了初始值，那么 oldThr 就是初值的 tableSizeFor(cap) 大小，是一个2的指数倍，这个值被赋值给 newCap
        newCap = oldThr;
    else {               // zero initial threshold signifies using defaults
        // 如果是默认的构造，那么，就全部是初值
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    // 看到，在 上面第二个条件中 newThr 没有设值， newThr 将变成 newCap * loadFactor
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                    (int)ft : Integer.MAX_VALUE);
    }
    // 一般是两倍扩容，特例情况是直接制定了参数和超过最大容量,cap 除非超过 最大值，否则，都是 2 的指数倍
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                // 如果原来的元素没有变成链表结构
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                // 如果节点是 红黑树的节点了
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                // 链表结构
                else { // preserve order
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        // 这里看起来，就是将链表，拆分，根据条件 e.hash & oldCap == 0 来判断，其实就是hash中对应位是否为1
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    // 拆分后，分别放在低位 j 和 高位 j + oldCap
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
``` 

getNode 是用来根据key找到对应节点的方法
```java
/**
 * get 方法
 */
/**
    * Implements Map.get and related methods
    *
    * @param hash hash for key
    * @param key the key
    * @return the node, or null if none
    */
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    // 数组有值，并且hash 对应 index 位置有值
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        // 判断 第一个 Node 的 hash，==，equals
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        // 第一个节点不满足，如果有后续节点继续判断
        if ((e = first.next) != null) {
            // 如果是红黑树节点，则在树里面查找
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            do {
                // 否则就遍历链表了，判断 hash, ==, equals
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
``` 

removeNode 这个是根据key删除 节点的方法
```java
/**
* Implements Map.remove and related methods
*
* @param hash hash for key
* @param key the key
* @param value the value to match if matchValue, else ignored
* @param matchValue if true only remove if value is equal
* @param movable if false do not move other nodes while removing
* @return the node, or null if none
*/
final Node<K,V> removeNode(int hash, Object key, Object value,
                            boolean matchValue, boolean movable) {
    Node<K,V>[] tab; Node<K,V> p; int n, index;
    // 根据 hash 找到对应 index，并且 index上有值
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (p = tab[index = (n - 1) & hash]) != null) {
        Node<K,V> node = null, e; K k; V v;
        // 如果数组上的节点就是给定的key（hash，== 或者 equals）
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            node = p;
        else if ((e = p.next) != null) {
            // 如果第一个节点不是，并且有后续节点（链表）
            if (p instanceof TreeNode)
                // 红黑树的情况
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
            else {
                // 遍历链表，找到 key 对应的节点
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key ||
                            (key != null && key.equals(k)))) {
                        node = e;
                        break;
                    }
                    p = e;
                } while ((e = e.next) != null);
            }
        }
        // 如果找到了，后面的条件是判断 节点对应的value ，是不是给定的 value（equals）
        if (node != null && (!matchValue || (v = node.value) == value ||
                                (value != null && value.equals(v)))) {
            // 如果是红黑树，那就红黑树里面删除节点
            if (node instanceof TreeNode)
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
            // 如果是第一个节点
            else if (node == p)
                tab[index] = node.next;
            // p 是e的前一个节点
            else
                p.next = node.next;
            ++modCount;
            --size;
            afterNodeRemoval(node);
            return node;
        }
    }
    return null;
}   
``` 

实现里面，只有红黑树的部分很难看懂（我也没有分析。）    
还有就是 HashMap 空间浪费严重，可以看懂，当 cap 为 16 的时候 threshold = cap * loadFactor(默认 0.75)    
当 size > threshold 的时候就要双倍扩容。。  

## Hashtable
这个就不贴代码了，简单说一个下，很少用。    
基本所有方法都有 synchronized 方法，所以线程安全（基本不用它），hash 方法跟 HashMap 不同，
扩容是双倍 + 1 扩容，并没有限定为2的指数，不能接受 key，value 为null    

 
## ConcurrentHashMap    
参考 [brickworkers的博客](https://blog.csdn.net/u012403290)
利用 CAS 和 锁数组里头结点来提高并发性能    
重点还是看看 put，get 和 扩容操作   

再看代码之前，先简单说一下PUT的具体操作： 
1. 先传入一个k和v的键值对，不可为空（HashMap是可以为空的），如果为空就直接报错。 
2. 接着去判断table是否为空，如果为空就进入初始化阶段。 
3. 如果判断数组中某个指定的桶是空的，那就直接把键值对插入到这个桶中作为头节点，而且这个操作不用加锁。 
4. 如果这个要插入的桶中的hash值为-1，也就是MOVED状态（也就是这个节点是forwordingNode），那就是说明有线程正在进行扩容操作，那么当前线程就进入协助扩容阶段。 
5. 需要把数据插入到链表或者树中，如果这个节点是一个链表节点，那么就遍历这个链表，如果发现有相同的key值就更新value值，如果遍历完了都没有发现相同的key值，就需要在链表的尾部插入该数据。
插入结束之后判断该链表节点个数是否大于8，如果大于就需要把链表转化为红黑树存储。 
6.  ⑥如果这个节点是一个红黑树节点，那就需要按照树的插入规则进行插入。 
7. put结束之后，需要给map已存储的数量+1，在addCount方法中判断是否需要扩容   

putVal 方法如下
```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    // key 和 value 不能为 null
    if (key == null || value == null) throw new NullPointerException();
    // hash spread 方法将hash再处理了一下，减少碰撞
    int hash = spread(key.hashCode());
    int binCount = 0;
    // 循环保证完成操作，完成后跳出去
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        // 初始化table，也就是数组
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        // 如果 index 上没有节点，这里的 tabAt 方法使用 Unsafe 里面的方法 getObjectVolatile 根据地址偏移取对象
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            // cas 设置新节点，如果设置成功，那就结束了
            if (casTabAt(tab, i, null,
                            new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        //如果在插入的时候，节点是一个forwordingNode状态，表示正在扩容，那么当前线程进行帮助扩容
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            // 
            V oldVal = null;
            // 锁数组里面的单个元素
            synchronized (f) {
                // 再次确认一下，数组对应位置的节点变了没
                if (tabAt(tab, i) == f) {
                    // fh 是 f.hash
                    if (fh >= 0) {
                        // 链表长度
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            // 逻辑跟 HashMap 一样，如果存在相同的key，那就覆盖
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                    (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            // 没有的话，就在链表后面添加
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                            value, null);
                                break;
                            }
                        }
                    }
                    // 红黑树节点
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                        value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            // 深度不为0
            if (binCount != 0) {
                // 深度超过要变化为 红黑树则变化
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                // 返回旧值
                if (oldVal != null)
                    return oldVal;
                // 没有旧值，代表添加了新节点，需要在外面设置一下
                break;
            }
        }
    }
    //map已存储的数量+1,里面存在扩容
    addCount(1L, binCount);
    return null;
}
``` 

代码上面已经有注释了，里面需要注意的方法是 addCount 和 helpTransfer 方法，这里跟扩容有关    
```java
private final void addCount(long x, int check) {
    CounterCell[] as; long b, s;
    /**
        一般 counterCells 是空的，所以总是执行 后面的cas 操作，如果是true 就直接返回了
        如果是并发插入，其中一个线程cas 操作失败返回false ,则触发counterCells 的操作。
        这里的设计理念我的理解是分两级(毕竟并发的风险不高)的进行计算，保证效率，防止自旋 , 所以进入 fullAddCount() 里面是自旋的
        */
    if ((as = counterCells) != null ||
        !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
        CounterCell a; long v; int m;
        boolean uncontended = true;
        // 当 as 为null 长度 < 0 获取随机获取一个位置为null，不为null，cas 增加操作失败，任意一个条件满足 调用，fullAddCount
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
            !(uncontended =
                U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
            // 这个方法就是在并发添加的时候，里面自旋保证增加成功
            fullAddCount(x, uncontended);
            return;
        }
        if (check <= 1)
            return;
        s = sumCount();
    }
    // 当check >= 0 的时候，就是需要检查 size 并扩容
    if (check >= 0) {
        Node<K,V>[] tab, nt; int n, sc;
        // sizeCtl 就是要扩容的阀值，默认是数组长度的 0.75
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
                (n = tab.length) < MAXIMUM_CAPACITY) {
            
            int rs = resizeStamp(n);
            /**
                当 sizeCtl 小于 0 进入扩容状态后，其他线程只需要拿到 nextTable 进入transfer() 方法 ，就可以触发并发扩容
                */
            if (sc < 0) {
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    /**
                        已经处于扩容状态下，直接传入 nextTable 进行触发 并发扩容 操作
                        */
                    transfer(tab, nt);
            }
            // 这里设置 sizeCtl 为负数了
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                            (rs << RESIZE_STAMP_SHIFT) + 2))
                // 设置成功，开始扩容
                transfer(tab, null);
            // sumCount 方法就是 遍历 counterCells 把value值相加,再加上baseCount  ，计算得long 值转成int 型输出
            s = sumCount();
        }
    }
}
``` 

addCount 不使用锁，cas 尝试添加，如果失败了，在 CounterCell 里面添加，循环加 cas 成功，count 就是 baseCount 加上 CounterCell 里面的和   

扩容方法 transfer 和 helpTransfer   
```java
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
    Node<K,V>[] nextTab; int sc;
    // 如果 table 不是空 且 node 节点是转移类型，数据检验
    // 且 node 节点的 nextTable（新 table） 不是空，同样也是数据校验
    // 尝试帮助扩容
    if (tab != null && (f instanceof ForwardingNode) &&
        (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
        int rs = resizeStamp(tab.length);
        // 如果 nextTab 没有被并发修改 且 tab 也没有被并发修改
        // 且 sizeCtl  < 0 （说明还在扩容）
        while (nextTab == nextTable && table == tab &&
                (sc = sizeCtl) < 0) {
        // 如果 sizeCtl 无符号右移  16 不等于 rs （ sc前 16 位如果不等于标识符，则标识符变化了）
        // 或者 sizeCtl == rs + 1  （扩容结束了，不再有线程进行扩容）（默认第一个线程设置 sc ==rs 左移 16 位 + 2，当第一个线程结束扩容了，就会将 sc 减一。这个时候，sc 就等于 rs + 1）
        // 或者 sizeCtl == rs + 65535  （如果达到最大帮助线程的数量，即 65535）
        // 或者转移下标正在调整 （扩容结束）
        // 结束循环，返回 table
            if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                sc == rs + MAX_RESIZERS || transferIndex <= 0)
                break;
            // 如果以上都不是, 将 sizeCtl + 1, （表示增加了一个线程帮助其扩容）
            if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                // 进行转移
                transfer(tab, nextTab);
                // 结束循环
                break;
            }
        }
        return nextTab;
    }
    return table;
}

``` 

扩容的过程：首先有且只能由一个线程构建一个nextTable，这个nextTable主要是扩容后的数组（容量已经扩大），然后把原table复制到nextTable中，这个过程可以多线程共同操作。但是一定要清楚，这个复制并不是简单的把原table的数据直接移动到nextTable中，而是需要有一定的规律和算法操控的（不然怎么把树转化为链表呢）。

简单说下复制的过程： 
数组中（桶中）总共分为3种存储情况：空，链表头，TreeBin头 
1. 遍历原来的数组（原table），如果数组中某个值为空，则直接放置一个forwordingNode（上篇博文介绍过）。 
2. 如果数组中某个值不为空，而是一个链表头结点，那么就对这个链表进行拆分为两个链表，存储到nextTable对应的两个位置。 
3. 如果数组中某个值不为空，而是一个TreeBin头结点，那么这个地方就存储的是红黑树的结构，这样一来，处理就会变得相对比较复杂，就需要先判断需不需要把树转换为链表，做完一系列的处理，然后把对应的结果存储在nextTable的对应两个位置。 

```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;
    //主要是判断CPU处理的量，如果小于16则直接赋值16
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range
    // initiating只能有一个线程进行构造nextTable，如果别的线程进入发现不为空就不用构造nextTable了
    if (nextTab == null) {            // initiating
        try {
            // 初始化新table ，原来的两倍
            
            @SuppressWarnings("unchecked")
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        nextTable = nextTab;
        transferIndex = n; //原先扩容大小
    }
    int nextn = nextTab.length;
    //构造一个ForwardingNode用于多线程之间的共同扩容情况
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    boolean advance = true;//遍历的确认标志,这个标志是每个槽是否处理完毕的标记
    boolean finishing = false; // to ensure sweep before committing nextTab
    //遍历每个节点
    for (int i = 0, bound = 0;;) {
        //定义一个节点和一个节点状态判断标志fh
        Node<K,V> f; int fh;
        // 这个循环，是用来找本线程要处理哪一个槽（index）变量 i
        while (advance) {
            int nextIndex, nextBound;
            if (--i >= bound || finishing)
                advance = false;
            else if ((nextIndex = transferIndex) <= 0) {
                i = -1;
                advance = false;
            }
            //下面就是一个CAS计算
            else if (U.compareAndSwapInt
                        (this, TRANSFERINDEX, nextIndex,
                        nextBound = (nextIndex > stride ?
                                    nextIndex - stride : 0))) {
                bound = nextBound;
                i = nextIndex - 1;
                advance = false;
            }
        }
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            //如果原table已经复制结束
            if (finishing) {
                //可以看出在扩容的时候nextTable只是类似于一个temp用完会丢掉
                nextTable = null;
                table = nextTab;
                sizeCtl = (n << 1) - (n >>> 1);//修改扩容后的阀值，应该是现在容量的0.75倍
                return;//结束循环
            }
            //采用CAS算法更新SizeCtl。
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                finishing = advance = true;
                i = n; // recheck before commit
            }
        }
        //CAS算法获取某一个数组的节点，为空就设为forwordingNode
        else if ((f = tabAt(tab, i)) == null)
            advance = casTabAt(tab, i, null, fwd);
        //如果这个节点的hash值是MOVED，就表示这个节点是forwordingNode节点，就表示这个节点已经被处理过了，直接跳过
        else if ((fh = f.hash) == MOVED)
            advance = true; // already processed
        else {
            //对头节点进行加锁，禁止别的线程进入
            synchronized (f) {
                //CAS校验这个节点是否在table对应的i处
                if (tabAt(tab, i) == f) {
                    Node<K,V> ln, hn;
                    //如果这个节点的确是链表节点
                    //把链表拆分成两个小列表并存储到nextTable对应的两个位置
                    if (fh >= 0) {
                        // fh & n 结果 为 0 或者 n （n 就是数组大小，是2 的指数倍，所以只有一位为1，其他为0）
                        int runBit = fh & n;
                        Node<K,V> lastRun = f;
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            int b = p.hash & n;
                            if (b != runBit) {
                                runBit = b;
                                lastRun = p;
                            }
                        }
                        if (runBit == 0) {
                            ln = lastRun;
                            hn = null;
                        }
                        else {
                            hn = lastRun;
                            ln = null;
                        }
                        // 这里 Node 最后一个参数是 next，循环只到 lastRun，确保的是 lastRun 后面跟之前的顺序是一样的，前面都是倒叙
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            if ((ph & n) == 0)
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                        //CAS存储在nextTable的i位置上
                        setTabAt(nextTab, i, ln);
                        //CAS存储在nextTable的i+n位置上
                        setTabAt(nextTab, i + n, hn);
                        //CAS在原table的i处设置forwordingNode节点，表示这个这个节点已经处理完毕
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                    //如果这个节点是红黑树
                    else if (f instanceof TreeBin) {
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> lo = null, loTail = null;
                        TreeNode<K,V> hi = null, hiTail = null;
                        int lc = 0, hc = 0;
                        for (Node<K,V> e = t.first; e != null; e = e.next) {
                            int h = e.hash;
                            TreeNode<K,V> p = new TreeNode<K,V>
                                (h, e.key, e.val, null, null);
                            if ((h & n) == 0) {
                                if ((p.prev = loTail) == null)
                                    lo = p;
                                else
                                    loTail.next = p;
                                loTail = p;
                                ++lc;
                            }
                            else {
                                if ((p.prev = hiTail) == null)
                                    hi = p;
                                else
                                    hiTail.next = p;
                                hiTail = p;
                                ++hc;
                            }
                        }
                        //如果拆分后的树的节点数量已经少于6个就需要重新转化为链表
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                            (hc != 0) ? new TreeBin<K,V>(lo) : t;
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                            (lc != 0) ? new TreeBin<K,V>(hi) : t;
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                }
            }
        }
    }
}
``` 
上面，链表拆分，写了个测试代码
可以将 n 设置为 8 和 16 看看
```java
public class NodeStudy {

    static class Node {
        private int hash;
        private int value;
        private Node next;

        public Node(int hash, int value, Node next) {
            this.hash = hash;
            this.value = value;
            this.next = next;
        }

        public int getHash() {
            return hash;
        }

        public int getValue() {
            return value;
        }

        public Node getNext() {
            return next;
        }
    }

    public static void main(String[] args) {
        int n = 16;
        Node head = null, c = null;
        for (int i = 0; i < 30; i++) {
            Node node = new Node(i, i, null);
            if (Objects.isNull(c)) {
                c = head = node;
            } else {
                c.next = node;
                c = node;
            }
        }
        // 长度为 30 的单向链表
//        for (Node p = head; p != null; p = p.next) {
//            System.out.println(p.hash);
//        }

        int fh = head.hash;
        int runBit = head.hash & n;
        Node ln, hn;
        Node lastRun = null;
        for (Node p = head.next; p != null; p = p.next) {
            int b = p.hash & n;
            if (b != runBit) {
                runBit = b;
                lastRun = p;
            }
        }
        System.out.println("lastRun");
        System.out.println(lastRun.hash);
        if (runBit == 0) {
            ln = lastRun;
            hn = null;
        }
        else {
            hn = lastRun;
            ln = null;
        }

        for (Node p = head; p != lastRun; p = p.next) {
            int ph = p.hash;
            if ((ph & n) == 0)
                ln = new Node(ph, p.value, ln);
            else
                hn = new Node(ph, p.value, hn);
        }

        System.out.println("======ln========");

        for (Node p = ln; p != null; p = p.next) {
            System.out.println(p.hash);
        }
        System.out.println("======hn========");
        for (Node p = hn; p != null; p = p.next) {
            System.out.println(p.hash);
        }
    }
}
``` 


get 方法就很简单了  
```java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    int h = spread(key.hashCode());
    //数组已被初始化且指定桶中不为空
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {
            //先判断头节点，如果头节点的hash值与入参key的hash值相同
        if ((eh = e.hash) == h) {
            //头节点的key就是传入的key
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        //eh<0表示这个节点是红黑树
        else if (eh < 0)
            return (p = e.find(h, key)) != null ? p.val : null;//直接从树上进行查找返回结果，不存在就返回null
        //如果首节点不是查找对象且不是红黑树结构，那边就遍历这个列表
        while ((e = e.next) != null) {
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    //都没有找到就直接返回null值
    return null;
}
``` 

删除和替换操作基于以下方法  
```java
final V replaceNode(Object key, V value, Object cv) {
    int hash = spread(key.hashCode());
    // 循环 加 cas 保证成功
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        // 没初始化，对应位置上没有节点，给定的key不存在
        if (tab == null || (n = tab.length) == 0 ||
            (f = tabAt(tab, i = (n - 1) & hash)) == null)
            break;
        // 如果 位置的头结点 hash == MOVED，代表真正扩容，本线程也去辅助扩容 
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            // 否则，就正常读取
            V oldVal = null;
            boolean validated = false;
            // 锁住头结点
            synchronized (f) {
                // double check 判断锁住的节点没被替换
                if (tabAt(tab, i) == f) {
                    // 链表结构
                    if (fh >= 0) {
                        validated = true;
                        for (Node<K,V> e = f, pred = null;;) {
                            K ek;
                            // 找到对应的 key了
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                    (ek != null && key.equals(ek)))) {
                                
                                V ev = e.val;
                                // 如果预期的 value 为null 或者 对应的value 就是预期的 value
                                if (cv == null || cv == ev ||
                                    (ev != null && cv.equals(ev))) {
                                    // 获得旧值
                                    oldVal = ev;
                                    // 如果要替换的 value 不为null，如果为null，就是要删除
                                    if (value != null)
                                        e.val = value;
                                    else if (pred != null)
                                        pred.next = e.next;
                                    else
                                        setTabAt(tab, i, e.next);
                                }
                                break;
                            }
                            pred = e;
                            if ((e = e.next) == null)
                                break;
                        }
                    }
                    // 红黑树节点
                    else if (f instanceof TreeBin) {
                        validated = true;
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> r, p;
                        if ((r = t.root) != null &&
                            (p = r.findTreeNode(hash, key, null)) != null) {
                            V pv = p.val;
                            if (cv == null || cv == pv ||
                                (pv != null && cv.equals(pv))) {
                                oldVal = pv;
                                if (value != null)
                                    p.val = value;
                                else if (t.removeTreeNode(p))
                                    setTabAt(tab, i, untreeify(t.first));
                            }
                        }
                    }
                }
            }
            // 如果是删除，数量 -1
            if (validated) {
                if (oldVal != null) {
                    if (value == null)
                        addCount(-1L, -1);
                    return oldVal;
                }
                break;
            }
        }
    }
    return null;
}
``` 


## TreeMap  
基于红黑树的map，key 有序   

### 红黑树概述  
红黑树（Red Black Tree） 是一种自平衡二叉查找树
参考 [红黑树(一)之 原理和算法详细介绍](http://www.cnblogs.com/skywang12345/p/3245399.html)
红黑树的特性:
1. 每个节点或者是黑色，或者是红色。
2. 根节点是黑色。
3. 每个叶子节点（NIL）是黑色。 [注意：这里叶子节点，是指为空(NIL或NULL)的叶子节点！]
4. 如果一个节点是红色的，则它的子节点必须是黑色的。
5. 从一个节点到该节点的子孙节点的所有路径上包含相同数目的黑节点。   

**左旋，右旋**
左旋中的“左”，意味着“被旋转的节点将变成一个左节点”
右旋中的“右”，意味着“被旋转的节点将变成一个右节点”  

**添加**    
1. 将红黑树当作一颗二叉查找树，将节点插入。
红黑树本身就是一颗二叉查找树，将节点插入后，该树仍然是一颗二叉查找树。也就意味着，树的键值仍然是有序的。此外，无论是左旋还是右旋，若旋转之前这棵树是二叉查找树，旋转之后它一定还是二叉查找树。这也就意味着，任何的旋转和重新着色操作，都不会改变它仍然是一颗二叉查找树的事实
那接下来，我们就来想方设法的旋转以及重新着色，使这颗树重新成为红黑树
2. 将插入的节点着色为"红色"
为什么着色成红色，而不是黑色呢？    
将插入的节点着色为红色，不会违背"特性(5)"！少违背一条特性，就意味着我们需要处理的情况越少。接下来，就要努力的让这棵树满足其它性质即可；满足了的话，它就又是一颗红黑树了

3. 通过一系列的旋转或着色等操作，使之重新成为一颗红黑树。   
第二步中，将插入节点着色为"红色"之后，不会违背"特性(5)"。那它到底会违背哪些特性呢？
对于"特性(1)"，显然不会违背了。因为我们已经将它涂成红色了。
对于"特性(2)"，显然也不会违背。在第一步中，我们是将红黑树当作二叉查找树，然后执行的插入操作。而根据二叉查找数的特点，插入操作不会改变根节点。所以，根节点仍然是黑色。
对于"特性(3)"，显然不会违背了。这里的叶子节点是指的空叶子节点，插入非空节点并不会对它们造成影响。
对于"特性(4)"，是有可能违背的！
那接下来，想办法使之"满足特性(4)"，就可以将树重新构造成红黑树了。

添加的伪码  
```c
RB-INSERT(T, z)  
 y ← nil[T]                        // 新建节点“y”，将y设为空节点。
 x ← root[T]                       // 设“红黑树T”的根节点为“x”
 while x ≠ nil[T]                  // 找出要插入的节点“z”在二叉树T中的位置“y”
     do y ← x                      
        if key[z] < key[x]  
           then x ← left[x]  
           else x ← right[x]  
 p[z] ← y                          // 设置 “z的父亲” 为 “y”
 if y = nil[T]                     
    then root[T] ← z               // 情况1：若y是空节点，则将z设为根
    else if key[z] < key[y]        
            then left[y] ← z       // 情况2：若“z所包含的值” < “y所包含的值”，则将z设为“y的左孩子”
            else right[y] ← z      // 情况3：(“z所包含的值” >= “y所包含的值”)将z设为“y的右孩子” 
 left[z] ← nil[T]                  // z的左孩子设为空
 right[z] ← nil[T]                 // z的右孩子设为空。至此，已经完成将“节点z插入到二叉树”中了。
 color[z] ← RED                    // 将z着色为“红色”
 RB-INSERT-FIXUP(T, z)             // 通过RB-INSERT-FIXUP对红黑树的节点进行颜色修改以及旋转，让树T仍然是一颗红黑树

 RB-INSERT-FIXUP(T, z)
while color[p[z]] = RED                                                  // 若“当前节点(z)的父节点是红色”，则进行以下处理。
    do if p[z] = left[p[p[z]]]                                           // 若“z的父节点”是“z的祖父节点的左孩子”，则进行以下处理。
          then y ← right[p[p[z]]]                                        // 将y设置为“z的叔叔节点(z的祖父节点的右孩子)”
               if color[y] = RED                                         // Case 1条件：叔叔是红色
                  then color[p[z]] ← BLACK                    ▹ Case 1   //  (01) 将“父节点”设为黑色。
                       color[y] ← BLACK                       ▹ Case 1   //  (02) 将“叔叔节点”设为黑色。
                       color[p[p[z]]] ← RED                   ▹ Case 1   //  (03) 将“祖父节点”设为“红色”。
                       z ← p[p[z]]                            ▹ Case 1   //  (04) 将“祖父节点”设为“当前节点”(红色节点)
                  else if z = right[p[z]]                                // Case 2条件：叔叔是黑色，且当前节点是右孩子
                          then z ← p[z]                       ▹ Case 2   //  (01) 将“父节点”作为“新的当前节点”。
                               LEFT-ROTATE(T, z)              ▹ Case 2   //  (02) 以“新的当前节点”为支点进行左旋。
                          color[p[z]] ← BLACK                 ▹ Case 3   // Case 3条件：叔叔是黑色，且当前节点是左孩子。(01) 将“父节点”设为“黑色”。
                          color[p[p[z]]] ← RED                ▹ Case 3   //  (02) 将“祖父节点”设为“红色”。
                          RIGHT-ROTATE(T, p[p[z]])            ▹ Case 3   //  (03) 以“祖父节点”为支点进行右旋。
       else (same as then clause with "right" and "left" exchanged)      // 若“z的父节点”是“z的祖父节点的右孩子”，将上面的操作中“right”和“left”交换位置，然后依次执行。
color[root[T]] ← BLACK
``` 

**删除**
将红黑树内的某一个节点删除。需要执行的操作依次是：首先，将红黑树当作一颗二叉查找树，将该节点从二叉查找树中删除；然后，通过"旋转和重新着色"等一系列来修正该树，使之重新成为一棵红黑树。详细描述如下：
1. 将红黑树当作一颗二叉查找树，将节点删除。
这和"删除常规二叉查找树中删除节点的方法是一样的"。分3种情况：
    1. 被删除节点没有儿子，即为叶节点。那么，直接将该节点删除就OK了。
    2. 被删除节点只有一个儿子。那么，直接删除该节点，并用该节点的唯一子节点顶替它的位置。
    3. 被删除节点有两个儿子。那么，先找出它的后继节点；然后把“它的后继节点的内容”复制给“该节点的内容”；之后，删除“它的后继节点”。在这里，后继节点相当于替身，在将后继节点的内容复制给"被删除节点"之后，再将后继节点删除。这样就巧妙的将问题转换为"删除后继节点"的情况了，下面就考虑后继节点。 在"被删除节点"有两个非空子节点的情况下，它的后继节点不可能是双子非空。既然"的后继节点"不可能双子都非空，就意味着"该节点的后继节点"要么没有儿子，要么只有一个儿子。若没有儿子，则按"情况1 "进行处理；若只有一个儿子，则按"情况2 "进行处理。

2. 通过"旋转和重新着色"等一系列来修正该树，使之重新成为一棵红黑树。
因为"第一步"中删除节点之后，可能会违背红黑树的特性。所以需要通过"旋转和重新着色"来修正该树，使之重新成为一棵红黑树。

```c
RB-DELETE(T, z)
if left[z] = nil[T] or right[z] = nil[T]         
   then y ← z                                  // 若“z的左孩子” 或 “z的右孩子”为空，则将“z”赋值给 “y”；
   else y ← TREE-SUCCESSOR(z)                  // 否则，将“z的后继节点”赋值给 “y”。
if left[y] ≠ nil[T]
   then x ← left[y]                            // 若“y的左孩子” 不为空，则将“y的左孩子” 赋值给 “x”；
   else x ← right[y]                           // 否则，“y的右孩子” 赋值给 “x”。
p[x] ← p[y]                                    // 将“y的父节点” 设置为 “x的父节点”
if p[y] = nil[T]                               
   then root[T] ← x                            // 情况1：若“y的父节点” 为空，则设置“x” 为 “根节点”。
   else if y = left[p[y]]                    
           then left[p[y]] ← x                 // 情况2：若“y是它父节点的左孩子”，则设置“x” 为 “y的父节点的左孩子”
           else right[p[y]] ← x                // 情况3：若“y是它父节点的右孩子”，则设置“x” 为 “y的父节点的右孩子”
if y ≠ z                                    
   then key[z] ← key[y]                        // 若“y的值” 赋值给 “z”。注意：这里只拷贝z的值给y，而没有拷贝z的颜色！！！
        copy y's satellite data into z         
if color[y] = BLACK                            
   then RB-DELETE-FIXUP(T, x)                  // 若“y为黑节点”，则调用
return y

RB-DELETE-FIXUP(T, x)
while x ≠ root[T] and color[x] = BLACK  
    do if x = left[p[x]]      
          then w ← right[p[x]]                                             // 若 “x”是“它父节点的左孩子”，则设置 “w”为“x的叔叔”(即x为它父节点的右孩子)                                          
               if color[w] = RED                                           // Case 1: x是“黑+黑”节点，x的兄弟节点是红色。(此时x的父节点和x的兄弟节点的子节点都是黑节点)。
                  then color[w] ← BLACK                        ▹  Case 1   //   (01) 将x的兄弟节点设为“黑色”。
                       color[p[x]] ← RED                       ▹  Case 1   //   (02) 将x的父节点设为“红色”。
                       LEFT-ROTATE(T, p[x])                    ▹  Case 1   //   (03) 对x的父节点进行左旋。
                       w ← right[p[x]]                         ▹  Case 1   //   (04) 左旋后，重新设置x的兄弟节点。
               if color[left[w]] = BLACK and color[right[w]] = BLACK       // Case 2: x是“黑+黑”节点，x的兄弟节点是黑色，x的兄弟节点的两个孩子都是黑色。
                  then color[w] ← RED                          ▹  Case 2   //   (01) 将x的兄弟节点设为“红色”。
                       x ←  p[x]                               ▹  Case 2   //   (02) 设置“x的父节点”为“新的x节点”。
                  else if color[right[w]] = BLACK                          // Case 3: x是“黑+黑”节点，x的兄弟节点是黑色；x的兄弟节点的左孩子是红色，右孩子是黑色的。
                          then color[left[w]] ← BLACK          ▹  Case 3   //   (01) 将x兄弟节点的左孩子设为“黑色”。
                               color[w] ← RED                  ▹  Case 3   //   (02) 将x兄弟节点设为“红色”。
                               RIGHT-ROTATE(T, w)              ▹  Case 3   //   (03) 对x的兄弟节点进行右旋。
                               w ← right[p[x]]                 ▹  Case 3   //   (04) 右旋后，重新设置x的兄弟节点。
                        color[w] ← color[p[x]]                 ▹  Case 4   // Case 4: x是“黑+黑”节点，x的兄弟节点是黑色；x的兄弟节点的右孩子是红色的。(01) 将x父节点颜色 赋值给 x的兄弟节点。
                        color[p[x]] ← BLACK                    ▹  Case 4   //   (02) 将x父节点设为“黑色”。
                        color[right[w]] ← BLACK                ▹  Case 4   //   (03) 将x兄弟节点的右子节设为“黑色”。
                        LEFT-ROTATE(T, p[x])                   ▹  Case 4   //   (04) 对x的父节点进行左旋。
                        x ← root[T]                            ▹  Case 4   //   (05) 设置“x”为“根节点”。
       else (same as then clause with "right" and "left" exchanged)        // 若 “x”是“它父节点的右孩子”，将上面的操作中“right”和“left”交换位置，然后依次执行。
color[x] ← BLACK
``` 

TreeMap 完全基于红黑树结构，重要的只有 插入 和 删除（包含修正），没有扩容概念，get 也是 红黑树查找，Ologn。 
其实代码不重要，重要是理解红黑树（挺难理解的），反正知道就行了。    

## WeakHashMap  
这个实现跟 HashMap 差不多，不过key使用了弱引用，当内存不足的时候，会被回收（回收时间不确定），可以通过一个queue，来搜集被回收的key  
重点代码    
```java
private static class Entry<K,V> extends WeakReference<Object> implements Map.Entry<K,V> {
    V value;
    final int hash;
    Entry<K,V> next;

    /**
        * Creates new entry.
        */
    Entry(Object key, V value,
            ReferenceQueue<Object> queue,
            int hash, Entry<K,V> next) {
        super(key, queue);
        this.value = value;
        this.hash  = hash;
        this.next  = next;
    }
``` 

复制了一部分，看到 Entry 继承了 WeakReference，构造的时候传了key 和 queue   
