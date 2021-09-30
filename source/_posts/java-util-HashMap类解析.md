---
title: java.util.HashMap类解析
date: 2019-04-16 21:22:49
categories: 
  - java
tags: 
  - little_eight
---
## 基本知识

Hash表也称为散列表，也有直接译作哈希表，Hash表是一种根据关键字值（key-value）而直接进行访问的数据结构。也就是说它通过把关键码值映射到表中的一个位置来访问记录，以此来加快查找的速度。在链表、数组等数据结构中，查找某个关键字，通常要遍历整个数据结构，也就是O(N)的时间级，但是对于哈希表来说，只是O(1)的时间级。（更多知识自行百度Google）

HashMap是一个利用哈希表原理来存储元素的无序不安全键值都可为空的集合。遇到冲突时，HashMap 是采用的链地址法来解决，在 JDK1.7 中，HashMap 是由 数组+链表构成的。但是在JDK1.8中，HashMap是由数组+链表+红黑树构成，新增了红黑树作为底层数据结构，结构变得复杂了，但是效率也变的更高效。


```
    /**
     * 继承 AbstractMap
     *          AbstractMap也是实现了Map类的，既继承AbstractMap又实现Map，脱裤子放屁....
     * 实现 Map 接口
     *          这个接口是 Map 类集合的上层接口，定义了实现该接口的类都必须要实现的一组方法
     * 实现 Cloneable 接口
     *          这个类是 java.lang.Cloneable，前面我们讲解深拷贝和浅拷贝的原理时，
     *          我们介绍了浅拷贝可以通过调用 Object.clone() 方法来实现，
     *          但是调用该方法的对象必须要实现 Cloneable 接口，
     *          否则会抛出 CloneNoSupportException异常。
     * 实现 Serializable 接口
     *          序列化
     */
    public class HashMap<K,V> extends AbstractMap<K,V>
        implements Map<K,V>, Cloneable, Serializable {}
        
```
<!--more-->
## 成员变量

```
 /**
     * 初始容量为16，容量都必须是偶数
     */
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

    /**
     * 集合的最大容量，如果通过带参构造指定的最大容量超过此数，默认还是使用此数
     */
    static final int MAXIMUM_CAPACITY = 1 << 30;

    /**
     * 默认的填充因子
     */
    static final float DEFAULT_LOAD_FACTOR = 0.75f;

    /**
     * 当桶(bucket)上的结点数大于这个值时会转成红黑树
     */
    static final int TREEIFY_THRESHOLD = 8;

    /**
     * 当桶(bucket)上的节点数小于这个值时会转成链表
     */
    static final int UNTREEIFY_THRESHOLD = 6;

    /**
     * 当集合中的容量大于这个值时，表中的桶才能进行树形化 ，否则桶内元素太多时会扩容，
     * 而不是树形化 为了避免进行扩容、树形化选择的冲突，这个值不能小于 4 * TREEIFY_THRESHOLD
     */
    static final int MIN_TREEIFY_CAPACITY = 64;

    /**
     * 存链表的数组，初始化使用，长度总是 2的幂
     */
    transient Node<K,V>[] table;

    /**
     * 保存缓存的entrySet（）
     */
    transient Set<Map.Entry<K,V>> entrySet;

    /**
     * 此映射中包含的键值映射的数量。（集合存储键值对的数量）
     */
    transient int size;

    /**
     * 跟前面ArrayList和LinkedList集合中的字段modCount一样，记录集合被修改的次数
     * 主要用于迭代器中的快速失败
     */
    transient int modCount;

    /**
     * 调整大小的下一个大小值（容量*加载因子）,相当于阈值。计算公式：capacity * loadFactor。
     * 这个值是当前已占用数组长度的最大值。过这个数目就重新resize(扩容)，
     * 扩容后的 HashMap 容量是之前容量的两倍
     */
    int threshold;

    /**
     * 散列表的加载因子，是用来衡量HashMap满的程度，
     * 计算HashMap的实时装载因子的方法为：size/capacity，而不是占用桶的数量去除以capacity。
     * capacity 是桶的数量，也就是 table 的长度length。
     */
    final float loadFactor;
```

```
    /**
     * 链表的类
     */
    static class Node<K,V> implements Map.Entry<K,V> {
        // hash值
        final int hash;
        final K key;
        V value;
        // 下一个链表
        Node<K,V> next;

        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }

        public final K getKey()        { return key; }
        public final V getValue()      { return value; }
        public final String toString() { return key + "=" + value; }

        public final int hashCode() {
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }

        public final V setValue(V newValue) {
            V oldValue = value;
            value = newValue;
            return oldValue;
        }

        public final boolean equals(Object o) {
            if (o == this)
                return true;
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

## 构造函数

```
    /**
     * 默认构造函数，初始化加载因子loadFactor = 0.75
     */
    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR;
    }
    
    /**
     * 指定容量，实现看下一个方法
     */
    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }


    /**
     * 指定Map集合构造函数。
     */
    public HashMap(Map<? extends K, ? extends V> m) {
        this.loadFactor = DEFAULT_LOAD_FACTOR;
        // 就是把这个集合put进去，后面再详细说
        putMapEntries(m, false);
    }

    /**
     * 指定容量和加载因子
     */
    public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        // 最大容量只会设置为MAXIMUM_CAPACITY                                       
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
        // 调整下一个扩容值
        this.threshold = tableSizeFor(initialCapacity);
    }
    
        /**
         * 返回给定目标容量的二次幂，
         * | 位或运算
         */
      static final int tableSizeFor(int cap) {
        int n = cap - 1;
        // 也就是 n = n | n >>> xx
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }

```
> 位或运算  （|）
0|0=0；   0|1=1；   1|0=1；    1|1=1
即：两位只要有一个为1，其值为1，其它都为0。
扩展：异或运算符（^），后面计算hash值用到
0^0=0；  0^1=1；  1^0=1；   1^1=0；
即：参加运算的两个对象，如果两个相应位为“异”（值不同），则该位结果为1，否则为0。
扩展：位与运算（&），后面的取模运算用到
0&0=0;   0&1=0;    1&0=0;     1&1=1
即：两位同时为1，结果才为1，否则为0。

## 方法

### capacity
```
    /**
     *  返回当前容量
     */
    final int capacity() {
        return (table != null) ? table.length :
            (threshold > 0) ? threshold :
            DEFAULT_INITIAL_CAPACITY;
    }
```

### clear
```
    /**
     * 把所有链表都设置为null，清空hashMap
     */
    public void clear() {
        Node<K,V>[] tab;
        modCount++;
        if ((tab = table) != null && size > 0) {
            size = 0;
            for (int i = 0; i < tab.length; ++i)
                tab[i] = null;
        }
    }
```

### clone
```
    /**
     * 浅拷贝
     */
        public Object clone() {
        HashMap<K,V> result;
        try {
            result = (HashMap<K,V>)super.clone();
        } catch (CloneNotSupportedException e) {
            // this shouldn't happen, since we are Cloneable
            throw new InternalError(e);
        }
        // 把关键成员变量都设置成null
        result.reinitialize();
        result.putMapEntries(this, false);
        return result;
    }
```

### containsKey
```
    /**
     * 判断是否包含此key
     */
    public boolean containsKey(Object key) {
        // 下面再详细说getNode
        return getNode(hash(key), key) != null;
    }
```

### entrySet
```
    /**
     * 返回this.entrySet，没有就new一个
     */
    public Set<Map.Entry<K,V>> entrySet() {
        Set<Map.Entry<K,V>> es;
        return (es = entrySet) == null ? (entrySet = new EntrySet()) : es;
    }
    
    /**
     * 内部EntrySet类
     */
    final class  extends AbstractSet<Map.Entry<K,V>> {
        public final int size()                 { return size; }
        public final void clear()               { HashMap.this.clear(); }
        public final Iterator<Map.Entry<K,V>> iterator() {
            return new EntryIterator();
        }
        public final boolean contains(Object o) {
            if (!(o instanceof Map.Entry))
                return false;
            Map.Entry<?,?> e = (Map.Entry<?,?>) o;
            Object key = e.getKey();
            Node<K,V> candidate = getNode(hash(key), key);
            return candidate != null && candidate.equals(e);
        }
        public final boolean remove(Object o) {
            if (o instanceof Map.Entry) {
                Map.Entry<?,?> e = (Map.Entry<?,?>) o;
                Object key = e.getKey();
                Object value = e.getValue();
                return removeNode(hash(key), key, value, true, true) != null;
            }
            return false;
        }
        public final Spliterator<Map.Entry<K,V>> spliterator() {
            return new EntrySpliterator<>(HashMap.this, 0, -1, 0, 0);
        }
        public final void forEach(Consumer<? super Map.Entry<K,V>> action) {
            Node<K,V>[] tab;
            if (action == null)
                throw new NullPointerException();
            if (size > 0 && (tab = table) != null) {
                int mc = modCount;
                for (int i = 0; i < tab.length; ++i) {
                    for (Node<K,V> e = tab[i]; e != null; e = e.next)
                        action.accept(e);
                }
                if (modCount != mc)
                    throw new ConcurrentModificationException();
            }
        }
    }
```

### get、getNode、getOrDefault
```
    /**
     * 根据key返回value
     */
    public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }
    
    /**
     * 获取对应链表
     */
    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        // 数组不为空，且hash对应的链表头也不为空
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            // 校验是否是链表头的，不是的话，就next遍历下去找到对应的链表
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            if ((e = first.next) != null) {
                // 如果是红黑树，就去getTreeNode，红黑树的部分看最后面TreeNode类解析
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
    
    
    /**
     * 如果取到的链表为null，则返回设定的defaultValue，否则就返回链表的value
     */
    public V getOrDefault(Object key, V defaultValue，) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? defaultValue : e.value;
    }
```
> 取模运算 (n - 1) & hash ，在计算机中 & 的效率比 % 高很多，所以采用&
且有这么一条结论，当 lenth = 2n 时，X % length = X & (length - 1)
具体可以去看下这篇博客分析https://blog.csdn.net/ysvae/article/details/81090894


### hash
```
    /**
     * 获取hash值，
     * (h = key.hashCode()) ^ (h >>> 16)，这段代码叫“扰动函数”，具体可以看这篇博客解析
     * https://www.cnblogs.com/zhengwang/p/8136164.html
     */
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```

### isEmpty
```
    /**
     * 根据size判断集合是否为空
     */
    public boolean isEmpty() {
        return size == 0;
    }
```

### keySet
```
    /**
     * 返回所有key的Set集合
     */
    public Set<K> keySet() {
        Set<K> ks = keySet;
        if (ks == null) {
            ks = new KeySet();
            keySet = ks;
        }
        return ks;
    }
    /**
     * 内部KeySet类
     */
    final class KeySet extends AbstractSet<K> {
        public final int size()                 { return size; }
        public final void clear()               { HashMap.this.clear(); }
        public final Iterator<K> iterator()     { return new KeyIterator(); }
        public final boolean contains(Object o) { return containsKey(o); }
        public final boolean remove(Object key) {
            return removeNode(hash(key), key, null, false, true) != null;
        }
        public final Spliterator<K> spliterator() {
            return new KeySpliterator<>(HashMap.this, 0, -1, 0, 0);
        }
        public final void forEach(Consumer<? super K> action) {
            Node<K,V>[] tab;
            if (action == null)
                throw new NullPointerException();
            if (size > 0 && (tab = table) != null) {
                int mc = modCount;
                for (int i = 0; i < tab.length; ++i) {
                    for (Node<K,V> e = tab[i]; e != null; e = e.next)
                        action.accept(e.key);
                }
                if (modCount != mc)
                    throw new ConcurrentModificationException();
            }
        }
    }
     
```

### put、putIfAbsent、putVal
```
    /**
     * 设值
     */
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    } 
    
    /**
     * 如果存在此key的值，那就不设值用原值
     */
    public V putIfAbsent(K key, V value) {
        return putVal(hash(key), key, value, true, true);
    }
    
    /**
     * 设值
     * @param onlyIfAbsent 为true，就不改动原值
     * @param evict 如果为false，则表处于创建模式
     */
       final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        // 如果数组为null，也就是第一次put值，就resize进行扩容，这个方法后面再讲
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        // 在数组中这个下标下如果这个链表为null，就根据传进的参数新建一个给它，也就是新建成链表头
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            // 链表头不为null，e、k会当做循环这条链表的暂时值
            Node<K,V> e; K k;
            // 如果刚好就是链表头，那就是e等于链表头
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            // 红黑树的部分看TreeNode类解析
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                // 循环这条链表
                for (int binCount = 0; ; ++binCount) {
                    // next==null说明到链表尾了
                    if ((e = p.next) == null) {
                        // 根据参数新建一个链表，作为链表尾
                        p.next = newNode(hash, key, value, null);
                        // 如果链表长度不小于TREEIFY_THRESHOLD（8），就转换成红黑树
                        if (binCount >= TREEIFY_THRESHOLD - 1)
                            // 如何转换的，这个后面再详解
                            treeifyBin(tab, hash);
                        break;
                    }
                    // 如果在循环过程中发现存在此key，那就不用新建链表尾了
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            // 如果设值的key是已存在的
            if (e != null) {
                V oldValue = e.value;
                // onlyIfAbsent为false，或者原值等于null，就设置成传入参数的value
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                // 这里是空方法，在LinkedHashMap才会实现
                afterNodeAccess(e);
                // 返回的旧值哦
                return oldValue;
            }
        }
        ++modCount;
        // 超过最大容量，进行扩容
        if (++size > threshold)
            resize();
        // 这里是空方法，在LinkedHashMap才会实现
        afterNodeInsertion(evict);
        return null;
    }
```

### remove、removeNode
```
    /**
     * 移除某个对应的值（链表），返回对应的value，没的话就返回null
     */
    public V remove(Object key) {
        Node<K,V> e;
        return (e = removeNode(hash(key), key, null, false, true)) == null ?
            null : e.value;
    }
    
    /**
     * 移除某个对应的值（链表），有对应的value就返回true，没的话就返回false
     */
    public boolean remove(Object key, Object value) {
        return removeNode(hash(key), key, value, true, true) != null;
    }
    
    /**
     * 移除某个对应的值（链表）
     * @param matchValue 为true时，进行移除
     * @param movable 如果为false，则在删除时不要移动其他节点
     */
    final Node<K,V> removeNode(int hash, Object key, Object value,
                               boolean matchValue, boolean movable) {
        Node<K,V>[] tab; Node<K,V> p; int n, index;
        // 数组不为空，且hash对应的链表头也不为空
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (p = tab[index = (n - 1) & hash]) != null) {
            Node<K,V> node = null, e; K k; V v;
            // 这里就是为了得到key对应的链表node
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                node = p;
            else if ((e = p.next) != null) {
                if (p instanceof TreeNode)
                    node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
                else {
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
            // 是否符合移除该node的条件
            if (node != null && (!matchValue || (v = node.value) == value ||
                                 (value != null && value.equals(v)))) {
                // 如果是红黑树，就去TreeNode的方法处理
                if (node instanceof TreeNode)
                    ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
                // 两者相等，说明node刚好是链表头，直接等于next即可
                else if (node == p)
                    tab[index] = node.next;
                else
                    // 不是链表头，改p的next，相当于移除下一位的node
                    p.next = node.next;
                ++modCount;
                --size;
                // 这里是空方法，在LinkedHashMap才会实现
                afterNodeRemoval(node);
                return node;
            }
        }
        return null;
    }
```

### replace、replaceAll
```
    /**
     * 根据key替换对应value，返回旧值
     */
    public V replace(K key, V value) {
        Node<K,V> e;
        if ((e = getNode(hash(key), key)) != null) {
            V oldValue = e.value;
            e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
        return null;
    }
    
    /**
     * 根据key，newValue替换旧值
     * 当旧值等于oldValue时，才会去替换，否则返回false
     */
    public boolean replace(K key, V oldValue, V newValue) {
        Node<K,V> e; V v;
        if ((e = getNode(hash(key), key)) != null &&
            ((v = e.value) == oldValue || (v != null && v.equals(oldValue)))) {
            e.value = newValue;
            afterNodeAccess(e);
            return true;
        }
        return false;
    } 
    
    /**
     * 这个不常用，不解析了..
     */
    public void replaceAll(BiFunction<? super K, ? super V, ? extends V> function) {
        Node<K,V>[] tab;
        if (function == null)
            throw new NullPointerException();
        if (size > 0 && (tab = table) != null) {
            int mc = modCount;
            for (int i = 0; i < tab.length; ++i) {
                for (Node<K,V> e = tab[i]; e != null; e = e.next) {
                    e.value = function.apply(e.key, e.value);
                }
            }
            if (modCount != mc)
                throw new ConcurrentModificationException();
        }
    }
```

### resize
```
    /**
     * 扩容
     */
    final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        // 原数组的长度，也就是旧容量
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        // 旧阀值
        int oldThr = threshold;
        // 新容量、新阀值
        int newCap, newThr = 0;
        if (oldCap > 0) {
            // 如果原数组已经是最大了，就不扩容了，返回原数组
            if (oldCap >= MAXIMUM_CAPACITY) {
                // 修改阈值为int的最大值(2^31-1)，这样以后就不会扩容了
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            // 否则就扩容一倍
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1;
        }
        // 旧阀值大于0，则将新容量直接等于就阀值 
        else if (oldThr > 0)
            newCap = oldThr;
        // 旧容量、旧阀值都为0，也就是未初始化，就设置新容量、新阀值为默认值
        else {               
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        // 如果新阀值为0，计算新的阀值上限
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
            // 新建扩容一倍的新数组
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        if (oldTab != null) {
            // 遍历旧数组，把它们的链表搬到新数组
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    // 便于gc回收
                    oldTab[j] = null;
                    // 这条链表只有一个链表头
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    // 是红黑树，就红黑树处理
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    // 遍历链表条
                    else {
                        // loHead - loTail、hiHead - hiTail，分别对应 链表头 - 链表尾
                        // 这样作为暂时值处理，得到链表尾
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            // 原索引（没懂。。）
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
                        // loTail链表尾不为null，
                        // 就设链表头尾到新数组对应的下标（也就是转移对应的链表条）
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        // hiTail链表尾不为null，
                        // 就设链表头尾到新数组对应的下标（也就是转移对应的链表条）
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

### values
```
    /**
     * 返回所有value值的Collection集合，（莫名奇妙就有了这个值。。。）
     */
    public Collection<V> values() {
        Collection<V> vs = values;
        if (vs == null) {
            vs = new Values();
            values = vs;
        }
        return vs;
    }
```


## TreeNode
比较特殊，专门来讲的红黑树类
```
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {}
```
然后看下LinkedHashMap.Entry<K,V>类，居然是继承回HashMap的Node类。。有点意思
```
 static class Entry<K,V> extends HashMap.Node<K,V> {}
```
### 基本知识
红黑树的概念
* 每个结点都是红色的或者是黑色的
* 根结点是黑色的
* 每个叶结点NIL是黑色的，但是通常我们不考虑NIL叶结点。
* 如果一个结点是红色的，它的两个子结点都是黑色的
* 每个结点到其他所有后代叶结点的简单路径上，均包含相同数目的黑色结点，这个属性被称为黑高，记作bh(x)


### 成员变量

```
        // 父节点
        TreeNode<K,V> parent;
        // 左子节点
        TreeNode<K,V> left;
        // 右子节点
        TreeNode<K,V> right;
        // 前方节点
        TreeNode<K,V> prev;
        // 是否红色
        boolean red;
```

### 构造函数

```
    /**
     * 就是用HashMap.Node的构造函数
     */
    TreeNode(int hash, K key, V val, Node<K,V> next) {
        super(hash, key, val, next);
    }
```

这里解析的方法从HashMap的get、put等入手


### getTreeNode
```
    /**
     * 得到对应的节点
     */
    final TreeNode<K,V> getTreeNode(int h, Object k) {
        return ((parent != null) ? root() : this).find(h, k, null);
    }
    
    
```

### root
```
    /**
     * 返回根节点
     */
     final TreeNode<K,V> root() {
            for (TreeNode<K,V> r = this, p;;) {
                if ((p = r.parent) == null)
                    return r;
                r = p;
            }
        }
```

### find
```
    /**
     * 从根结点p开始根据hash和key值寻找指定的结点。二叉树的查找
     */
      final TreeNode<K,V> find(int h, Object k, Class<?> kc) {
            TreeNode<K,V> p = this;
            // 遍历红黑树
            do {
                int ph, dir; K pk;
                TreeNode<K,V> pl = p.left, pr = p.right, q;
                // p.hash大于参数hash时，移向左子树
                if ((ph = p.hash) > h)
                    p = pl;
                // p.hash小于参数hash时，移向右子树
                else if (ph < h)
                    p = pr;
                // p.hash等于参数hash，且参数k也等于p.key，返回这个p，也就是根节点
                else if ((pk = p.key) == k || (k != null && k.equals(pk)))
                    return p;
                // 若hash相等但key不等，向左右子树非空的一侧移动
                else if (pl == null)
                    p = pr;
                else if (pr == null)
                    p = pl;
                else if ((kc != null ||
                          (kc = comparableClassFor(k)) != null) && // kc是否是一个可比较的类
                         (dir = compareComparables(kc, k, pk)) != 0)  // 比较k和p.key
                    p = (dir < 0) ? pl : pr;
                else if ((q = pr.find(h, k, kc)) != null)
                    return q;
                else
                    p = pl;
            } while (p != null);
            return null;
        }
```

### putTreeVal
```
    /**
     * 
     */
  final TreeNode<K,V> putTreeVal(HashMap<K,V> map, Node<K,V>[] tab,
                                       int h, K k, V v) {
            Class<?> kc = null;
            boolean searched = false;
            // 根节点
            TreeNode<K,V> root = (parent != null) ? root() : this;
            // 
            for (TreeNode<K,V> p = root;;) {
                int dir, ph; K pk;
                if ((ph = p.hash) > h)
                    dir = -1;
                else if (ph < h)
                    dir = 1;
                else if ((pk = p.key) == k || (k != null && k.equals(pk)))
                    return p;
                else if ((kc == null &&
                          (kc = comparableClassFor(k)) == null) ||
                         (dir = compareComparables(kc, k, pk)) == 0) {
                    if (!searched) {
                        TreeNode<K,V> q, ch;
                        searched = true;
                        if (((ch = p.left) != null &&
                             (q = ch.find(h, k, kc)) != null) ||
                            ((ch = p.right) != null &&
                             (q = ch.find(h, k, kc)) != null))
                            return q;
                    }
                    dir = tieBreakOrder(k, pk);
                }

                TreeNode<K,V> xp = p;
                if ((p = (dir <= 0) ? p.left : p.right) == null) {
                    Node<K,V> xpn = xp.next;
                    TreeNode<K,V> x = map.newTreeNode(h, k, v, xpn);
                    if (dir <= 0)
                        xp.left = x;
                    else
                        xp.right = x;
                    xp.next = x;
                    x.parent = x.prev = xp;
                    if (xpn != null)
                        ((TreeNode<K,V>)xpn).prev = x;
                    moveRootToFront(tab, balanceInsertion(root, x));
                    return null;
                }
            }
        }
```

### treeifyBin
```
    /**
     * 转换成红黑树，但其实只是链表转成TreeNode，里面的treeify（）才是对链表条转换成红黑树
     */
  final void treeifyBin(Node<K,V>[] tab, int hash) {
        int n, index; Node<K,V> e;
        // 数组为空，或者数组长度小于64，只扩容不转换
        if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
            resize();
        // 跳到该链表条的链表头，准备循环
        else if ((e = tab[index = (n - 1) & hash]) != null) {
            // 定义头尾节点，因为一直遍历，尾节点的值会一直变化
            TreeNode<K,V> hd = null, tl = null;
            do {
                // 这里replacementTreeNode就是new TreeNode<>(e.hash, e.key, e.value, null)
                // p为当前节点
                TreeNode<K,V> p = replacementTreeNode(e, null);
                // 尾节点为null，说明是初始化阶段，把头节点=p
                if (tl == null)
                    hd = p;
                // 尾节点不为null，设置p的prev（上一个节点）为尾节点，尾节点的next为p，
                // 也就是把p接到当前暂时的尾节点后面
                else {
                    p.prev = tl;
                    tl.next = p;
                }
                // 把p设为暂时尾节点
                tl = p;
            } while ((e = e.next) != null);
            // 上面只是把链表转换成TreeNode，现在才进行链表条转换成红黑树
            if ((tab[index] = hd) != null)
                // 看下面解析
                hd.treeify(tab);
        }
    }
```

### treeify
```
    /**
     * 把链表条转换成红黑树
     */
  final void treeify(Node<K,V>[] tab) {
            TreeNode<K,V> root = null;
            // 从this开始遍历，x为当前节点，next为当前节点的下一节点
            for (TreeNode<K,V> x = this, next; x != null; x = next) {
                next = (TreeNode<K,V>)x.next;
                x.left = x.right = null;
                // 如果还没有根节点，把x设为跟节点
                if (root == null) {
                    x.parent = null;
                    x.red = false;
                    root = x;
                }
                else {
                    K k = x.key;
                    int h = x.hash;
                    Class<?> kc = null;
                    /*
                     * 从根节点开始遍历，此遍历没有设置边界，只能从内部跳出
                     * x为要插进的节点，p为遍历到的要进行对比的节点
                     */
                    for (TreeNode<K,V> p = root;;) {
                        // dir 标识方向（-1为左，1为右）、ph标识当前树节点的hash值
                        int dir, ph;
                        K pk = p.key;
                        if ((ph = p.hash) > h)
                            dir = -1;
                        else if (ph < h)
                            dir = 1;
                        // 对比的两个节点x、p的hash值相等，判断hash碰撞再决定dir的值
                        else if ((kc == null &&
                                  (kc = comparableClassFor(k)) == null) ||
                                 (dir = compareComparables(kc, k, pk)) == 0)
                            dir = tieBreakOrder(k, pk);
                        // 先保存好当前对比的p，可能要作为x即将插入的父节点
                        TreeNode<K,V> xp = p;
                        // 根据dir来决定下一个遍历的对比节点p，如果p为null说明到末节点了，把x插进去
                      
                        if ((p = (dir <= 0) ? p.left : p.right) == null) {
                            // x的父节点
                            x.parent = xp;
                            // 决定x在左还是右节点
                            if (dir <= 0)
                                xp.left = x;
                            else
                                xp.right = x;
                            // 加入新的节点，要对红黑树进行重新平衡，这个下面分析
                            root = balanceInsertion(root, x);
                            break;
                        }
                    // p不为null，就重新开始整个遍历
                    }
                }
            }
            // 把红黑树的根节点设为其所在的链表条的链表头，下面会有分析
            moveRootToFront(tab, root);
        }
```

### balanceInsertion
```
    /**
     * 平衡红黑树，root为根节点，x为新插入的节点
     * 有兴趣可以看这篇图文讲解，https://www.cnblogs.com/oldbai/p/9890808.html
     * 还有这篇 https://blog.csdn.net/weixin_42340670/article/details/80550932
     */
  static <K,V> TreeNode<K,V> balanceInsertion(TreeNode<K,V> root,
                                                    TreeNode<K,V> x) {
            // 设x为红色
            x.red = true;
            // 循环依然只能内部跳出
            // xp：x的父节点、xpp：x的爷爷节点、xppl：x的左叔叔节点、xppr：x的右叔叔节点
            for (TreeNode<K,V> xp, xpp, xppl, xppr;;) {
                // xp为x的父节点，为null说明x为根节点，设成黑色，返回x
                if ((xp = x.parent) == null) {
                    x.red = false;
                    return x;
                }
                // xp为黑色或者x的爷爷节点xpp为null，返回root
                else if (!xp.red || (xpp = xp.parent) == null)
                    return root;
                // xpp不为null，xp与xpp的左子节点相等，则说明xp在xpp的左子节点
                if (xp == (xppl = xpp.left)) {
                    // 如果 xpp 的右子节点xppr不为空且为红色，进行变色操作
                    // 这里直接变色可达到关于黑色数量规则，且无连续红色，
                    // 然后继续往上循环判断
                    if ((xppr = xpp.right) != null && xppr.red) {
                        xppr.red = false;
                        xp.red = false;
                        xpp.red = true;
                        // x设为xpp，作为下次循环的指定节点
                        x = xpp;
                    }
                    // 如果 xpp 的右子节点xppr为空，或者为黑色，变色已经不能满足红黑树规则
                    else {
                        // 如果x为右子节点,左旋转
                        if (x == xp.right) {
                            root = rotateLeft(root, x = xp);
                            xpp = (xp = x.parent) == null ? null : xp.parent;
                        }
                        if (xp != null) {
                            xp.red = false;
                            if (xpp != null) {
                                xpp.red = true;
                                root = rotateRight(root, xpp);
                            }
                        }
                    }
                }
                else {
                    if (xppl != null && xppl.red) {
                        xppl.red = false;
                        xp.red = false;
                        xpp.red = true;
                        x = xpp;
                    }
                    else {
                        if (x == xp.left) {
                            root = rotateRight(root, x = xp);
                            xpp = (xp = x.parent) == null ? null : xp.parent;
                        }
                        if (xp != null) {
                            xp.red = false;
                            if (xpp != null) {
                                xpp.red = true;
                                root = rotateLeft(root, xpp);
                            }
                        }
                    }
                }
            }
        }
```
### rotateLeft   
```
    /**
     * 左旋转
     */
   static <K,V> TreeNode<K,V> rotateLeft(TreeNode<K,V> root,
                                              TreeNode<K,V> p) {
            TreeNode<K,V> r, pp, rl;
            if (p != null && (r = p.right) != null) {
                if ((rl = p.right = r.left) != null)
                    rl.parent = p;
                if ((pp = r.parent = p.parent) == null)
                    (root = r).red = false;
                else if (pp.left == p)
                    pp.left = r;
                else
                    pp.right = r;
                r.left = p;
                p.parent = r;
            }
            return root;
        }
```

### rotateRight
```
    /**
     * 右旋转
     */
    static <K,V> TreeNode<K,V> rotateRight(TreeNode<K,V> root,
                                               TreeNode<K,V> p) {
            TreeNode<K,V> l, pp, lr;
            if (p != null && (l = p.left) != null) {
                if ((lr = p.left = l.right) != null)
                    lr.parent = p;
                if ((pp = l.parent = p.parent) == null)
                    (root = l).red = false;
                else if (pp.right == p)
                    pp.right = l;
                else
                    pp.left = l;
                l.right = p;
                p.parent = l;
            }
            return root;
        }
```

方法有点难懂，深入研究红黑树后再补充。
