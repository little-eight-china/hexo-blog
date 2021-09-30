---
title: java.util.ArrayList类解析
date: 2019-04-11 22:19:30
categories: 
  - java
tags: 
  - little_eight
---
ArrayList 是一个用数组实现的集合，支持随机访问，元素有序且可以重复。

```
    /**
     * 实现 List 接口
     *          这个接口是 List 类集合的上层接口，定义了实现该接口的类都必须要实现的一组方法
     * 实现 RandomAccess 接口
     *          这是一个标记接口，一般此标记接口用于 List 实现，
     *          以表明它们支持快速（通常是恒定时间）的随机访问。
     *          该接口的主要目的是允许通用算法改变其行为，
     *          以便在应用于随机或顺序访问列表时提供良好的性能
     * 实现 Cloneable 接口
     *          这个类是 java.lang.Cloneable，前面我们讲解深拷贝和浅拷贝的原理时，
     *          我们介绍了浅拷贝可以通过调用 Object.clone() 方法来实现，
     *          但是调用该方法的对象必须要实现 Cloneable 接口，
     *          否则会抛出 CloneNoSupportException异常。
     * 实现 Serializable 接口
     *          序列化
     */
    public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
    {}
```

## 成员变量

```
    /**
     * 默认初始大小.
     */
    private static final int DEFAULT_CAPACITY = 10;

    /**
     * 用于空实例的共享空数组实例。
     */
    private static final Object[] EMPTY_ELEMENTDATA = {};

    /**
     * 共享空数组实例，用于默认大小的空实例。我们将其与空元素数据区分开来，
     * 以了解添加第一个元素时要加多少量。
     */
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

    /**
     * 存储集合元素的数组缓冲区。集合的容量是这个数组缓冲区的长度。
     * 任何带有elementdata==DEFAULTCAPACITY_EMPTY_ELEMENTDATA的空集合
     * 将在添加第一个元素时扩展为默认容量。
     */
    transient Object[] elementData;

    /**
     * 集合大小
     */
    private int size;

```
<!--more-->
## 构造函数

```
    /**
     * Constructs an empty list with an initial capacity of ten.(构造初始容量为10的空列表。)
     * 明明说是10，其实看代码只是创建了一个空数组，应该是版本更新忘了改注释了。
     */
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }

```

```

    /**
     * 构造具有指定初始容量的空列表
     */
    public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }
```
```
    /**
     * 按照集合迭代器返回元素的顺序构造包含指定集合的元素的列表。
     */
    public ArrayList(Collection<? extends E> c) {
        elementData = c.toArray();
        if ((size = elementData.length) != 0) {
            // c.toArray might (incorrectly) not return Object[] (see 6260652)
            if (elementData.getClass() != Object[].class)
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            // replace with empty array.
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }
```

## 方法

## add

```
   
   public boolean add(E e) {
        ensureCapacityInternal(size + 1);
        elementData[size++] = e;
        return true;
    }
```
 看看ensureCapacityInternal方法
```
     private void ensureCapacityInternal(int minCapacity) {
        ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
    }
```
再看看calculateCapacity,可以看到当elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA时，也就是这是new ArrayList后第一次加进元素的话，就会返回初始容量10，后面根据这个新建一个10容量的数组
```
     private static int calculateCapacity(Object[] elementData, int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            return Math.max(DEFAULT_CAPACITY, minCapacity);
        }
        return minCapacity;
    }
```
ensureExplicitCapacity是干嘛的呢，是确保数组可用，如果容量不够，就要进行扩容了，也就是grow方法

```
  private void ensureExplicitCapacity(int minCapacity) {
        modCount++;
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }
```
看看grow方法
```
    /**
     * 增加容量以确保它至少能容纳最小容量参数指定的元素数。一般是扩容1.5倍
     */
   private void grow(int minCapacity) {
        // 现在数组的容量
        int oldCapacity = elementData.length;
        // 扩容1.5倍的容量
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        // 如果1.5倍容量小于minCapacity，那把minC赋值给newC
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        // 如果1.5倍容量大于MAX_ARRAY_SIZE（int最大值 - 8）
        if (newCapacity - MAX_ARRAY_SIZE（0x7fffffff - 8） > 0)
            // hugeCapacity是这样的，minC > MAX_ARRAY_SIZE) ?Integer.MAX_VALUE : MAX_ARRAY_SIZE;
            newCapacity = hugeCapacity(minCapacity);
        // 这里用到数组复制的方法，扩容到指定大小
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
```


### clear

```
    // 就是清空数组，也就是清空集合
   public void clear() {
        modCount++;

        // clear to let GC do its work
        for (int i = 0; i < size; i++)
            elementData[i] = null;

        size = 0;
    }
```

### clone

```
  // 克隆集合。。
  public Object clone() {
        try {
            ArrayList<?> v = (ArrayList<?>) super.clone();
            v.elementData = Arrays.copyOf(elementData, size);
            v.modCount = 0;
            return v;
        } catch (CloneNotSupportedException e) {
            // this shouldn't happen, since we are Cloneable
            throw new InternalError(e);
        }
   
 }
```

### contains

```
    public boolean contains(Object o) {
        return indexOf(o) >= 0;
    }
    //  // 从前面开始遍历，返回第一个与o相等的下标，不存在返回-1
    public int indexOf(Object o) {
        if (o == null) {
            for (int i = 0; i < size; i++)
                if (elementData[i]==null)
                    return i;
        } else {
            for (int i = 0; i < size; i++)
                if (o.equals(elementData[i]))
                    return i;
        }
        return -1;
    }

```

### get

```
 // 根据下标值获取对应元素
 public E get(int index) {
        // 检查是否超过最大下标值
        rangeCheck(index);
        // 就是 elementData[index]
        return elementData(index);
    }

```

### iterator

```
  // 返回的是一个内部类 Itr，Itr这里就不做多解释了，有兴趣可以自己去看。
  public Iterator<E> iterator() {
        return new Itr();
    }
```
### lastIndexOf
```
 // 从后面开始遍历，返回第一个与o相等的下标，不存在返回-1
  public int lastIndexOf(Object o) {
        if (o == null) {
            for (int i = size-1; i >= 0; i--)
                if (elementData[i]==null)
                    return i;
        } else {
            for (int i = size-1; i >= 0; i--)
                if (o.equals(elementData[i]))
                    return i;
        }
        return -1;
    }
```


### remove
```
   // 移除指定下标的元素，可以看到是用System.arraycopy来实现对移除元素后数组进行重新组合，让其他元素位置不变
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
还有一个根据元素值来移除的，就是遍历数组，找到下标，移除，也是用System.arraycopy来实现

### removeAll
```
   // 从此列表中删除指定集合中包含的所有元素。
    public boolean removeAll(Collection<?> c) {
        Objects.requireNonNull(c);
        return batchRemove(c, false);
    }
```

```
 // 核心还是用到了System.arraycopy
 private boolean batchRemove(Collection<?> c, boolean complement) {
        final Object[] elementData = this.elementData;
        int r = 0, w = 0;
        boolean modified = false;
        try {
            for (; r < size; r++)
                // complement等于false，也就是c不包含的元素，重新从下标0赋值到elementData
                if (c.contains(elementData[r]) == complement)
                    elementData[w++] = elementData[r];
        } finally {
            if (r != size) {
                System.arraycopy(elementData, r,
                                 elementData, w,
                                 size - r);
                w += size - r;
            }
            if (w != size) {
                // clear to let GC do its work
                for (int i = w; i < size; i++)
                    elementData[i] = null;
                modCount += size - w;
                size = w;
                modified = true;
            }
        }
        return modified;
    }
```
### set
```
 // 设值。。
  public E set(int index, E element) {
        rangeCheck(index);

        E oldValue = elementData(index);
        elementData[index] = element;
        return oldValue;
    }
```

### sort
```
   // 对elementData进行排序，用Arrays.sort实现
   public void sort(Comparator<? super E> c) {
        final int expectedModCount = modCount;
        Arrays.sort((E[]) elementData, 0, size, c);
        if (modCount != expectedModCount) {
            throw new ConcurrentModificationException();
        }
        modCount++;
    }
```

### subList
```
 // 截取指定访问的元素到SubList.,也就是生成新的集合吧
 public List<E> subList(int fromIndex, int toIndex) {
        subListRangeCheck(fromIndex, toIndex, size);
        return new SubList(this, 0, fromIndex, toIndex);
    }
```

### toArray
```
  // 转换成size大小的数组。。。
  public Object[] toArray() {
        return Arrays.copyOf(elementData, size);
    }
```

```
 // 还可以指定转换的类型
 public <T> T[] toArray(T[] a) {
        if (a.length < size)
            // Make a new array of a's runtime type, but my contents:
            return (T[]) Arrays.copyOf(elementData, size, a.getClass());
        System.arraycopy(elementData, 0, a, 0, size);
        if (a.length > size)
            a[size] = null;
        return a;
    }
```

### trimToSize
```
  // 把数组转换成size大小的数组
   public void trimToSize() {
        modCount++;
        if (size < elementData.length) {
            elementData = (size == 0)
              ? EMPTY_ELEMENTDATA
              : Arrays.copyOf(elementData, size);
        }
    }

```
