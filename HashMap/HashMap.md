# HashMap
## 继承、实现关系

```
//继承了AbstractMap，实现了Map，Cloneable，Serializable接口
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable 
```
## 内部成员介绍
### DEFAULT_INITIAL_CAPACITY
```
    /**
     * The default initial capacity - MUST be a power of two.
     */
    //默认的初始化容量，官方要求必须是2的幂，此处为位运算 1 << 4 == 16
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
   
```
### MAXIMUM_CAPACITY
```
    /**
     * The maximum capacity, used if a higher value is implicitly specified
     * by either of the constructors with arguments.
     * MUST be a power of two <= 1<<30.
     */
    //最大容量为2的30次方
    static final int MAXIMUM_CAPACITY = 1 << 30;
```
### DEFAULT_LOAD_FACTOR
```
    /**
     * The load factor used when none specified in constructor.
     */
    //默认负载因子 为0.75,与后续扩容相关
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
```
### TREEIFY_THRESHOLD
```
    /**
     * The bin count threshold for using a tree rather than list for a
     * bin.  Bins are converted to trees when adding an element to a
     * bin with at least this many nodes. The value must be greater
     * than 2 and should be at least 8 to mesh with assumptions in
     * tree removal about conversion back to plain bins upon
     * shrinkage.
     */
    //java8中，hashmap采用数组+链表+红黑树的数据结构，
    //当每个Node节点的链表长度大于等于8时，则会自动将链表转换成红黑树结构，
    //以便提升查询效率(o(log(n)))
    //在put（merge、compute等）方法中判断
    static final int TREEIFY_THRESHOLD = 8;
```
### UNTREEIFY_THRESHOLD
```
    /**
     * The bin count threshold for untreeifying a (split) bin during a
     * resize operation. Should be less than TREEIFY_THRESHOLD, and at
     * most 6 to mesh with shrinkage detection under removal.
     */
    //当树节点小于等于6时转回链表
    //在resize方法中可能判断
    static final int UNTREEIFY_THRESHOLD = 6;
```

## 内部类Node
```
    /**
     * Basic hash bin node, used for most entries.  (See below for
     * TreeNode subclass, and in LinkedHashMap for its Entry subclass.)
     */
    static class Node<K,V> implements Map.Entry<K,V> {
        //本身会存储key，key的hash值，value和下一个node节点
        final int hash;
        final K key;
        V value;
        Node<K,V> next; //此处链表结构为尾插法

        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }

        public final K getKey()        { return key; }
        public final V getValue()      { return value; }
        public final String toString() { return key + "=" + value; }
        
        //重写了hashCode方法
        public final int hashCode() {
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }

        public final V setValue(V newValue) {
            V oldValue = value;
            value = newValue;
            return oldValue;
        }
        
        /**
         *重写了equals方法
         */
        public final boolean equals(Object o) {
            if (o == this)
                return true;
            //如果是Map.Entry类型，且内部的k/v通过Objects的equals方法比较为true
            if (o instanceof Map.Entry) {
                Map.Entry<?,?> e = (Map.Entry<?,?>)o;
                if (Objects.equals(key, e.getKey()) &&
                    Objects.equals(value, e.getValue()))
                    return true;
            }
            return false;
        }
    }
```
## 方法介绍
### hash

```
    //当key为null时，hash落在0的位置
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```
### tableSizeFor

```
    保证初始化大小始终为>=cap的2的幂
    5 -> 8
    11 -> 16
    static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
```
### HashMap构造器

```
    public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
        //此处参照上面tableSizeFor所述
        this.threshold = tableSizeFor(initialCapacity);
    }

    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }

    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }
```
> ### put & putVal

```
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }
    
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        //table为Node数组
        //n为数组长度
        //如果当前table为空或者长度为0，则会扩容resize
        //p为根据hash后值找到的数组中的节点
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        //如果hash运算后发现节点为null，则会新建一个节点给到对应hash的数组位置
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        //如果已经存在该节点
        else {
            Node<K,V> e; K k;
            //如果p头节点的hash，key值相同，则e = p
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            //如果p为树节点，则按照putTreeVal得到e
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            //否则开始遍历链表，找到对应的节点e
            else {
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            /**
             *当key存在：替换节点上的value，返回oldValue
             */
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        //判断当前size是否超过阈值
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        //如果hash后得到的数组位置没有值，则put后会返回null
        return null;
    }
```

> ### get & getNode

```
    //return null/e.value
    public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }
    
    final Node<K,V> getNode(int hash, Object key) {
        /**
         *如果node数组不为空 && node数组的长度>0 && node数组的hash值位置不为空，则根据key,找到对应node
         *first = tab[(n - 1) & hash]) != null这里不知道为什么要求与，但是结果始终是hash的int值
         */
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            if ((e = first.next) != null) {
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }
```

> ### resize

```
 final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        //原Node数组长度大于0时：
        if (oldCap > 0) {
            //如果已经大于了最大容量，那就只能修改threshold了
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            //如果原node数组大于16且扩容至2倍小于最大容量则成倍扩容
            //newCap和newThr都会翻倍
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
        //先判断threshold是不是大于0，会将threshold的值给到newCap
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
        //否则的话，就用默认的初始化值了
        //此处可以看到threshold的值 = DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY
        else {               // zero initial threshold signifies using defaults
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        //如果newThr=0，则会再判断一次
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
        //新建一个扩容后大小的Node数组
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    //如果当前node节点没有指向的next，就直接把当前节点复制给新node节点
                    if (e.next == null)
                        //这里因为newCap已经变了，所以新数据的hash值也随之发生了变化
                        //在node数组的位置是根据(length - 1) & hash 计算的
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    //扩容时维持原有链表顺序
                    else { // preserve order
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
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

### 线程安全问题
#### resize
```
java7中，链表采用头插法，如果多线程同时向需要扩容的hashmap中插入数据，
如果在某一线程在resize时挂起：
    1.原本期望的A->B->C,有可能根据扩容后length变化导致重新计算的hash值与之前不同，从而C到了新的节点上；
    2.同上，原本的A->B,可能变成B->A,那多线程调度结束，就有可能变成 B->A->B，无限循环
    
java8中，虽然用尾插法，且会保持原有链表顺序，但是对于put/get方法，由于没有加锁，还是可能会出现put/get取值不一的情况。此时就要引入juc包中的conCurrentHashMap了。
```