---
title: 数据结构-map-HashMap代码分析
date: 2021-07-11 23:15:50
tags: "hashMap"
category: "数据结构"
description: "对hashmap进行源码级别的复习"
---

来，好好学习hashMap。

# 基本介绍

## 构造以及初始成员变量

```
    /**
     * Constructs an empty <tt>HashMap</tt> with the default initial capacity
     * (16) and the default load factor (0.75).
     */
    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }
```
单纯的new HashMap()，只是设定了满载率，并未设定别的参数，满载率为0.75

```
    // (The javadoc description is true upon serialization.
    // Additionally, if the table array has not been allocated, this
    // field holds the initial array capacity, or zero signifying
    // DEFAULT_INITIAL_CAPACITY.)
    int threshold;
```
阈值初始是未赋值

```
    /**
     * The default initial capacity - MUST be a power of two.
     */
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
```
默认容积是16


## put(K key, V value)

```
    /**
     * Associates the specified value with the specified key in this map.
     * If the map previously contained a mapping for the key, the old
     * value is replaced.
     *
     * @param key key with which the specified value is to be associated
     * @param value value to be associated with the specified key
     * @return the previous value associated with <tt>key</tt>, or
     *         <tt>null</tt> if there was no mapping for <tt>key</tt>.
     *         (A <tt>null</tt> return can also indicate that the map
     *         previously associated <tt>null</tt> with <tt>key</tt>.)
     */
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }
```

put方法调用的是 

```
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
                   boolean evict)
```

### hash()

```
    /**
     * Computes key.hashCode() and spreads (XORs) higher bits of hash
     * to lower.  Because the table uses power-of-two masking, sets of
     * hashes that vary only in bits above the current mask will
     * always collide. (Among known examples are sets of Float keys
     * holding consecutive whole numbers in small tables.)  So we
     * apply a transform that spreads the impact of higher bits
     * downward. There is a tradeoff between speed, utility, and
     * quality of bit-spreading. Because many common sets of hashes
     * are already reasonably distributed (so don't benefit from
     * spreading), and because we use trees to handle large sets of
     * collisions in bins, we just XOR some shifted bits in the
     * cheapest possible way to reduce systematic lossage, as well as
     * to incorporate impact of the highest bits that would otherwise
     * never be used in index calculations because of table bounds.
     */
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```

hash的操作，直接使用的是key的hashcode和hashcode的右移16位做亦或运算。

其主要是为了扰乱hash，让更多二进制位参与到需要根据数组长度找索引的与运算，尽量让元素散落在数组不同的索引上面

### putVal(int hash, K key, V value, boolean onlyIfAbsent,boolean evict)

这个方法第一个参数是key的hash操作

第二个参数就是key

第三个参数是value

第四个参数代表当前这个key不存在数的时候，才更改，默认是覆盖。需要不覆盖的话需要使用 `putIfAbsent`

第五个参数代表到达上限时再添加是否需要删除最老的节点，hashmap没有用到，主要给子类linkedHashmap使用。

```
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        //这里进行了初始化table的操作
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        //这里是对hash进行 n-1长度的与运算，确保算下来的引用在[0, n-1]范围内
        if ((p = tab[i = (n - 1) & hash]) == null)
            //该索引上面的桶没东西，创建新节点
            tab[i] = newNode(hash, key, value, null);
        else {
            //该索引已经有节点了
            Node<K,V> e; K k;
            //该索引的节点的hash和传入的hash一样，且key也一样，拷贝一下老的节点
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            //该老节点不相等并且老节点已经是树节点，直接put新的节点进去
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            //两个节点不相等，且老节点不是树节点，此时根据该桶的大小，小于8直接尾部插入，否则树化
            else {
                for (int binCount = 0; ; ++binCount) {
                    //走到尾结点还没有找到，就尾插，因此不需要管onlyIfAbsent
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    //发现有一个节点和当前节点相等
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            //之前发现有节点的key是相同的，根据参数决定是否更改
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                //节点存在并且需要修改 或对应value为空，则插入
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                //afterNodeAccess给linkedhashmap使用的
                afterNodeAccess(e);
                return oldValue;
            }
        }
        //新节点导致加了一个空的桶被使用，或者尾插导致一个桶的key增加了，这时候增加操作数，并且增加整个hashmap的size，超过阈值就扩容
        ++modCount;
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```

![HashMapput图](/images/数据结构/hashMap的put操作.png)

#### resize()

resize相比较于1.7的代码，除了扩容以外，还多了一个初始化的操作

```
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
        //记录老的数组长度，如果是刚开始，则默认初始化长度为0
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        //记录老的阈值，如果刚开始未初始化，也是0
        int oldThr = threshold;
        int newCap, newThr = 0;
        //初始化的时候oldCap是0不会走这个条件
        if (oldCap > 0) {
            //数组长度打到上限了，阈值增加，直接返回。
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            //没有超过上限，通过位运算，数组长度变成原来的两倍，，如果新的长度没有达到上限并且老的长度也比16大，则阈值变成原来的两倍
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
        //初始化oldThr也是0，这个条件也进不去
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
        //初始化走的条件
        else {               // zero initial threshold signifies using defaults
            //数组长度赋值为16
            newCap = DEFAULT_INITIAL_CAPACITY;
            //新的阈值设置为 16 * 0.75 = 12
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        //初始化上面已经改了阈值，初始化不走这里
        //newThr被更改为0，重新设置新的阈值
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        //这里赋值阈值
        threshold = newThr;
        //这里创建新的node数组给table
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        //老的数组不为空，此时转移节点，初始化不走这里
        //正式的扩容拷贝
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                //e如果是null其实就跳过就行了
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    //当前这个老节点并没有下一位了，这种节点直接hash一下，覆盖即可
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    //树节点则走树节点的方式添加
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    //该节点并非树节点，并且还有下一位
                    else { // preserve order
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            //与老数组长度进行与运算，如果是低位，则往列表低位尾部插入
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            //与老数组长度进行与运算，算下来是低位，这时候往高位列表尾部插入
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        //插入完毕之后，开始走设置新节点的属性
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
        //返回直接的数组
        return newTab;
    }
```

## 部分总结

### hashMap无法保证线程安全

因为get 和put事实上都没有加锁
需要线程安全可以使用ConcurrentHashMap，hashTable或者Collections.SychronizedXXX方法进行加锁使用

### 初始容量是16

16主要是因为不至于太小一下子就扩容，也不至于太大造成浪费

### 大小是2的N次方

主要原因是因为计算桶位置的时候，公式是((N -1 ) & hash)，2的指数减去1，所有位都是1，与运算的时候计算结果可以保证对象的hash生成足够的散列

### 负载因子默认是0.75

设置过大会导致同一个桶的位置存放好多value，导致增加搜索时间，性能下降。设置过小会造成桶比较浪费，浪费空间。

### hashmap处理hash碰撞的方式

key相同的时候会替换key对应内容的最小值，key不相同则进行尾插插入到链表的后方，如果链表长度大于8且数组的长度大于64的时候，会树化，数组长度小于64，则单纯的扩容。




