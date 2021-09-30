---
title: java.util.LinkedHashMap类解析
date: 2019-04-22 21:27:25
categories: 
  - java
tags: 
  - little_eight
---


LinkedHashMap 是基于 HashMap实现的一种集合，有序，它单独维护了一个具有所有数据的双向链表，该链表保证了元素迭代的顺序。

```
/**
 * 继承HashMap，实现Map
 */
public class LinkedHashMap<K,V> extends HashMap<K,V> implements Map<K,V>{}
```

## 成员变量

```
 /**
     * 双向链表头
     */
    transient LinkedHashMap.Entry<K,V> head;

    /**
     * 双向链表尾
     */
    transient LinkedHashMap.Entry<K,V> tail;

    /**
     * 决定迭代排序方法：true是根据访问顺序来排序，false是根据插入顺序来排序
     */
    final boolean accessOrder;
```

## 构造函数
4种构造方法，都用HaspMap的实现方法。
```
    public LinkedHashMap() {
        super();
        accessOrder = false;
    }
    
    public LinkedHashMap(int initialCapacity) {
        super(initialCapacity);
        accessOrder = false;
    }
    public LinkedHashMap(int initialCapacity, float loadFactor) {
        super(initialCapacity, loadFactor);
        accessOrder = false;
    }

    public LinkedHashMap(int initialCapacity,
                         float loadFactor,
                         boolean accessOrder) {
        super(initialCapacity, loadFactor);
        this.accessOrder = accessOrder;
    }
```
<!--more-->

## 方法

### afterNodeAccess
> 这个方法除了在LinkedHashMap的get方法用到，还在HashMap的put、replace用到，因为这些方法是直接实现HashMap的，下面的afterNodeInsertion、afterNodeRemoval同理。

```
    /**
     *  把指定节点放到双向链表的尾部
     */
   void afterNodeAccess(Node<K,V> e) { // move node to last
        LinkedHashMap.Entry<K,V> last;
        // 当accessOrder为true，指定节点不是链表尾才有效
        if (accessOrder && (last = tail) != e) {
            // 把p当做指定节点，并注明它的前后节点 b，a
            LinkedHashMap.Entry<K,V> p =
                (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
            // 清除p的后节点指向
            p.after = null;
            /**
             * 下面会分别对a跟b判空，是为了把p提取出来同时，成功接上断掉的节点
             */
            // 如果b是空的，说明p是链表头，把head等于p的后节点
            if (b == null)
                head = a;
            // 如果b不为空，就让p的前节点的后节点指向p的后节点
            else
                b.after = a;
            // 如果a不为空，就让p的后节点的前节点指向p的前节点
            if (a != null)
                a.before = b;
            /**
             * 本人对下面注释可能有问题，理解不到位吧
             */
            // 如果a是空的，说明链表尾tail为null（不然就解释不通p不等于tail了），last等于p的前节点
            else
                last = b;
            // last为空 说明链表条还没有链表，而且p的前节点为null，链表头设置为p
            if (last == null)
                head = p;
            // last不为空，就把last跟p连到一起
            else {
                p.before = last;
                last.after = p;
            }
            // 最后把p设为链表尾
            tail = p;
            ++modCount;
        }
    }
```

### afterNodeInsertion
```
    /**
     * 移除链表头
     */
    void afterNodeInsertion(boolean evict) {
        LinkedHashMap.Entry<K,V> first;
        // 当evict = true，并且 头节点不为null，removeEldestEntry(first)为true，才进行移除
        if (evict && (first = head) != null && removeEldestEntry(first)) {
            K key = first.key;
            // 用到HashMap的，里面有个方法afterNodeRemoval，下面讲
            removeNode(hash(key), key, null, false, true);
        }
    }
    
    /**
     * 再看看这个方法，其实是返回false的，所以要想实现移除链表头，必须得继承LinkedHashMap然后重写这个方法
     */
    protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
        return false;
    }
    
```

### afterNodeRemoval
```
    /**
     * 移除了e后，保证链表条不断掉
     */
  void afterNodeRemoval(Node<K,V> e) {
        LinkedHashMap.Entry<K,V> p =
            (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
        p.before = p.after = null;
        if (b == null)
            head = a;
        else
            b.after = a;
        if (a == null)
            tail = b;
        else
            a.before = b;
    }
```

### containsValue
```
    /**
     * 是否包含此值
     */
    public boolean containsValue(Object value) {
        // 很明显是遍历链表条了
        for (LinkedHashMap.Entry<K,V> e = head; e != null; e = e.after) {
            V v = e.value;
            if (v == value || (value != null && value.equals(v)))
                return true;
        }
        return false;
    }
```

### get、getOrDefault
```
    /**
     * 根据key获取对应值
     */
    public V get(Object key) {
        Node<K,V> e;
        if ((e = getNode(hash(key), key)) == null)
            return null;
        // 如果true，还会把这个链表放到尾部
        if (accessOrder)
            afterNodeAccess(e);
        return e.value;
    }
    
    /**
     * 还提供一个不存在就返回默认值
     */
    public V getOrDefault(Object key, V defaultValue) {
       Node<K,V> e;
       if ((e = getNode(hash(key), key)) == null)
           return defaultValue;
       if (accessOrder)
           afterNodeAccess(e);
       return e.value;
   }
```

### keySet
```
    /**
     * 获取key集合，有兴趣可以自己看LinkedKeySet
     */
    public Set<K> keySet() {
        Set<K> ks = keySet;
        if (ks == null) {
            ks = new LinkedKeySet();
            keySet = ks;
        }
        return ks;
    }
    
    final class LinkedKeySet extends AbstractSet<K> {
        public final int size()                 { return size; }
        public final void clear()               { LinkedHashMap.this.clear(); }
        public final Iterator<K> iterator() {
            return new LinkedKeyIterator();
        }
        public final boolean contains(Object o) { return containsKey(o); }
        public final boolean remove(Object key) {
            return removeNode(hash(key), key, null, false, true) != null;
        }
        public final Spliterator<K> spliterator()  {
            return Spliterators.spliterator(this, Spliterator.SIZED |
                                            Spliterator.ORDERED |
                                            Spliterator.DISTINCT);
        }
        public final void forEach(Consumer<? super K> action) {
            if (action == null)
                throw new NullPointerException();
            int mc = modCount;
            for (LinkedHashMap.Entry<K,V> e = head; e != null; e = e.after)
                action.accept(e.key);
            if (modCount != mc)
                throw new ConcurrentModificationException();
        }
    }
```

### newNode、newTreeNode
```
    /**
     * 新建链表
     */
    Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
        LinkedHashMap.Entry<K,V> p =
            new LinkedHashMap.Entry<K,V>(hash, key, value, e);
        linkNodeLast(p);
        return p;
    }
    /**
     * 新建红黑树节点
     */ 
    TreeNode<K,V> newTreeNode(int hash, K key, V value, Node<K,V> next) {
        TreeNode<K,V> p = new TreeNode<K,V>(hash, key, value, next);
        linkNodeLast(p);
        return p;
    }
    
    /**
     * 上面都有用到，把p接到旧链表尾后面
     */
    private void linkNodeLast(LinkedHashMap.Entry<K,V> p) {
        LinkedHashMap.Entry<K,V> last = tail;
        tail = p;
        if (last == null)
            head = p;
        else {
            p.before = last;
            last.after = p;
        }
    }
    
```
### values
```
    /**
     * 获取value集合
     */
    public Collection<V> values() {
        Collection<V> vs = values;
        if (vs == null) {
            vs = new LinkedValues();
            values = vs;
        }
        return vs;
    }

    final class LinkedValues extends AbstractCollection<V> {
        public final int size()                 { return size; }
        public final void clear()               { LinkedHashMap.this.clear(); }
        public final Iterator<V> iterator() {
            return new LinkedValueIterator();
        }
        public final boolean contains(Object o) { return containsValue(o); }
        public final Spliterator<V> spliterator() {
            return Spliterators.spliterator(this, Spliterator.SIZED |
                                            Spliterator.ORDERED);
        }
        public final void forEach(Consumer<? super V> action) {
            if (action == null)
                throw new NullPointerException();
            int mc = modCount;
            for (LinkedHashMap.Entry<K,V> e = head; e != null; e = e.after)
                action.accept(e.value);
            if (modCount != mc)
                throw new ConcurrentModificationException();
        }
    }
```
