---
title: java.util.Optional类解析
date: 2019-04-22 21:28:01
categories: 
  - java
tags: 
  - little_eight
---

## 成员变量

```
    /**
     * 从后面的构造方法可以看出，这里new一个value为null的Optional
     */
    private static final Optional<?> EMPTY = new Optional<>();

    /**
     * 值
     */
    private final T value;
```


## 构造方法 
```
    /**
     * value = null
     */
    private Optional() {
        this.value = null;
    }
    
    /**
     * 自定义value值，value为null会抛异常
     */
    private Optional(T value) {
        this.value = Objects.requireNonNull(value);
    }
    
    public static <T> T requireNonNull(T obj) {
        if (obj == null)
            throw new NullPointerException();
        return obj;
    }
```
<!--more-->
## empty
```
    /**
     * 返回一个空Optional
     */
     public static<T> Optional<T> empty() {
        @SuppressWarnings("unchecked")
        Optional<T> t = (Optional<T>) EMPTY;
        return t;
    }
```

## equals
```
    /**
     * 重写equals，为了实现最后2行代码
     */
    public boolean equals(Object obj) {
        if (this == obj) {
            return true;
        }

        if (!(obj instanceof Optional)) {
            return false;
        }

        Optional<?> other = (Optional<?>) obj;
        return Objects.equals(value, other.value);
    }
```

## filter

```
    /**
     * 如果value不为空并且满足断言条件返回包含该值的Optional，否则返回空Optional。
     * 用法：filter((value) -> xx())
     */
    public Optional<T> filter(Predicate<? super T> predicate) {
        Objects.requireNonNull(predicate);
        if (!isPresent())
            return this;
        else
            return predicate.test(value) ? this : empty();
    }
```

## get
```
    /**
     * 获取value，为null抛异常
     */
    public T get() {
        if (value == null) {
            throw new NoSuchElementException("No value present");
        }
        return value;
    }
```

## isPresent
```
    /**
     * value是否为null
     */
    public boolean isPresent() {
        return value != null;
    }
    /**
     * 跟上面一样，不过是迎合lambda，写法ifPresent((value) -> xx()）
     */
    public void ifPresent(Consumer<? super T> consumer) {
        if (value != null)
            consumer.accept(value);
    }
```

## map
```
    /**
     * 如果value不为null，则对其执行调用mapping函数得到返回值。
     * 如果返回值不为null，则创建包含mapping返回值的Optional作为map方法返回值，否则返回空Optional。
     */
    public<U> Optional<U> map(Function<? super T, ? extends U> mapper) {
        Objects.requireNonNull(mapper);
        if (!isPresent())
            return empty();
        else {
            return Optional.ofNullable(mapper.apply(value));
        }
    }
```

## of
``` 
    /**
     *  返回一个带value的Optional
     */
    public static <T> Optional<T> of(T value) {
        return new Optional<>(value);
    }
```

## ofNullable
```
    /**
     *  相当于综合了empty跟of
     */
     public static <T> Optional<T> ofNullable(T value) {
        return value == null ? empty() : of(value);
    }
```

## orElse
```
    /**
     * value为null就返回other，不为空返回value
     */
    public T orElse(T other) {
        return value != null ? value : other;
    }
```

## orElseGet
```
    /**
     * value为null就返回other，不为空返回value
     * 用法 orElseGet(() -> xx())
     */
    public T orElseGet(Supplier<? extends T> other) {
        return value != null ? value : other.get();
    }
```
## orElseThrow
```
    /**
     * 如果value不为空返回value，否则抛出supplier接口创建的异常。
     * 用法： orElseThrow(XXException::new)
     */
    public <X extends Throwable> T orElseThrow(Supplier<? extends X> exceptionSupplier) throws X {
        if (value != null) {
            return value;
        } else {
            throw exceptionSupplier.get();
        }
    }
```
