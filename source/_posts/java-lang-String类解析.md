---
title: java.lang.String类解析
date: 2019-05-09 23:37:16
categories: 
  - java
tags: 
  - little_eight
---

## 成员变量
```
    /**
     * 底层由数组存数据
     */
    private final char value[];

    /**
     * hash值
     */
    private int hash;
```
## 方法
<!--more-->
### charAt
```
    /**
     * 返回指定下标的字符
     */
   public char charAt(int index) {
        if ((index < 0) || (index >= value.length)) {
            throw new StringIndexOutOfBoundsException(index);
        }
        return value[index];
    }
```

### compareTo、compareToIgnoreCase
```
    /**
     * 比较两个字符串的字典顺序，anotherString在后面的话就返回负数，相等为0，前面就正数
     */
  public int compareTo(String anotherString) {
        int len1 = value.length;
        int len2 = anotherString.value.length;
        int lim = Math.min(len1, len2);
        char v1[] = value;
        char v2[] = anotherString.value;

        int k = 0;
        while (k < lim) {
            char c1 = v1[k];
            char c2 = v2[k];
            if (c1 != c2) {
                return c1 - c2;
            }
            k++;
        }
        return len1 - len2;
    }
    /**
     * 忽略大小写的对比
     */
    public int compareToIgnoreCase(String str) {
        return CASE_INSENSITIVE_ORDER.compare(this, str);
    }
```
### concat
```
    /**
     * 将指定的字符串参数连接到字符串上
     */
    public String concat(String str) {
        int otherLen = str.length();
        if (otherLen == 0) {
            return this;
        }
        int len = value.length;
        char buf[] = Arrays.copyOf(value, len + otherLen);
        str.getChars(buf, len);
        return new String(buf, true);
    }
```
### contains
```
    /**
     * 是否存在此字符
     */
   public boolean contains(CharSequence s) {
        return indexOf(s.toString()) > -1;
    }
```
### contentEquals
```
    /**
     * 比较两者的内容是否相同，不检查被比较对象的类型
     */
  public boolean contentEquals(CharSequence cs) {
        // Argument is a StringBuffer, StringBuilder
        if (cs instanceof AbstractStringBuilder) {
            if (cs instanceof StringBuffer) {
                synchronized(cs) {
                   return nonSyncContentEquals((AbstractStringBuilder)cs);
                }
            } else {
                return nonSyncContentEquals((AbstractStringBuilder)cs);
            }
        }
        // Argument is a String
        if (cs instanceof String) {
            return equals(cs);
        }
        // Argument is a generic CharSequence
        char v1[] = value;
        int n = v1.length;
        if (n != cs.length()) {
            return false;
        }
        for (int i = 0; i < n; i++) {
            if (v1[i] != cs.charAt(i)) {
                return false;
            }
        }
        return true;
    }
    /**
     * 还可以直接传入StringBuffer对比
     */
    public boolean contentEquals(StringBuffer sb) {
        return contentEquals((CharSequence)sb);
    }
```
### copyValueOf
```
    /**
     * 根据char数组new一个新的string
     */
    public static String copyValueOf(char data[]) {
        return new String(data);
    }
     /**
     * 更多参数new一个新的string
     */
    public static String copyValueOf(char data[], int offset, int count) {
        return new String(data, offset, count);
    }
```

### endsWith、startsWith
```
    /**
     * 比较是否与尾部的字符串一致
     */
    public boolean endsWith(String suffix) {
        return startsWith(suffix, value.length - suffix.value.length);
    }
    
    /**
     * 对比指定范围的字符串是否一致
     */
    public boolean startsWith(String prefix, int toffset) {
        char ta[] = value;
        int to = toffset;
        char pa[] = prefix.value;
        int po = 0;
        int pc = prefix.value.length;
        // Note: toffset might be near -1>>>1.
        if ((toffset < 0) || (toffset > value.length - pc)) {
            return false;
        }
        while (--pc >= 0) {
            if (ta[to++] != pa[po++]) {
                return false;
            }
        }
        return true;
    }
        
    /**
     * 比较是否与头部的字符串一致
     */
     public boolean startsWith(String prefix) {
        return startsWith(prefix, 0);
    }
```

### equals
```
    /**
     * 比较两者的内容是否相同，还会检查被比较对象的类型
     */
  public boolean equals(Object anObject) {
        if (this == anObject) {
            return true;
        }
        if (anObject instanceof String) {
            String anotherString = (String)anObject;
            int n = value.length;
            if (n == anotherString.value.length) {
                char v1[] = value;
                char v2[] = anotherString.value;
                int i = 0;
                while (n-- != 0) {
                    if (v1[i] != v2[i])
                        return false;
                    i++;
                }
                return true;
            }
        }
        return false;
    }
```
### format
```
    /**
     * 字符串格式化
     * %s 字符串类型、 %c 字符类型、 %b 布尔类型、%d 整数类型（十进制）、%f 浮点类型.....
     */
    public static String format(String format, Object... args) {
        return new Formatter().format(format, args).toString();
    }
```
### getBytes
```
    /**
     * char数组转换成byte 数组
     */
   public byte[] getBytes() {
        return StringCoding.encode(value, 0, value.length);
    }
```

### indexOf、lastIndexOf    
```
    /**
     * 其实有很多重载方法，这是最常用的，返回第一次出现该字符串的下标，不存在返回-1
     */
   public int indexOf(String str) {
        return indexOf(str, 0);
    }
    /**
     * 返回最后一次出现该字符串的下标，不存在返回-1
     */
    public int lastIndexOf(String str) {
        return lastIndexOf(str, value.length);
    }
```

### length
```
    /**
     * 经常会提问的数组跟String是属性还是方法
     */
    public int length() {
        return value.length;
    }
```

### matches
```
    /**
     * 检测字符串是否匹配给定的正则表达式
     */
        public boolean matches(String regex) {
        return Pattern.matches(regex, this);
    }
```

### replaceAll
```
    /**
     * 替换指定字符串的内容
     */
    public String replaceAll(String regex, String replacement) {
        return Pattern.compile(regex).matcher(this).replaceAll(replacement);
    }
```

### split
```
    /**
     * 转换成数组，regex为区分标识
     */
    public String[] split(String regex) {
        return split(regex, 0);
    }
```


### substring
```
    /**
     * 从指定下标开始截取字符串
     * 感觉直接返回substring(beginIndex, value.length - beginIndex)可以吧
     */
        public String substring(int beginIndex) {
        if (beginIndex < 0) {
            throw new StringIndexOutOfBoundsException(beginIndex);
        }
        int subLen = value.length - beginIndex;
        if (subLen < 0) {
            throw new StringIndexOutOfBoundsException(subLen);
        }
        return (beginIndex == 0) ? this : new String(value, beginIndex, subLen);
    }
    
    /**
     * 从指定下标开始截取字符串
     */
     public String substring(int beginIndex, int endIndex) {
        if (beginIndex < 0) {
            throw new StringIndexOutOfBoundsException(beginIndex);
        }
        if (endIndex > value.length) {
            throw new StringIndexOutOfBoundsException(endIndex);
        }
        int subLen = endIndex - beginIndex;
        if (subLen < 0) {
            throw new StringIndexOutOfBoundsException(subLen);
        }
        return ((beginIndex == 0) && (endIndex == value.length)) ? this
                : new String(value, beginIndex, subLen);
    }
```

### toCharArray
```
    /**
     * 转换成字符数组
     */
    public char[] toCharArray() {
        // Cannot use Arrays.copyOf because of class initialization order issues
        char result[] = new char[value.length];
        System.arraycopy(value, 0, result, 0, value.length);
        return result;
    }

```

### toLowerCase、toUpperCase
```
    /**
     * 全部转换小写
     */
     public String toLowerCase() {
        return toLowerCase(Locale.getDefault());
    }
     /**
     * 全部转换大写
     */
    public String toUpperCase() {
        return toUpperCase(Locale.getDefault());
    }
```

### valueOf
```
    /**
     * 转换成z
     */
    public static String valueOf(boolean b) {
        return b ? "true" : "false";
    }
    public static String valueOf(char data[]) {
        return new String(data);
    }
    public static String valueOf(char c) {
        char data[] = {c};
        return new String(data, true);
    }
    public static String valueOf(char data[], int offset, int count) {
        return new String(data, offset, count);
    }
    public static String valueOf(double d) {
        return Double.toString(d);
    }
    
   public static String valueOf(float f) {
        return Float.toString(f);
    }
    public static String valueOf(int i) {
        return Integer.toString(i);
    }
    
    public static String valueOf(long l) {
        return Long.toString(l);
    }
    public static String valueOf(Object obj) {
        return (obj == null) ? "null" : obj.toString();
    }
```
