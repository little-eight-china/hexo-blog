---
title: java.util.LinkedHashSet类解析
date: 2019-04-22 21:27:48
categories: 
  - java
tags: 
  - little_eight
---

元素有序且不能重复。
```
/**
    * 实现 Cloneable 接口
    *          这个类是 java.lang.Cloneable，前面我们讲解深拷贝和浅拷贝的原理时，
    *          我们介绍了浅拷贝可以通过调用 Object.clone() 方法来实现，
    *          但是调用该方法的对象必须要实现 Cloneable 接口，
    *          否则会抛出 CloneNoSupportException异常。
    * 实现 Serializable 接口
    *          序列化
    */
    public class LinkedHashSet<E> extends HashSet<E> implements Set<E>,
    Cloneable,java.io.Serializable{}
```
看上面似乎LinkedHashSet 是由HashSet实现的集合，其实是LinkedHashMap实现的。从下面的构造方法即可看出。


## 构造方法
```
    /**
     * 依然是提供三种构造方法，实现都是super方法，下面我们看看是怎么实现的 
     */
   public LinkedHashSet() {
        super(16, .75f, true);
    }
    
   public LinkedHashSet(Collection<? extends E> c) {
        super(Math.max(2*c.size(), 11), .75f, true);
        addAll(c);
    }
    
    public LinkedHashSet(int initialCapacity) {
        super(initialCapacity, .75f, true);
    }
    
    
    /**
     * 之前还疑惑这里dummy为啥没用，原来是为了用到这里做区分的
     * 明显看出看出LinkedHashSet 是由LinkedHashMap实现的集合
     */
    HashSet(int initialCapacity, float loadFactor, boolean dummy) {
        map = new LinkedHashMap<>(initialCapacity, loadFactor);
    }    

```

## 其他方法基本都可以看HashSet了
