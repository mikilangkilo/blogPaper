---
title: 数据结构-LruCache缓存源码解析
date: 2018-04-01 12:26:43
tag : "lrucache"
category : "数据结构"
description : "本文主要是lrucache的学习"
---

以前曾经在volley中使用networkimageview的时候使用过lrucache，当时以为这只是一个第三方的开源框架。昨天在网上找安卓的缓存框架的时候看到了lrucache，居然是android.util包下面的。所以这次就着源码看看是个怎么缓存的原理。

# 类解释

首先看一下这个类官方的解释。

 * A cache that holds strong references to a limited number of values. Each time
 * a value is accessed, it is moved to the head of a queue. When a value is
 * added to a full cache, the value at the end of that queue is evicted and may
 * become eligible for garbage collection.

这个类是一个用于对有限的值持有强引用的缓存类。

每次一个值被获取了，它将被挪到队列的头部。

每当一个值被加入完全缓存（full-cache,完全缓存模式，讲一个对象完全的加载到内存中，而非只加载其映射关系。对应的还有Partial Cache，部分缓存模式，部分缓存模式多数用于加载对象的部分，用于判断对象是否曾经加载过，或者是否需要再次加载），缓存队列末尾的值将会被驱逐，丢失了强引用的关系，就会变得可以被gc清除。

 * <p>If your cached values hold resources that need to be explicitly released,
 * override {@link #entryRemoved}.

如果希望避免部分资源被lrucache缓存，可以继承entryremoved

 * <p>If a cache miss should be computed on demand for the corresponding keys,
 * override {@link #create}. This simplifies the calling code, allowing it to
 * assume a value will always be returned, even when there's a cache miss.

如果需要对一个响应返回一个完整的缓存，但是这个缓存目前是缺失状态，可以继承create方法，这样可以使得针对这个响应总是可以有回应，即时目前是缓存缺失的状态，也可以构造一个回应。

 * <p>By default, the cache size is measured in the number of entries. Override
 * {@link #sizeOf} to size the cache in different units. For example, this cache
 * is limited to 4MiB of bitmaps:
 * <pre>   {@code
 *   int cacheSize = 4 * 1024 * 1024; // 4MiB
 *   LruCache<String, Bitmap> bitmapCache = new LruCache<String, Bitmap>(cacheSize) {
 *       protected int sizeOf(String key, Bitmap value) {
 *           return value.getByteCount();
 *       }
 *   }}</pre>

缓存的数量是通过对象的数量来确定的。继承sizeOf方法可以使用不同的单位来计算缓存。（该方法提供的例子是用于限制缓存，那缓存不够怎么办？一张bitmap一般有8m啊？事实上它并不会精确计算占用的内存，只能说你说a有3m，总共给30m的缓存的话，他就会放最多10个a，事实上a并不一定只有3m，当然这也没必要关心，在不同的地方自己精确赋一下就好了）

 * <p>This class is thread-safe. Perform multiple cache operations atomically by
 * synchronizing on the cache: <pre>   {@code
 *   synchronized (cache) {
 *     if (cache.get(key) == null) {
 *         cache.put(key, value);
 *     }
 *   }}</pre>

该类是线程安全的，主要是对缓存操作的部分都加了锁。

 * <p>This class does not allow null to be used as a key or value. A return
 * value of null from {@link #get}, {@link #put} or {@link #remove} is
 * unambiguous: the key was not in the cache.

该类不允许空指针被使用作为key或者value。

 * <p>This class appeared in Android 3.1 (Honeycomb MR1); it's available as part
 * of <a href="http://developer.android.com/sdk/compatibility-library.html">Android's
 * Support Package</a> for earlier releases.

该类在android3.1出现。

# 源码探究

```
 private final LinkedHashMap<K, V> map;
```

 使用了linkedHashMap的方法来存储数据。有向图就是强引用方式，key在，value在。

```
	/** Size of this cache in units. Not necessarily the number of elements. */
	private int size;
    private int maxSize;

    private int putCount;
    private int createCount;
    private int evictionCount;
    private int hitCount;
    private int missCount;
```

```
    /**
     * @param maxSize for caches that do not override {@link #sizeOf}, this is
     *     the maximum number of entries in the cache. For all other caches,
     *     this is the maximum sum of the sizes of the entries in this cache.
     */
    public LruCache(int maxSize) {
        if (maxSize <= 0) {
            throw new IllegalArgumentException("maxSize <= 0");
        }
        this.maxSize = maxSize;
        this.map = new LinkedHashMap<K, V>(0, 0.75f, true);
    }
```

构造方面需要给一个maxSize，该参数是用于在为定义sizeof的对象计算容量大小

```
    /**
     * Sets the size of the cache.
     *
     * @param maxSize The new maximum size.
     */
    public void resize(int maxSize) {
        if (maxSize <= 0) {
            throw new IllegalArgumentException("maxSize <= 0");
        }

        synchronized (this) {
            this.maxSize = maxSize;
        }
        trimToSize(maxSize);
    }

```

重新设计maxsize

```
    /**
     * Returns the value for {@code key} if it exists in the cache or can be
     * created by {@code #create}. If a value was returned, it is moved to the
     * head of the queue. This returns null if a value is not cached and cannot
     * be created.
     */
    public final V get(K key) {
        if (key == null) {
            throw new NullPointerException("key == null");
        }

        V mapValue;
        synchronized (this) {
            mapValue = map.get(key);
            if (mapValue != null) {
                hitCount++;
                return mapValue;
            }
            missCount++;
        }
         /*
         * Attempt to create a value. This may take a long time, and the map
         * may be different when create() returns. If a conflicting value was
         * added to the map while create() was working, we leave that value in
         * the map and release the created value.
         */

        V createdValue = create(key);
        if (createdValue == null) {
            return null;
        }

        synchronized (this) {
            createCount++;
            mapValue = map.put(key, createdValue);

            if (mapValue != null) {
                // There was a conflict so undo that last put
                map.put(key, mapValue);
            } else {
                size += safeSizeOf(key, createdValue);
            }
        }
        if (mapValue != null) {
            entryRemoved(false, key, createdValue, mapValue);
            return mapValue;
        } else {
            trimToSize(maxSize);
            return createdValue;
        }
    }
```

get的时候加锁，每次有返回的话，hitcount自增。没有返回代表是缺失缓存，misscount自增。

若找不到值，会调用createvalue来创建，创建成功，createcount自增，失败就代表没有该缓存内容，直接终端。创建成功同时加入到map里面，map的size会增加。

创建出来的mapvalue会执行entryremoved的检查，对有设置过不加入的参数进行操作。

```
    /**
     * Called for entries that have been evicted or removed. This method is
     * invoked when a value is evicted to make space, removed by a call to
     * {@link #remove}, or replaced by a call to {@link #put}. The default
     * implementation does nothing.
     *
     * <p>The method is called without synchronization: other threads may
     * access the cache while this method is executing.
     *
     * @param evicted true if the entry is being removed to make space, false
     *     if the removal was caused by a {@link #put} or {@link #remove}.
     * @param newValue the new value for {@code key}, if it exists. If non-null,
     *     this removal was caused by a {@link #put}. Otherwise it was caused by
     *     an eviction or a {@link #remove}.
     */
    protected void entryRemoved(boolean evicted, K key, V oldValue, V newValue) {}
```

entryRemoved的方法准确的说法是用于吊起被驱逐和移除的对象。确保key对应的value可以是正确的，newvalue为空的话就代表是移除老的对象。

```
    /**
     * Remove the eldest entries until the total of remaining entries is at or
     * below the requested size.
     *
     * @param maxSize the maximum size of the cache before returning. May be -1
     *            to evict even 0-sized elements.
     */
    public void trimToSize(int maxSize) {
        while (true) {
            K key;
            V value;
            synchronized (this) {
                if (size < 0 || (map.isEmpty() && size != 0)) {
                    throw new IllegalStateException(getClass().getName()
                            + ".sizeOf() is reporting inconsistent results!");
                }

                if (size <= maxSize) {
                    break;
                }

                Map.Entry<K, V> toEvict = map.eldest();
                if (toEvict == null) {
                    break;
                }

                key = toEvict.getKey();
                value = toEvict.getValue();
                map.remove(key);
                size -= safeSizeOf(key, value);
                evictionCount++;
            }

            entryRemoved(true, key, value, null);
        }
    }
```

trimToSize方法是针对目前队列中的对象，移除最老的。移除的操作就是移除了key，去掉强引用的部分。移除完之后要减去移除的大小，之后移除数自增，确保安全之后将之前的key，映射关系改为一个null，这个移除的数量并不是固定的，这是一个死循环，会移除到size小于maxsize，或者最老的是空。

```
    /**
     * Removes the entry for {@code key} if it exists.
     *
     * @return the previous value mapped by {@code key}.
     */
    public final V remove(K key) {
        if (key == null) {
            throw new NullPointerException("key == null");
        }

        V previous;
        synchronized (this) {
            previous = map.remove(key);
            if (previous != null) {
                size -= safeSizeOf(key, previous);
            }
        }

        if (previous != null) {
            entryRemoved(false, key, previous, null);
        }

        return previous;
    }
```
remove操作和之前的那个操作几乎一样，移除，如果曾经存在就在此赋空。

```
    /**
     * Clear the cache, calling {@link #entryRemoved} on each removed entry.
     */
    public final void evictAll() {
        trimToSize(-1); // -1 will evict 0-sized elements
    }

```

这是移除所有的元素

```
    @Override public synchronized final String toString() {
        int accesses = hitCount + missCount;
        int hitPercent = accesses != 0 ? (100 * hitCount / accesses) : 0;
        return String.format("LruCache[maxSize=%d,hits=%d,misses=%d,hitRate=%d%%]",
                maxSize, hitCount, missCount, hitPercent);
    }
```

toString方法可以看到状态。

```
    /**
     * Returns a copy of the current contents of the cache, ordered from least
     * recently accessed to most recently accessed.
     */
    public synchronized final Map<K, V> snapshot() {
        return new LinkedHashMap<K, V>(map);
    }

```

snapshot直接一个以自己map构造的对象。（这种避免返回自身的操作值得学习）

# 学习心得

整个LruCache的作用，说到底就是针对gc的树状搜索删除算法的一种方案。
但是我们平时使用的时候，需要针对几个地方进行定制，一是sizeof,二是removeentry。

整个缓存还是在内存中的，所以lrucache是一种内存缓存框架。

# 注意点

value最好放软应用对象，确保释放之后的第一次gc就可以回收。
设置maxsize，可以使用
```
Runtime.getRuntime().maxMemory()
```
也可针对不同情况进行设置。