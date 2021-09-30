---
title: java.util.HashSet类解析
date: 2019-04-16 21:23:50
categories: 
  - java
tags: 
  - little_eight
---

HashSet 是一个由 HashMap 实现的集合，元素无序且不能重复。
其方法基本都是HashMap的方法。
```
 /**
     * 继承 AbstractSet
     *          AbstractSet也是实现了Set接口的，跟HashMap一个毛病，脱裤子放屁....
     * 实现 Set 接口
     *          这个接口是 Set 类集合的上层接口，定义了实现该接口的类都必须要实现的一组方法
     * 实现 Cloneable 接口
     *          这个类是 java.lang.Cloneable，前面我们讲解深拷贝和浅拷贝的原理时，
     *          我们介绍了浅拷贝可以通过调用 Object.clone() 方法来实现，
     *          但是调用该方法的对象必须要实现 Cloneable 接口，
     *          否则会抛出 CloneNoSupportException异常。
     * 实现 Serializable 接口
     *          序列化
     */
    public class HashSet<E> extends AbstractSet<E>
    implements Set<E>, Cloneable, java.io.Serializable{}
```

## 成员变量
```
    /**
     * 就是一个HashMap，说明靠这个来存储数据了
     */
      private transient HashMap<E,Object> map;
    /**
     * 向HashSet中添加数据，数据在上面的 map 结构是作为 key 存在的，而value统一都是 PRESENT
     */
    private static final Object PRESENT = new Object();
```
<!--more-->
## 构造函数
```
    /**
     * 就是new一个HashMap
     */
    public HashSet() {
        map = new HashMap<>();
    }
    /**
     * 指定初始容量HashMap 
     */
    public HashSet(int initialCapacity) {
        map = new HashMap<>(initialCapacity);
    }
     
    /**
     * 指定初始容量和加载因子HashMap 
     */
    public HashSet(int initialCapacity, float loadFactor) {
        map = new HashMap<>(initialCapacity, loadFactor);
    }
    
    /**
     * 这个dummy都没用到,兼容旧版本用的吧。
     */
    HashSet(int initialCapacity, float loadFactor, boolean dummy) {
        map = new LinkedHashMap<>(initialCapacity, loadFactor);
    }
    
    
    /**
     * 构造包含指定集合中的元素
     */
    public HashSet(Collection<? extends E> c) {
        map = new HashMap<>(Math.max((int) (c.size()/.75f) + 1, 16));
        addAll(c);
    }
     
```
## 方法

### add
```
    /**
     * 往map里put值
     */
     public boolean add(E e) {
        return map.put(e, PRESENT)==null;
    }
```

### clear
```
    /**
     * 清除map
     */
    public void clear() {
        map.clear();
    }
```

### clone
```
    /**
     * 克隆
     */
    @SuppressWarnings("unchecked")
    public Object clone() {
        try {
            HashSet<E> newSet = (HashSet<E>) super.clone();
            newSet.map = (HashMap<E, Object>) map.clone();
            return newSet;
        } catch (CloneNotSupportedException e) {
            throw new InternalError(e);
        }
    }
```

### contains
```
    /**
     * 是否包含此值，也就是判断是否包含此key
     */
    public boolean contains(Object o) {
        return map.containsKey(o);
    }
```

### isEmpty
```
    /**
     * 是否为空
     */
    public boolean isEmpty() {
        return map.isEmpty();
    }
```

### iterator
```
    /**
     * 迭代类
     */
    public Iterator<E> iterator() {
        return map.keySet().iterator();
    }
```

### remove
```
    /**
     * 移除某值
     */
    public boolean remove(Object o) {
        return map.remove(o)==PRESENT;
    }
```
