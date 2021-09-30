---
title: java.util.LinkedList类解析
date: 2019-04-15 22:11:43
categories: 
  - java
tags: 
  - little_eight
---

　LinkedList 是一个用链表实现的集合，元素有序且可以重复。

```
    /**
     * 实现 List 接口
     *          这个接口是 List 类集合的上层接口，定义了实现该接口的类都必须要实现的一组方法
     * 实现 Deque 接口
     *          这是一个双向队列接口，双向队列就是两端都可以进行增加和删除操作。
     * 实现 Cloneable 接口
     *          这个类是 java.lang.Cloneable，前面我们讲解深拷贝和浅拷贝的原理时，
     *          我们介绍了浅拷贝可以通过调用 Object.clone() 方法来实现，
     *          但是调用该方法的对象必须要实现 Cloneable 接口，
     *          否则会抛出 CloneNoSupportException异常。
     * 实现 Serializable 接口
     *          序列化
     */
    public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
    {}
```

## 成员变量

```
    /**
     * 链表的个数，也就是集合的大小吧
     */
    transient int size = 0;

    /**
     * 链表的头
     */
    transient Node<E> first;

    /**
     * 链表的尾
     */
    transient Node<E> last;
```

```
    /**
     * 链表的类
     */
   private static class Node<E> {
        // 存的值
        E item;
        // 指向下一个节点的引用
        Node<E> next;
        // 指向上一个节点的引用
        Node<E> prev;

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }

```
<!--more-->
## 构造方法

```
  /**
     * 什么都没初始化。。。
     */
    public LinkedList() {
    }
```


```
    /**
     * 就是集合c转换成LinkedList吧，addAll方法后面再说
     */
    public LinkedList(Collection<? extends E> c) {
        this();
        addAll(c);
    }
    
     public boolean addAll(int index, Collection<? extends E> c) {
        checkPositionIndex(index);

        Object[] a = c.toArray();
        int numNew = a.length;
        if (numNew == 0)
            return false;

        Node<E> pred, succ;
        if (index == size) {
            succ = null;
            pred = last;
        } else {
            succ = node(index);
            pred = succ.prev;
        }

        for (Object o : a) {
            @SuppressWarnings("unchecked") E e = (E) o;
            Node<E> newNode = new Node<>(pred, e, null);
            if (pred == null)
                first = newNode;    
            else
                pred.next = newNode;
            pred = newNode;
        }

        if (succ == null) {
            last = pred;
        } else {
            pred.next = succ;
            succ.prev = pred;
        }

        size += numNew;
        modCount++;
        return true;
    }

```
## 方法

### add
```
 /**
  *把元素加到指定下标，也就是在指定下标插入（新建）链表
  */

public void add(int index, E element) {
        checkPositionIndex(index);
        // 如果下标刚好是集合大小，就直接把这个元素加到链表尾
        if (index == size)
            linkLast(element);
        else
            linkBefore(element, node(index));
    }
      /**
       * 加到链表尾的方法
       */
      void linkLast(E e) {
        // 现在的链表尾赋给 l
        final Node<E> l = last;
        // 新建链表，上一节点为 l，下一节点为null
        final Node<E> newNode = new Node<>(l, e, null);
        // newNode为链表尾
        last = newNode;
        // 如果l为空，说明是没有链表的，把newNode也设置成链表头
        if (l == null)
            first = newNode;
        // 否则的话 就把newNode赋值给l.next，让整条链表完整
        else
            l.next = newNode;
        size++;
        modCount++;
    }
        /**
         * 在某个链表前面插入一个链表
         */
       void linkBefore(E e, Node<E> succ) {
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
### addAll
```
 /**
  * 在下标index后，添加集合c
  */
 public boolean addAll(int index, Collection<? extends E> c) {
        checkPositionIndex(index);

        Object[] a = c.toArray();
        int numNew = a.length;
        if (numNew == 0)
            return false;

        Node<E> pred, succ;
        if (index == size) {
            succ = null;
            pred = last;
        } else {
            succ = node(index);
            pred = succ.prev;
        }

        for (Object o : a) {
            @SuppressWarnings("unchecked") E e = (E) o;
            Node<E> newNode = new Node<>(pred, e, null);
            if (pred == null)
                first = newNode;
            else
                pred.next = newNode;
            pred = newNode;
        }

        if (succ == null) {
            last = pred;
        } else {
            pred.next = succ;
            succ.prev = pred;
        }

        size += numNew;
        modCount++;
        return true;
    }

```

### addFirst
```
 /**
  * 实现的办法是这个私有的，就是在表头加链表
  */
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
```

### addLast
```
 /**
  * 实现的办法是这个私有的，就是在表尾加链表
  */
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
```

### clear
```
 /**
  * 清除所有链表
  */
      public void clear() {
        for (Node<E> x = first; x != null; ) {
            Node<E> next = x.next;
            x.item = null;
            x.next = null;
            x.prev = null;
            x = next;
        }
        first = last = null;
        size = 0;
        modCount++;
    }
```

### clone
```
 /**
  * 浅拷贝
  */
    public Object clone() {
        // 就是Object.clone()实现克隆
        LinkedList<E> clone = superClone();
        clone.first = clone.last = null;
        clone.size = 0;
        clone.modCount = 0;

        for (Node<E> x = first; x != null; x = x.next)
            clone.add(x.item);

        return clone;
    }
```

### contains、indexOf、lastIndexOf
```
 /**
  * 是否包含此元素
  */
    public boolean contains(Object o) {
        return indexOf(o) != -1;
    }
    /**
     * 判断是否存在此元素，存在返回下标，否则返回-1
     */ 
    public int indexOf(Object o) {
        int index = 0;
        if (o == null) {
            for (Node<E> x = first; x != null; x = x.next) {
                if (x.item == null)
                    return index;
                index++;
            }
        } else {
            for (Node<E> x = first; x != null; x = x.next) {
                if (o.equals(x.item))
                    return index;
                index++;
            }
        }
        return -1;
    }
    
     /**
     * 从尾部开始循环，判断是否存在此元素，存在返回下标，否则返回-1
     */ 
     public int lastIndexOf(Object o) {
        int index = size;
        if (o == null) {
            for (Node<E> x = last; x != null; x = x.prev) {
                index--;
                if (x.item == null)
                    return index;
            }
        } else {
            for (Node<E> x = last; x != null; x = x.prev) {
                index--;
                if (o.equals(x.item))
                    return index;
            }
        }
        return -1;
    }
```

### get、getFirst、getLast
这三个get方法都必须保证元素非null的。

```
 /**
  * 根据下标返回元素（）
  */
 public E get(int index) {
        checkElementIndex(index);
        return node(index).item;
    }
     /**
      * 返回链表头的元素
      */
     public E getFirst() {
        final Node<E> f = first;
        if (f == null)
            throw new NoSuchElementException();
        return f.item;
    }
     /**
      * 返回链表尾的元素
      */
        public E getLast() {
        final Node<E> l = last;
        if (l == null)
            throw new NoSuchElementException();
        return l.item;
    }
```

### listIterator
```
 /**
  * 就是根据下标new一个内部迭代类
  */
   public ListIterator<E> listIterator(int index) {
        checkPositionIndex(index);
        return new ListItr(index);
    }
```
### offer、offerFirst、offerLast
这个跟add一样的，只不过返回一个true的结果
```
   public boolean offer(E e) {
        return add(e);
    }
    
    public boolean offerFirst(E e) {
        addFirst(e);
        return true;
    }
    
      public boolean offerLast(E e) {
        addLast(e);
        return true;
    }

```



### peek、peekFirst、peekLast
这三个方法都不用保证元素非null的。
```
   /**
    * 1.5版本的获取链表头的元素
    */
    public E peek() {
        final Node<E> f = first;
        return (f == null) ? null : f.item;
    }
     /**
      *1.6版本的获取链表头的元素
      */
    public E peekFirst() {
        final Node<E> f = first;
        return (f == null) ? null : f.item;
     }
     
      /**
       *获取链表尾的元素
       */
    public E peekLast() {
        final Node<E> l = last;
        return (l == null) ? null : l.item;
    }

    
    
```

### poll、pollFirst、pollLast
```
     /**
      * 1.5版本的删除链表头
      */
   public E poll() {
        final Node<E> f = first;
        return (f == null) ? null : unlinkFirst(f);
    }
     /**
      * 1.6版本删除链表头
      */
    public E pollFirst() {
        final Node<E> f = first;
        return (f == null) ? null : unlinkFirst(f);
    }
     /**
      * 删除链表尾
      */
     public E pollLast() {
        final Node<E> l = last;
        return (l == null) ? null : unlinkLast(l);
    }
    
    /**
     * 把指定的链表设置为null，让gc自己去回收
     */
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
```

### remove、removeFirst、removeLast
顾名思义，就是删除链表，实现方法都是unlink（必须保证非null）
```
 /**
  * 删除指定链表
  */
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

### set
```
 /**
  * 根据下标设置对应链表的元素
  */
    public E set(int index, E element) {
        checkElementIndex(index);
        Node<E> x = node(index);
        E oldVal = x.item;
        x.item = element;
        return oldVal;
    }
```

### toArray
```
 /**
  * 转换成数组
  */
   public Object[] toArray() {
        Object[] result = new Object[size];
        int i = 0;
        for (Node<E> x = first; x != null; x = x.next)
            result[i++] = x.item;
        return result;
    }
```
