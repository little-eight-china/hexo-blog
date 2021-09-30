---
title: java.util.Arrays类解析
date: 2019-04-11 22:18:45
categories: 
  - java
tags: 
  - little_eight
---
源码的类注释：This class contains various methods for manipulating arrays (such as sorting and searching). This class also contains a static factory that allows arrays to be viewed as lists.
可见这就是一个处理数组的类，直接研究含有的方法。

## asList

```
    /**
     * 就是数组转换成List集合
     */
   public static <T> List<T> asList(T... a) {
        return new ArrayList<>(a);
    }
```
<!--more-->
## binarySearch

源码方法注释：Searches the specified array of longs for the specified value using the binary search algorithm. The array must be sorted
就是 采用二进制搜索算法，而且其数组必须是要经过排序的（后面的binarySearch0的使用基础必须是已经排序好的数组），排序刚好可以使用它的sort方法。
```
    /**
     * 在数组中寻找此元素，返回它的下标
     * 参数可换成其他基本数据类型和Object
     */
    public static int binarySearch(long[] a, long key) {
        return binarySearch0(a, 0, a.length, key);
    }
    
     /**
      * 遍历的二进制搜索算法
      * @param a 数组
      * @param fromIndex 开始下标
      * @param toIndex 结束下标
      * @param key 寻找的元素
      */
     private static int binarySearch0(long[] a, int fromIndex, int toIndex,
                                     long key) {
        int low = fromIndex;
        int high = toIndex - 1;

        while (low <= high) {
            // 取中间值
            int mid = (low + high) >>> 1;
            long midVal = a[mid];

            if (midVal < key)
                low = mid + 1;
            else if (midVal > key)
                high = mid - 1;
            else
                return mid; // key found
        }
        return -(low + 1);  // key not found.
    }

```

## copyOf

```

  /**
   * 顾名思义，就是复制指定的数组
   * @param newLength 指定新数组的长度，如果小于复制的数组，则截取只保留到此长度的元素
   * @param newType 默认是复制数组的类型(注意是T[]不是T)，也可以定义成新的类型
   */
  public static <T,U> T[] copyOf(U[] original, int newLength, Class<? extends T[]> newType) {
        @SuppressWarnings("unchecked")
        T[] copy = ((Object)newType == (Object)Object[].class)
            ? (T[]) new Object[newLength]
            : (T[]) Array.newInstance(newType.getComponentType(), newLength);
        System.arraycopy(original, 0, copy, 0,
                         Math.min(original.length, newLength));
        return copy;
    }

```

```
  /**
   * 还提供另一个可指定范围复制数组的方法
   */
   public static <T,U> T[] copyOfRange(U[] original, int from, int to, Class<? extends T[]> newType) {
        int newLength = to - from;
        if (newLength < 0)
            throw new IllegalArgumentException(from + " > " + to);
        @SuppressWarnings("unchecked")
        T[] copy = ((Object)newType == (Object)Object[].class)
            ? (T[]) new Object[newLength]
            : (T[]) Array.newInstance(newType.getComponentType(), newLength);
        System.arraycopy(original, from, copy, 0,
                         Math.min(original.length - from, newLength));
        return copy;
    }

```

可以看到最终实现的都是System.arraycopy() 

## deepEquals

```
 /**
  * 就是对比两个数组的值（包括类型）是否完全一致
  */
 public static boolean deepEquals(Object[] a1, Object[] a2) {
        if (a1 == a2)
            return true;
        if (a1 == null || a2==null)
            return false;
        int length = a1.length;
        if (a2.length != length)
            return false;

        for (int i = 0; i < length; i++) {
            Object e1 = a1[i];
            Object e2 = a2[i];

            if (e1 == e2)
                continue;
            if (e1 == null)
                return false;

            // Figure out whether the two elements are equal
            boolean eq = deepEquals0(e1, e2);

            if (!eq)
                return false;
        }
        return true;
    }
```


### 

源码方法注释：Returns a hash code based on the "deep contents" of the specified array. 

```
/**
 * 从 result = 1起，然后从第一个元素进行 result = 31 * result + element.HashCode(); 进行循环计算。
 * elementHash ： 就是下一级元素计算出来的。
 * 当element为 引用数据类型数组时，elementHash使用	Arrays.deepHashCode(Object a[])计算。
 * 当element为 基本数据类型数组时，elementHash使用	Arrays.hashCode(Object a[])计算。
 * 当element为 非数组时，elementHash使用	element.hashCode()计算。
 * 很少用过，不理它了。
 */
public static int deepHashCode(Object a[]) {
        if (a == null)
            return 0;

        int result = 1;

        for (Object element : a) {
            int elementHash = 0;
            if (element instanceof Object[])
                elementHash = deepHashCode((Object[]) element);
            else if (element instanceof byte[])
                elementHash = hashCode((byte[]) element);
            else if (element instanceof short[])
                elementHash = hashCode((short[]) element);
            else if (element instanceof int[])
                elementHash = hashCode((int[]) element);
            else if (element instanceof long[])
                elementHash = hashCode((long[]) element);
            else if (element instanceof char[])
                elementHash = hashCode((char[]) element);
            else if (element instanceof float[])
                elementHash = hashCode((float[]) element);
            else if (element instanceof double[])
                elementHash = hashCode((double[]) element);
            else if (element instanceof boolean[])
                elementHash = hashCode((boolean[]) element);
            else if (element != null)
                elementHash = element.hashCode();

            result = 31 * result + elementHash;
        }

        return result;
    }

```

## deepToString
就是把数组转换成“[xx,yy,zz...]”，这样的字符串，可转换多层次嵌套的数组。而toString方法就只能转换一层数组。


## equals

```
 /**
  * 相比于deepEquals()，这个只要求元素的值一样就返回true了
  */
 public static boolean equals(short[] a, short a2[]) {
        if (a==a2)
            return true;
        if (a==null || a2==null)
            return false;

        int length = a.length;
        if (a2.length != length)
            return false;

        for (int i=0; i<length; i++)
            if (a[i] != a2[i])
                return false;

        return true;
    }
```

## fill

```
  /**
   * 将val值赋给每个元素..
   */
  public static void fill(long[] a, long val) {
        for (int i = 0, len = a.length; i < len; i++)
            a[i] = val;
    }
```

```
   /**
    * 你还可以指定赋值的范围
    */
    public static void fill(long[] a, int fromIndex, int toIndex, long val) {
        rangeCheck(a.length, fromIndex, toIndex);
        for (int i = fromIndex; i < toIndex; i++)
            a[i] = val;
    }

```

## hashCode

```
    /**
     * 从 result = 1起，然后从第一个元素进行 result = 31 * result +element.HashCode();进行循环计算。
     */
    public static int hashCode(byte a[]) {
        if (a == null)
            return 0;

        int result = 1;
        for (byte element : a)
            result = 31 * result + element;

        return result;
    }
```

## legacyMergeSort

源码方法注释有这玩意：/** To be removed in a future release. */，那就不看他了

## parallelPrefix、parallelSetAll
太少用了，忽略不看。

## parallelSort

```
 /**
  * 就是对数组排序，还提供对指定范围内的方法
  */
 public static void parallelSort(long[] a) {
        int n = a.length, p, g;
        if (n <= MIN_ARRAY_SORT_GRAN ||
            (p = ForkJoinPool.getCommonPoolParallelism()) == 1)
            DualPivotQuicksort.sort(a, 0, n - 1, null, 0, 0);
        else
            new ArraysParallelSortHelpers.FJLong.Sorter
                (null, a, new long[n], 0, n, 0,
                 ((g = n / (p << 2)) <= MIN_ARRAY_SORT_GRAN) ?
                 MIN_ARRAY_SORT_GRAN : g).invoke();
    }

```

## setAll
不常用，忽略

## sort
该方法是用于数组排序，在 Arrays 类中有该方法的一系列重载方法，能对7种基本数据类型，包括 byte,char,double,float,int,long,short 等都能进行排序，还有 Object 类型（实现了Comparable接口），以及比较器 Comparator 。这里我们以 int[ ] 为例看看。
```
    /**
     * 就是对数组排序，还提供对指定范围内的方法
     */
     public static void sort(int[] a) {
        DualPivotQuicksort.sort(a, 0, a.length - 1, null, 0, 0);
    }
```

在 Arrays.sort 方法内部调用 DualPivotQuicksort.sort 方法，这个方法的源码很长，分别对于数组的长度进行了各种算法的划分，包括快速排序，插入排序，冒泡排序都有使用。特意翻译了DualPivotQuicksort这个类，有兴趣可以往下看(会逐渐翻译完毕)。

```
 final class DualPivotQuicksort {
        private DualPivotQuicksort() {
        }

        /**
         * 合并排序中的最大运行次数
         */
        private static final int MAX_RUN_COUNT = 67;

        /**
         * 合并排序中运行的最大长度。
         */
        private static final int MAX_RUN_LENGTH = 33;

        /**
         * 如果要排序的数组的长度小于此值常量，快速排序优先用于合并排序。
         */
        private static final int QUICKSORT_THRESHOLD = 286;

        /**
         * 如果要排序的数组的长度小于此常量，则插入排序优先于快速排序。
         */
        private static final int INSERTION_SORT_THRESHOLD = 47;

        /**
         * 如果要排序的字节数组的长度大于此常量，则使用计数排序优先于插入排序。
         */
        private static final int COUNTING_SORT_THRESHOLD_FOR_BYTE = 29;

        /**
         * 如果要排序的短数组或字符数组的长度大于此常量，则计数排序优先于快速排序。
         */
        private static final int COUNTING_SORT_THRESHOLD_FOR_SHORT_OR_CHAR = 3200;

        /**
         * 如果可以合并，请使用给定的工作区数组切片对数组的指定范围进行排序
         *
         * @param a        要排序的数组
         * @param left     排序开始位置
         * @param right    排序结束位置
         * @param work     工作区数组（切片）
         * @param workBase 工作阵列中可用空间的原点
         * @param workLen  工作阵列的可用大小
         */
         static void sort(int[] a, int left, int right,
                         int[] work, int workBase, int workLen) {
            // 在小数组上使用快速排序
            if (right - left < QUICKSORT_THRESHOLD) {
                sort(a, left, right, true);
                return;
            }

        /*
         * 索引运行[i]是第i次运行的开始（升序或降序）。
         */
            int[] run = new int[MAX_RUN_COUNT + 1];
            int count = 0;
            run[0] = left;

            // 检查数组是否接近排序
            for (int k = left; k < right; run[count] = k) {
                if (a[k] < a[k + 1]) { //
                    while (++k <= right && a[k - 1] <= a[k]) ;
                } else if (a[k] > a[k + 1]) { // descending
                    while (++k <= right && a[k - 1] >= a[k]) ;
                    for (int lo = run[count] - 1, hi = k; ++lo < --hi; ) {
                        int t = a[lo];
                        a[lo] = a[hi];
                        a[hi] = t;
                    }
                } else {
                    for (int m = MAX_RUN_LENGTH; ++k <= right && a[k - 1] == a[k]; ) {
                        if (--m == 0) {
                            sort(a, left, right, true);
                            return;
                        }
                    }
                }

            /*
             * 数组不是高度结构化的，请使用快速排序而不是合并排序。
             */
                if (++count == MAX_RUN_COUNT) {
                    sort(a, left, right, true);
                    return;
                }
            }

            // 检查特殊情况
            // 实施说明：变量“right”增加1。
            if (run[count] == right++) { // 上次运行包含一个元素
                run[++count] = right;
            } else if (count == 1) { // 数组已排序
                return;
            }

            // 确定合并的替换基
            byte odd = 0;
            for (int n = 1; (n <<= 1) < count; odd ^= 1) ;

            // 使用或创建用于合并的临时数组B
            int[] b;                 // 临时数组；替换为 a
            int ao, bo;              // 从“左”开始的数组偏移量
            int blen = right - left; // B所需空间
            if (work == null || workLen < blen || workBase + blen > work.length) {
                work = new int[blen];
                workBase = 0;
            }
            if (odd == 0) {
                System.arraycopy(a, left, work, workBase, blen);
                b = a;
                bo = 0;
                a = work;
                ao = workBase - left;
            } else {
                b = work;
                ao = 0;
                bo = workBase - left;
            }

            // 合并
            for (int last; count > 1; count = last) {
                for (int k = (last = 0) + 2; k <= count; k += 2) {
                    int hi = run[k], mi = run[k - 1];
                    for (int i = run[k - 2], p = i, q = mi; i < hi; ++i) {
                        if (q >= hi || p < mi && a[p + ao] <= a[q + ao]) {
                            b[i + bo] = a[p++ + ao];
                        } else {
                            b[i + bo] = a[q++ + ao];
                        }
                    }
                    run[++last] = hi;
                }
                if ((count & 1) != 0) {
                    for (int i = right, lo = run[count - 1]; --i >= lo;
                         b[i + bo] = a[i + ao]
                            )
                        ;
                    run[++last] = right;
                }
                int[] t = a;
                a = b;
                b = t;
                int o = ao;
                ao = bo;
                bo = o;
            }
        }

        /**
         * 快速排序
         *
         * @param leftmost 指示此部分是否在范围内最左边
         */
        private static void sort(int[] a, int left, int right, boolean leftmost) {
            int length = right - left + 1;

            // 在小数组上使用插入排序
            if (length < INSERTION_SORT_THRESHOLD) {
                if (leftmost) {
                /*
                 * 传统的（不带sentinel）插入排序，针对服务器虚拟机进行了优化，用于最左边的部分。
                 */
                    for (int i = left, j = i; i < right; j = ++i) {
                        int ai = a[i + 1];
                        while (ai < a[j]) {
                            a[j + 1] = a[j];
                            if (j-- == left) {
                                break;
                            }
                        }
                        a[j + 1] = ai;
                    }
                } else {
                /*
                 * 跳过最长的升序。
                 */
                    do {
                        if (left >= right) {
                            return;
                        }
                    } while (a[++left] >= a[left - 1]);

                /*
                 * 相邻部分的每一个元素都扮演着哨兵的角色，
                 * 因此这允许我们避免在每次迭代中进行左范围检查。
                 * 此外，我们使用了更优化的算法，即对插入排序，
                 * 这比传统的插入排序实现更快（在快速排序的上下文中）。
                 */
                    for (int k = left; ++left <= right; k = ++left) {
                        int a1 = a[k], a2 = a[left];

                        if (a1 < a2) {
                            a2 = a1;
                            a1 = a[left];
                        }
                        while (a1 < a[--k]) {
                            a[k + 2] = a[k];
                        }
                        a[++k + 1] = a1;

                        while (a2 < a[--k]) {
                            a[k + 1] = a[k];
                        }
                        a[k + 1] = a2;
                    }
                    int last = a[right];

                    while (last < a[--right]) {
                        a[right + 1] = a[right];
                    }
                    a[right + 1] = last;
                }
                return;
            }

            // 长度的廉价近似值/7
            int seventh = (length >> 3) + (length >> 6) + 1;

        /*
         * 在范围内的中心元素周围（包括中心元素）对五个等距元素进行排序。
         * 这些元素将用于如下所述的轴选择。
         * 选择这些元素的间距是根据经验确定的，可以很好地处理各种输入。
         */
            int e3 = (left + right) >>> 1; // The midpoint
            int e2 = e3 - seventh;
            int e1 = e2 - seventh;
            int e4 = e3 + seventh;
            int e5 = e4 + seventh;

            // 使用插入排序对这些元素排序
            if (a[e2] < a[e1]) {
                int t = a[e2];
                a[e2] = a[e1];
                a[e1] = t;
            }

            if (a[e3] < a[e2]) {
                int t = a[e3];
                a[e3] = a[e2];
                a[e2] = t;
                if (t < a[e1]) {
                    a[e2] = a[e1];
                    a[e1] = t;
                }
            }
            if (a[e4] < a[e3]) {
                int t = a[e4];
                a[e4] = a[e3];
                a[e3] = t;
                if (t < a[e2]) {
                    a[e3] = a[e2];
                    a[e2] = t;
                    if (t < a[e1]) {
                        a[e2] = a[e1];
                        a[e1] = t;
                    }
                }
            }
            if (a[e5] < a[e4]) {
                int t = a[e5];
                a[e5] = a[e4];
                a[e4] = t;
                if (t < a[e3]) {
                    a[e4] = a[e3];
                    a[e3] = t;
                    if (t < a[e2]) {
                        a[e3] = a[e2];
                        a[e2] = t;
                        if (t < a[e1]) {
                            a[e2] = a[e1];
                            a[e1] = t;
                        }
                    }
                }
            }

            // 指针
            int less = left;  // 中心部分第一个元素的索引
            int great = right; // 右部分第一个元素前的索引

            if (a[e1] != a[e2] && a[e2] != a[e3] && a[e3] != a[e4] && a[e4] != a[e5]) {
            /*
             * 使用五个已排序元素中的第二个和第四个元素作为轴心。
             * 这些值是数组第一和第二个词组的廉价近似值。注意，Pivot1<=Pivot2。
             */
                int pivot1 = a[e2];
                int pivot2 = a[e4];

            /*
             * 要排序的第一个和最后一个元素将移动到以前由枢轴占用的位置。
             * 当分区完成时，数据透视被交换回它们的最终位置，并从随后的排序中排除。
             */
                a[e2] = a[left];
                a[e4] = a[right];

            /*
             * 跳过小于或大于透视值的元素。
             */
                while (a[++less] < pivot1) ;
                while (a[--great] > pivot2) ;

            /*
             * 划分:
             *
             *   左边部分           中间部分                   右边部分
             * +--------------------------------------------------------------+
             * |  < pivot1  |  pivot1 <= && <= pivot2  |    ?    |  > pivot2  |
             * +--------------------------------------------------------------+
             *               ^                          ^       ^
             *               |                          |       |
             *              less                        k     great
             *
             * 不变式:
             *
             *              all in (左边, less)   < pivot1
             *    pivot1 <= all in [less, k)     <= pivot2
             *              all in (great, 右边) > pivot2
             *
             * Pointer k is the first index of ?-part.
             */
                outer:
                for (int k = less - 1; ++k <= great; ) {
                    int ak = a[k];
                    if (ak < pivot1) { // 移动 a[k] 到左边
                        a[k] = a[less];
                    /*
                     * 由于性能问题，这里和我们使用下面的 "a[i] = b; i++;" 而不是"a[i++] = b;"
                     */
                        a[less] = ak;
                        ++less;
                    } else if (ak > pivot2) { // 移动 a[k] to 到右边
                        while (a[great] > pivot2) {
                            if (great-- == k) {
                                break outer;
                            }
                        }
                        if (a[great] < pivot1) { // a[great] <= pivot2
                            a[k] = a[less];
                            a[less] = a[great];
                            ++less;
                        } else { // pivot1 <= a[great] <= pivot2
                            a[k] = a[great];
                        }
                    /*
                     * 由于性能问题，这里和我们使用下面的 "a[i] = b; i--;"  而不是"a[i--] = b;"
                     */
                        a[great] = ak;
                        --great;
                    }
                }

                // 将pivots调到最终位置
                a[left] = a[less - 1];
                a[less - 1] = pivot1;
                a[right] = a[great + 1];
                a[great + 1] = pivot2;

                // 以递归方式对左右部分排序，不包括已知的数据透视
                sort(a, left, less - 2, leftmost);
                sort(a, great + 2, right, false);

            /*
             * 如果中心部分太大（包括数组的4/7以上），请将内部轴值交换到末端。
             */
                if (less < e1 && e5 < great) {
                /*
                 * 跳过与透视值相等的元素。
                 */
                    while (a[less] == pivot1) {
                        ++less;
                    }

                    while (a[great] == pivot2) {
                        --great;
                    }

                /*
                 * 划分:
                 *
                 *   左边部分           中间部分                   右边部分
                 * +----------------------------------------------------------+
                 * | == pivot1 |  pivot1 < && < pivot2  |    ?    | == pivot2 |
                 * +----------------------------------------------------------+
                 *              ^                        ^       ^
                 *              |                        |       |
                 *             less                      k     great
                 *
                 * 不变式:
                 *
                 *              all in (*,  less) == pivot1
                 *     pivot1 < all in [less,  k)  < pivot2
                 *              all in (great, *) == pivot2
                 *
                 * Pointer k is the first index of ?-part.
                 */
                    outer:
                    for (int k = less - 1; ++k <= great; ) {
                        int ak = a[k];
                        if (ak == pivot1) { // 移动 a[k] 到左边
                            a[k] = a[less];
                            a[less] = ak;
                            ++less;
                        } else if (ak == pivot2) { // 移动 a[k] 到右边
                            while (a[great] == pivot2) {
                                if (great-- == k) {
                                    break outer;
                                }
                            }
                            if (a[great] == pivot1) { // a[great] < pivot2
                                a[k] = a[less];
                            /*
                             * 即使[great]等于Pivot1，如果[great]和Pivot1是不同符号的浮点零，
                             * 则赋值a[less]=Pivot1可能不正确。
                             * 因此，在浮动和双重排序方法中，
                             * 我们必须使用更精确的赋值a[less]=a[great]。
                             */
                                a[less] = pivot1;
                                ++less;
                            } else { // pivot1 < a[great] < pivot2
                                a[k] = a[great];
                            }
                            a[great] = ak;
                            --great;
                        }
                    }
                }

                // 递归排序中心部分
                sort(a, less, great, false);

            } else { // Partitioning with one pivot
            /*
             * Use the third of the five sorted elements as pivot.
             * This value is inexpensive approximation of the median.
             */
                int pivot = a[e3];

            /*
             * Partitioning degenerates to the traditional 3-way
             * (or "Dutch National Flag") schema:
             *
             *   left part    center part              right part
             * +-------------------------------------------------+
             * |  < pivot  |   == pivot   |     ?    |  > pivot  |
             * +-------------------------------------------------+
             *              ^              ^        ^
             *              |              |        |
             *             less            k      great
             *
             * Invariants:
             *
             *   all in (left, less)   < pivot
             *   all in [less, k)     == pivot
             *   all in (great, right) > pivot
             *
             * Pointer k is the first index of ?-part.
             */
                for (int k = less; k <= great; ++k) {
                    if (a[k] == pivot) {
                        continue;
                    }
                    int ak = a[k];
                    if (ak < pivot) { // Move a[k] to left part
                        a[k] = a[less];
                        a[less] = ak;
                        ++less;
                    } else { // a[k] > pivot - Move a[k] to right part
                        while (a[great] > pivot) {
                            --great;
                        }
                        if (a[great] < pivot) { // a[great] <= pivot
                            a[k] = a[less];
                            a[less] = a[great];
                            ++less;
                        } else { // a[great] == pivot
                        /*
                         * Even though a[great] equals to pivot, the
                         * assignment a[k] = pivot may be incorrect,
                         * if a[great] and pivot are floating-point
                         * zeros of different signs. Therefore in float
                         * and double sorting methods we have to use
                         * more accurate assignment a[k] = a[great].
                         */
                            a[k] = pivot;
                        }
                        a[great] = ak;
                        --great;
                    }
                }

            /*
             * Sort left and right parts recursively.
             * All elements from center part are equal
             * and, therefore, already sorted.
             */
                sort(a, left, less - 1, leftmost);
                sort(a, great + 1, right, false);
            }
        }

        /**
         * Sorts the specified range of the array using the given
         * workspace array slice if possible for merging
         *
         * @param a        the array to be sorted
         * @param left     the index of the first element, inclusive, to be sorted
         * @param right    the index of the last element, inclusive, to be sorted
         * @param work     a workspace array (slice)
         * @param workBase origin of usable space in work array
         * @param workLen  usable size of work array
         */
        static void sort(long[] a, int left, int right,
                         long[] work, int workBase, int workLen) {
            // Use Quicksort on small arrays
            if (right - left < QUICKSORT_THRESHOLD) {
                sort(a, left, right, true);
                return;
            }

        /*
         * Index run[i] is the start of i-th run
         * (ascending or descending sequence).
         */
            int[] run = new int[MAX_RUN_COUNT + 1];
            int count = 0;
            run[0] = left;

            // Check if the array is nearly sorted
            for (int k = left; k < right; run[count] = k) {
                if (a[k] < a[k + 1]) { // ascending
                    while (++k <= right && a[k - 1] <= a[k]) ;
                } else if (a[k] > a[k + 1]) { // descending
                    while (++k <= right && a[k - 1] >= a[k]) ;
                    for (int lo = run[count] - 1, hi = k; ++lo < --hi; ) {
                        long t = a[lo];
                        a[lo] = a[hi];
                        a[hi] = t;
                    }
                } else { // equal
                    for (int m = MAX_RUN_LENGTH; ++k <= right && a[k - 1] == a[k]; ) {
                        if (--m == 0) {
                            sort(a, left, right, true);
                            return;
                        }
                    }
                }

            /*
             * The array is not highly structured,
             * use Quicksort instead of merge sort.
             */
                if (++count == MAX_RUN_COUNT) {
                    sort(a, left, right, true);
                    return;
                }
            }

            // Check special cases
            // Implementation note: variable "right" is increased by 1.
            if (run[count] == right++) { // The last run contains one element
                run[++count] = right;
            } else if (count == 1) { // The array is already sorted
                return;
            }

            // Determine alternation base for merge
            byte odd = 0;
            for (int n = 1; (n <<= 1) < count; odd ^= 1) ;

            // Use or create temporary array b for merging
            long[] b;                 // temp array; alternates with a
            int ao, bo;              // array offsets from 'left'
            int blen = right - left; // space needed for b
            if (work == null || workLen < blen || workBase + blen > work.length) {
                work = new long[blen];
                workBase = 0;
            }
            if (odd == 0) {
                System.arraycopy(a, left, work, workBase, blen);
                b = a;
                bo = 0;
                a = work;
                ao = workBase - left;
            } else {
                b = work;
                ao = 0;
                bo = workBase - left;
            }

            // Merging
            for (int last; count > 1; count = last) {
                for (int k = (last = 0) + 2; k <= count; k += 2) {
                    int hi = run[k], mi = run[k - 1];
                    for (int i = run[k - 2], p = i, q = mi; i < hi; ++i) {
                        if (q >= hi || p < mi && a[p + ao] <= a[q + ao]) {
                            b[i + bo] = a[p++ + ao];
                        } else {
                            b[i + bo] = a[q++ + ao];
                        }
                    }
                    run[++last] = hi;
                }
                if ((count & 1) != 0) {
                    for (int i = right, lo = run[count - 1]; --i >= lo;
                         b[i + bo] = a[i + ao]
                            )
                        ;
                    run[++last] = right;
                }
                long[] t = a;
                a = b;
                b = t;
                int o = ao;
                ao = bo;
                bo = o;
            }
        }

        /**
         * Sorts the specified range of the array by Dual-Pivot Quicksort.
         *
         * @param a        the array to be sorted
         * @param left     the index of the first element, inclusive, to be sorted
         * @param right    the index of the last element, inclusive, to be sorted
         * @param leftmost indicates if this part is the leftmost in the range
         */
        private static void sort(long[] a, int left, int right, boolean leftmost) {
            int length = right - left + 1;

            // Use insertion sort on tiny arrays
            if (length < INSERTION_SORT_THRESHOLD) {
                if (leftmost) {
                /*
                 * Traditional (without sentinel) insertion sort,
                 * optimized for server VM, is used in case of
                 * the leftmost part.
                 */
                    for (int i = left, j = i; i < right; j = ++i) {
                        long ai = a[i + 1];
                        while (ai < a[j]) {
                            a[j + 1] = a[j];
                            if (j-- == left) {
                                break;
                            }
                        }
                        a[j + 1] = ai;
                    }
                } else {
                /*
                 * Skip the longest ascending sequence.
                 */
                    do {
                        if (left >= right) {
                            return;
                        }
                    } while (a[++left] >= a[left - 1]);

                /*
                 * Every element from adjoining part plays the role
                 * of sentinel, therefore this allows us to avoid the
                 * left range check on each iteration. Moreover, we use
                 * the more optimized algorithm, so called pair insertion
                 * sort, which is faster (in the context of Quicksort)
                 * than traditional implementation of insertion sort.
                 */
                    for (int k = left; ++left <= right; k = ++left) {
                        long a1 = a[k], a2 = a[left];

                        if (a1 < a2) {
                            a2 = a1;
                            a1 = a[left];
                        }
                        while (a1 < a[--k]) {
                            a[k + 2] = a[k];
                        }
                        a[++k + 1] = a1;

                        while (a2 < a[--k]) {
                            a[k + 1] = a[k];
                        }
                        a[k + 1] = a2;
                    }
                    long last = a[right];

                    while (last < a[--right]) {
                        a[right + 1] = a[right];
                    }
                    a[right + 1] = last;
                }
                return;
            }

            // Inexpensive approximation of length / 7
            int seventh = (length >> 3) + (length >> 6) + 1;

        /*
         * Sort five evenly spaced elements around (and including) the
         * center element in the range. These elements will be used for
         * pivot selection as described below. The choice for spacing
         * these elements was empirically determined to work well on
         * a wide variety of inputs.
         */
            int e3 = (left + right) >>> 1; // The midpoint
            int e2 = e3 - seventh;
            int e1 = e2 - seventh;
            int e4 = e3 + seventh;
            int e5 = e4 + seventh;

            // Sort these elements using insertion sort
            if (a[e2] < a[e1]) {
                long t = a[e2];
                a[e2] = a[e1];
                a[e1] = t;
            }

            if (a[e3] < a[e2]) {
                long t = a[e3];
                a[e3] = a[e2];
                a[e2] = t;
                if (t < a[e1]) {
                    a[e2] = a[e1];
                    a[e1] = t;
                }
            }
            if (a[e4] < a[e3]) {
                long t = a[e4];
                a[e4] = a[e3];
                a[e3] = t;
                if (t < a[e2]) {
                    a[e3] = a[e2];
                    a[e2] = t;
                    if (t < a[e1]) {
                        a[e2] = a[e1];
                        a[e1] = t;
                    }
                }
            }
            if (a[e5] < a[e4]) {
                long t = a[e5];
                a[e5] = a[e4];
                a[e4] = t;
                if (t < a[e3]) {
                    a[e4] = a[e3];
                    a[e3] = t;
                    if (t < a[e2]) {
                        a[e3] = a[e2];
                        a[e2] = t;
                        if (t < a[e1]) {
                            a[e2] = a[e1];
                            a[e1] = t;
                        }
                    }
                }
            }

            // Pointers
            int less = left;  // The index of the first element of center part
            int great = right; // The index before the first element of right part

            if (a[e1] != a[e2] && a[e2] != a[e3] && a[e3] != a[e4] && a[e4] != a[e5]) {
            /*
             * Use the second and fourth of the five sorted elements as pivots.
             * These values are inexpensive approximations of the first and
             * second terciles of the array. Note that pivot1 <= pivot2.
             */
                long pivot1 = a[e2];
                long pivot2 = a[e4];

            /*
             * The first and the last elements to be sorted are moved to the
             * locations formerly occupied by the pivots. When partitioning
             * is complete, the pivots are swapped back into their final
             * positions, and excluded from subsequent sorting.
             */
                a[e2] = a[left];
                a[e4] = a[right];

            /*
             * Skip elements, which are less or greater than pivot values.
             */
                while (a[++less] < pivot1) ;
                while (a[--great] > pivot2) ;

            /*
             * Partitioning:
             *
             *   left part           center part                   right part
             * +--------------------------------------------------------------+
             * |  < pivot1  |  pivot1 <= && <= pivot2  |    ?    |  > pivot2  |
             * +--------------------------------------------------------------+
             *               ^                          ^       ^
             *               |                          |       |
             *              less                        k     great
             *
             * Invariants:
             *
             *              all in (left, less)   < pivot1
             *    pivot1 <= all in [less, k)     <= pivot2
             *              all in (great, right) > pivot2
             *
             * Pointer k is the first index of ?-part.
             */
                outer:
                for (int k = less - 1; ++k <= great; ) {
                    long ak = a[k];
                    if (ak < pivot1) { // Move a[k] to left part
                        a[k] = a[less];
                    /*
                     * Here and below we use "a[i] = b; i++;" instead
                     * of "a[i++] = b;" due to performance issue.
                     */
                        a[less] = ak;
                        ++less;
                    } else if (ak > pivot2) { // Move a[k] to right part
                        while (a[great] > pivot2) {
                            if (great-- == k) {
                                break outer;
                            }
                        }
                        if (a[great] < pivot1) { // a[great] <= pivot2
                            a[k] = a[less];
                            a[less] = a[great];
                            ++less;
                        } else { // pivot1 <= a[great] <= pivot2
                            a[k] = a[great];
                        }
                    /*
                     * Here and below we use "a[i] = b; i--;" instead
                     * of "a[i--] = b;" due to performance issue.
                     */
                        a[great] = ak;
                        --great;
                    }
                }

                // Swap pivots into their final positions
                a[left] = a[less - 1];
                a[less - 1] = pivot1;
                a[right] = a[great + 1];
                a[great + 1] = pivot2;

                // Sort left and right parts recursively, excluding known pivots
                sort(a, left, less - 2, leftmost);
                sort(a, great + 2, right, false);

            /*
             * If center part is too large (comprises > 4/7 of the array),
             * swap internal pivot values to ends.
             */
                if (less < e1 && e5 < great) {
                /*
                 * Skip elements, which are equal to pivot values.
                 */
                    while (a[less] == pivot1) {
                        ++less;
                    }

                    while (a[great] == pivot2) {
                        --great;
                    }

                /*
                 * Partitioning:
                 *
                 *   left part         center part                  right part
                 * +----------------------------------------------------------+
                 * | == pivot1 |  pivot1 < && < pivot2  |    ?    | == pivot2 |
                 * +----------------------------------------------------------+
                 *              ^                        ^       ^
                 *              |                        |       |
                 *             less                      k     great
                 *
                 * Invariants:
                 *
                 *              all in (*,  less) == pivot1
                 *     pivot1 < all in [less,  k)  < pivot2
                 *              all in (great, *) == pivot2
                 *
                 * Pointer k is the first index of ?-part.
                 */
                    outer:
                    for (int k = less - 1; ++k <= great; ) {
                        long ak = a[k];
                        if (ak == pivot1) { // Move a[k] to left part
                            a[k] = a[less];
                            a[less] = ak;
                            ++less;
                        } else if (ak == pivot2) { // Move a[k] to right part
                            while (a[great] == pivot2) {
                                if (great-- == k) {
                                    break outer;
                                }
                            }
                            if (a[great] == pivot1) { // a[great] < pivot2
                                a[k] = a[less];
                            /*
                             * Even though a[great] equals to pivot1, the
                             * assignment a[less] = pivot1 may be incorrect,
                             * if a[great] and pivot1 are floating-point zeros
                             * of different signs. Therefore in float and
                             * double sorting methods we have to use more
                             * accurate assignment a[less] = a[great].
                             */
                                a[less] = pivot1;
                                ++less;
                            } else { // pivot1 < a[great] < pivot2
                                a[k] = a[great];
                            }
                            a[great] = ak;
                            --great;
                        }
                    }
                }

                // Sort center part recursively
                sort(a, less, great, false);

            } else { // Partitioning with one pivot
            /*
             * Use the third of the five sorted elements as pivot.
             * This value is inexpensive approximation of the median.
             */
                long pivot = a[e3];

            /*
             * Partitioning degenerates to the traditional 3-way
             * (or "Dutch National Flag") schema:
             *
             *   left part    center part              right part
             * +-------------------------------------------------+
             * |  < pivot  |   == pivot   |     ?    |  > pivot  |
             * +-------------------------------------------------+
             *              ^              ^        ^
             *              |              |        |
             *             less            k      great
             *
             * Invariants:
             *
             *   all in (left, less)   < pivot
             *   all in [less, k)     == pivot
             *   all in (great, right) > pivot
             *
             * Pointer k is the first index of ?-part.
             */
                for (int k = less; k <= great; ++k) {
                    if (a[k] == pivot) {
                        continue;
                    }
                    long ak = a[k];
                    if (ak < pivot) { // Move a[k] to left part
                        a[k] = a[less];
                        a[less] = ak;
                        ++less;
                    } else { // a[k] > pivot - Move a[k] to right part
                        while (a[great] > pivot) {
                            --great;
                        }
                        if (a[great] < pivot) { // a[great] <= pivot
                            a[k] = a[less];
                            a[less] = a[great];
                            ++less;
                        } else { // a[great] == pivot
                        /*
                         * Even though a[great] equals to pivot, the
                         * assignment a[k] = pivot may be incorrect,
                         * if a[great] and pivot are floating-point
                         * zeros of different signs. Therefore in float
                         * and double sorting methods we have to use
                         * more accurate assignment a[k] = a[great].
                         */
                            a[k] = pivot;
                        }
                        a[great] = ak;
                        --great;
                    }
                }

            /*
             * Sort left and right parts recursively.
             * All elements from center part are equal
             * and, therefore, already sorted.
             */
                sort(a, left, less - 1, leftmost);
                sort(a, great + 1, right, false);
            }
        }

        /**
         * Sorts the specified range of the array using the given
         * workspace array slice if possible for merging
         *
         * @param a        the array to be sorted
         * @param left     the index of the first element, inclusive, to be sorted
         * @param right    the index of the last element, inclusive, to be sorted
         * @param work     a workspace array (slice)
         * @param workBase origin of usable space in work array
         * @param workLen  usable size of work array
         */
        static void sort(short[] a, int left, int right,
                         short[] work, int workBase, int workLen) {
            // Use counting sort on large arrays
            if (right - left > COUNTING_SORT_THRESHOLD_FOR_SHORT_OR_CHAR) {
                int[] count = new int[NUM_SHORT_VALUES];

                for (int i = left - 1; ++i <= right;
                     count[a[i] - Short.MIN_VALUE]++
                        )
                    ;
                for (int i = NUM_SHORT_VALUES, k = right + 1; k > left; ) {
                    while (count[--i] == 0) ;
                    short value = (short) (i + Short.MIN_VALUE);
                    int s = count[i];

                    do {
                        a[--k] = value;
                    } while (--s > 0);
                }
            } else { // Use Dual-Pivot Quicksort on small arrays
                doSort(a, left, right, work, workBase, workLen);
            }
        }

        /**
         * The number of distinct short values.
         */
        private static final int NUM_SHORT_VALUES = 1 << 16;

        /**
         * Sorts the specified range of the array.
         *
         * @param a        the array to be sorted
         * @param left     the index of the first element, inclusive, to be sorted
         * @param right    the index of the last element, inclusive, to be sorted
         * @param work     a workspace array (slice)
         * @param workBase origin of usable space in work array
         * @param workLen  usable size of work array
         */
        private static void doSort(short[] a, int left, int right,
                                   short[] work, int workBase, int workLen) {
            // Use Quicksort on small arrays
            if (right - left < QUICKSORT_THRESHOLD) {
                sort(a, left, right, true);
                return;
            }

        /*
         * Index run[i] is the start of i-th run
         * (ascending or descending sequence).
         */
            int[] run = new int[MAX_RUN_COUNT + 1];
            int count = 0;
            run[0] = left;

            // Check if the array is nearly sorted
            for (int k = left; k < right; run[count] = k) {
                if (a[k] < a[k + 1]) { // ascending
                    while (++k <= right && a[k - 1] <= a[k]) ;
                } else if (a[k] > a[k + 1]) { // descending
                    while (++k <= right && a[k - 1] >= a[k]) ;
                    for (int lo = run[count] - 1, hi = k; ++lo < --hi; ) {
                        short t = a[lo];
                        a[lo] = a[hi];
                        a[hi] = t;
                    }
                } else { // equal
                    for (int m = MAX_RUN_LENGTH; ++k <= right && a[k - 1] == a[k]; ) {
                        if (--m == 0) {
                            sort(a, left, right, true);
                            return;
                        }
                    }
                }

            /*
             * The array is not highly structured,
             * use Quicksort instead of merge sort.
             */
                if (++count == MAX_RUN_COUNT) {
                    sort(a, left, right, true);
                    return;
                }
            }

            // Check special cases
            // Implementation note: variable "right" is increased by 1.
            if (run[count] == right++) { // The last run contains one element
                run[++count] = right;
            } else if (count == 1) { // The array is already sorted
                return;
            }

            // Determine alternation base for merge
            byte odd = 0;
            for (int n = 1; (n <<= 1) < count; odd ^= 1) ;

            // Use or create temporary array b for merging
            short[] b;                 // temp array; alternates with a
            int ao, bo;              // array offsets from 'left'
            int blen = right - left; // space needed for b
            if (work == null || workLen < blen || workBase + blen > work.length) {
                work = new short[blen];
                workBase = 0;
            }
            if (odd == 0) {
                System.arraycopy(a, left, work, workBase, blen);
                b = a;
                bo = 0;
                a = work;
                ao = workBase - left;
            } else {
                b = work;
                ao = 0;
                bo = workBase - left;
            }

            // Merging
            for (int last; count > 1; count = last) {
                for (int k = (last = 0) + 2; k <= count; k += 2) {
                    int hi = run[k], mi = run[k - 1];
                    for (int i = run[k - 2], p = i, q = mi; i < hi; ++i) {
                        if (q >= hi || p < mi && a[p + ao] <= a[q + ao]) {
                            b[i + bo] = a[p++ + ao];
                        } else {
                            b[i + bo] = a[q++ + ao];
                        }
                    }
                    run[++last] = hi;
                }
                if ((count & 1) != 0) {
                    for (int i = right, lo = run[count - 1]; --i >= lo;
                         b[i + bo] = a[i + ao]
                            )
                        ;
                    run[++last] = right;
                }
                short[] t = a;
                a = b;
                b = t;
                int o = ao;
                ao = bo;
                bo = o;
            }
        }

        /**
         * Sorts the specified range of the array by Dual-Pivot Quicksort.
         *
         * @param a        the array to be sorted
         * @param left     the index of the first element, inclusive, to be sorted
         * @param right    the index of the last element, inclusive, to be sorted
         * @param leftmost indicates if this part is the leftmost in the range
         */
        private static void sort(short[] a, int left, int right, boolean leftmost) {
            int length = right - left + 1;

            // Use insertion sort on tiny arrays
            if (length < INSERTION_SORT_THRESHOLD) {
                if (leftmost) {
                /*
                 * Traditional (without sentinel) insertion sort,
                 * optimized for server VM, is used in case of
                 * the leftmost part.
                 */
                    for (int i = left, j = i; i < right; j = ++i) {
                        short ai = a[i + 1];
                        while (ai < a[j]) {
                            a[j + 1] = a[j];
                            if (j-- == left) {
                                break;
                            }
                        }
                        a[j + 1] = ai;
                    }
                } else {
                /*
                 * Skip the longest ascending sequence.
                 */
                    do {
                        if (left >= right) {
                            return;
                        }
                    } while (a[++left] >= a[left - 1]);

                /*
                 * Every element from adjoining part plays the role
                 * of sentinel, therefore this allows us to avoid the
                 * left range check on each iteration. Moreover, we use
                 * the more optimized algorithm, so called pair insertion
                 * sort, which is faster (in the context of Quicksort)
                 * than traditional implementation of insertion sort.
                 */
                    for (int k = left; ++left <= right; k = ++left) {
                        short a1 = a[k], a2 = a[left];

                        if (a1 < a2) {
                            a2 = a1;
                            a1 = a[left];
                        }
                        while (a1 < a[--k]) {
                            a[k + 2] = a[k];
                        }
                        a[++k + 1] = a1;

                        while (a2 < a[--k]) {
                            a[k + 1] = a[k];
                        }
                        a[k + 1] = a2;
                    }
                    short last = a[right];

                    while (last < a[--right]) {
                        a[right + 1] = a[right];
                    }
                    a[right + 1] = last;
                }
                return;
            }

            // Inexpensive approximation of length / 7
            int seventh = (length >> 3) + (length >> 6) + 1;

        /*
         * Sort five evenly spaced elements around (and including) the
         * center element in the range. These elements will be used for
         * pivot selection as described below. The choice for spacing
         * these elements was empirically determined to work well on
         * a wide variety of inputs.
         */
            int e3 = (left + right) >>> 1; // The midpoint
            int e2 = e3 - seventh;
            int e1 = e2 - seventh;
            int e4 = e3 + seventh;
            int e5 = e4 + seventh;

            // Sort these elements using insertion sort
            if (a[e2] < a[e1]) {
                short t = a[e2];
                a[e2] = a[e1];
                a[e1] = t;
            }

            if (a[e3] < a[e2]) {
                short t = a[e3];
                a[e3] = a[e2];
                a[e2] = t;
                if (t < a[e1]) {
                    a[e2] = a[e1];
                    a[e1] = t;
                }
            }
            if (a[e4] < a[e3]) {
                short t = a[e4];
                a[e4] = a[e3];
                a[e3] = t;
                if (t < a[e2]) {
                    a[e3] = a[e2];
                    a[e2] = t;
                    if (t < a[e1]) {
                        a[e2] = a[e1];
                        a[e1] = t;
                    }
                }
            }
            if (a[e5] < a[e4]) {
                short t = a[e5];
                a[e5] = a[e4];
                a[e4] = t;
                if (t < a[e3]) {
                    a[e4] = a[e3];
                    a[e3] = t;
                    if (t < a[e2]) {
                        a[e3] = a[e2];
                        a[e2] = t;
                        if (t < a[e1]) {
                            a[e2] = a[e1];
                            a[e1] = t;
                        }
                    }
                }
            }

            // Pointers
            int less = left;  // The index of the first element of center part
            int great = right; // The index before the first element of right part

            if (a[e1] != a[e2] && a[e2] != a[e3] && a[e3] != a[e4] && a[e4] != a[e5]) {
            /*
             * Use the second and fourth of the five sorted elements as pivots.
             * These values are inexpensive approximations of the first and
             * second terciles of the array. Note that pivot1 <= pivot2.
             */
                short pivot1 = a[e2];
                short pivot2 = a[e4];

            /*
             * The first and the last elements to be sorted are moved to the
             * locations formerly occupied by the pivots. When partitioning
             * is complete, the pivots are swapped back into their final
             * positions, and excluded from subsequent sorting.
             */
                a[e2] = a[left];
                a[e4] = a[right];

            /*
             * Skip elements, which are less or greater than pivot values.
             */
                while (a[++less] < pivot1) ;
                while (a[--great] > pivot2) ;

            /*
             * Partitioning:
             *
             *   left part           center part                   right part
             * +--------------------------------------------------------------+
             * |  < pivot1  |  pivot1 <= && <= pivot2  |    ?    |  > pivot2  |
             * +--------------------------------------------------------------+
             *               ^                          ^       ^
             *               |                          |       |
             *              less                        k     great
             *
             * Invariants:
             *
             *              all in (left, less)   < pivot1
             *    pivot1 <= all in [less, k)     <= pivot2
             *              all in (great, right) > pivot2
             *
             * Pointer k is the first index of ?-part.
             */
                outer:
                for (int k = less - 1; ++k <= great; ) {
                    short ak = a[k];
                    if (ak < pivot1) { // Move a[k] to left part
                        a[k] = a[less];
                    /*
                     * Here and below we use "a[i] = b; i++;" instead
                     * of "a[i++] = b;" due to performance issue.
                     */
                        a[less] = ak;
                        ++less;
                    } else if (ak > pivot2) { // Move a[k] to right part
                        while (a[great] > pivot2) {
                            if (great-- == k) {
                                break outer;
                            }
                        }
                        if (a[great] < pivot1) { // a[great] <= pivot2
                            a[k] = a[less];
                            a[less] = a[great];
                            ++less;
                        } else { // pivot1 <= a[great] <= pivot2
                            a[k] = a[great];
                        }
                    /*
                     * Here and below we use "a[i] = b; i--;" instead
                     * of "a[i--] = b;" due to performance issue.
                     */
                        a[great] = ak;
                        --great;
                    }
                }

                // Swap pivots into their final positions
                a[left] = a[less - 1];
                a[less - 1] = pivot1;
                a[right] = a[great + 1];
                a[great + 1] = pivot2;

                // Sort left and right parts recursively, excluding known pivots
                sort(a, left, less - 2, leftmost);
                sort(a, great + 2, right, false);

            /*
             * If center part is too large (comprises > 4/7 of the array),
             * swap internal pivot values to ends.
             */
                if (less < e1 && e5 < great) {
                /*
                 * Skip elements, which are equal to pivot values.
                 */
                    while (a[less] == pivot1) {
                        ++less;
                    }

                    while (a[great] == pivot2) {
                        --great;
                    }

                /*
                 * Partitioning:
                 *
                 *   left part         center part                  right part
                 * +----------------------------------------------------------+
                 * | == pivot1 |  pivot1 < && < pivot2  |    ?    | == pivot2 |
                 * +----------------------------------------------------------+
                 *              ^                        ^       ^
                 *              |                        |       |
                 *             less                      k     great
                 *
                 * Invariants:
                 *
                 *              all in (*,  less) == pivot1
                 *     pivot1 < all in [less,  k)  < pivot2
                 *              all in (great, *) == pivot2
                 *
                 * Pointer k is the first index of ?-part.
                 */
                    outer:
                    for (int k = less - 1; ++k <= great; ) {
                        short ak = a[k];
                        if (ak == pivot1) { // Move a[k] to left part
                            a[k] = a[less];
                            a[less] = ak;
                            ++less;
                        } else if (ak == pivot2) { // Move a[k] to right part
                            while (a[great] == pivot2) {
                                if (great-- == k) {
                                    break outer;
                                }
                            }
                            if (a[great] == pivot1) { // a[great] < pivot2
                                a[k] = a[less];
                            /*
                             * Even though a[great] equals to pivot1, the
                             * assignment a[less] = pivot1 may be incorrect,
                             * if a[great] and pivot1 are floating-point zeros
                             * of different signs. Therefore in float and
                             * double sorting methods we have to use more
                             * accurate assignment a[less] = a[great].
                             */
                                a[less] = pivot1;
                                ++less;
                            } else { // pivot1 < a[great] < pivot2
                                a[k] = a[great];
                            }
                            a[great] = ak;
                            --great;
                        }
                    }
                }

                // Sort center part recursively
                sort(a, less, great, false);

            } else { // Partitioning with one pivot
            /*
             * Use the third of the five sorted elements as pivot.
             * This value is inexpensive approximation of the median.
             */
                short pivot = a[e3];

            /*
             * Partitioning degenerates to the traditional 3-way
             * (or "Dutch National Flag") schema:
             *
             *   left part    center part              right part
             * +-------------------------------------------------+
             * |  < pivot  |   == pivot   |     ?    |  > pivot  |
             * +-------------------------------------------------+
             *              ^              ^        ^
             *              |              |        |
             *             less            k      great
             *
             * Invariants:
             *
             *   all in (left, less)   < pivot
             *   all in [less, k)     == pivot
             *   all in (great, right) > pivot
             *
             * Pointer k is the first index of ?-part.
             */
                for (int k = less; k <= great; ++k) {
                    if (a[k] == pivot) {
                        continue;
                    }
                    short ak = a[k];
                    if (ak < pivot) { // Move a[k] to left part
                        a[k] = a[less];
                        a[less] = ak;
                        ++less;
                    } else { // a[k] > pivot - Move a[k] to right part
                        while (a[great] > pivot) {
                            --great;
                        }
                        if (a[great] < pivot) { // a[great] <= pivot
                            a[k] = a[less];
                            a[less] = a[great];
                            ++less;
                        } else { // a[great] == pivot
                        /*
                         * Even though a[great] equals to pivot, the
                         * assignment a[k] = pivot may be incorrect,
                         * if a[great] and pivot are floating-point
                         * zeros of different signs. Therefore in float
                         * and double sorting methods we have to use
                         * more accurate assignment a[k] = a[great].
                         */
                            a[k] = pivot;
                        }
                        a[great] = ak;
                        --great;
                    }
                }

            /*
             * Sort left and right parts recursively.
             * All elements from center part are equal
             * and, therefore, already sorted.
             */
                sort(a, left, less - 1, leftmost);
                sort(a, great + 1, right, false);
            }
        }

        /**
         * Sorts the specified range of the array using the given
         * workspace array slice if possible for merging
         *
         * @param a        the array to be sorted
         * @param left     the index of the first element, inclusive, to be sorted
         * @param right    the index of the last element, inclusive, to be sorted
         * @param work     a workspace array (slice)
         * @param workBase origin of usable space in work array
         * @param workLen  usable size of work array
         */
        static void sort(char[] a, int left, int right,
                         char[] work, int workBase, int workLen) {
            // Use counting sort on large arrays
            if (right - left > COUNTING_SORT_THRESHOLD_FOR_SHORT_OR_CHAR) {
                int[] count = new int[NUM_CHAR_VALUES];

                for (int i = left - 1; ++i <= right;
                     count[a[i]]++
                        )
                    ;
                for (int i = NUM_CHAR_VALUES, k = right + 1; k > left; ) {
                    while (count[--i] == 0) ;
                    char value = (char) i;
                    int s = count[i];

                    do {
                        a[--k] = value;
                    } while (--s > 0);
                }
            } else { // Use Dual-Pivot Quicksort on small arrays
                doSort(a, left, right, work, workBase, workLen);
            }
        }

        /**
         * The number of distinct char values.
         */
        private static final int NUM_CHAR_VALUES = 1 << 16;

        /**
         * Sorts the specified range of the array.
         *
         * @param a        the array to be sorted
         * @param left     the index of the first element, inclusive, to be sorted
         * @param right    the index of the last element, inclusive, to be sorted
         * @param work     a workspace array (slice)
         * @param workBase origin of usable space in work array
         * @param workLen  usable size of work array
         */
        private static void doSort(char[] a, int left, int right,
                                   char[] work, int workBase, int workLen) {
            // Use Quicksort on small arrays
            if (right - left < QUICKSORT_THRESHOLD) {
                sort(a, left, right, true);
                return;
            }

        /*
         * Index run[i] is the start of i-th run
         * (ascending or descending sequence).
         */
            int[] run = new int[MAX_RUN_COUNT + 1];
            int count = 0;
            run[0] = left;

            // Check if the array is nearly sorted
            for (int k = left; k < right; run[count] = k) {
                if (a[k] < a[k + 1]) { // ascending
                    while (++k <= right && a[k - 1] <= a[k]) ;
                } else if (a[k] > a[k + 1]) { // descending
                    while (++k <= right && a[k - 1] >= a[k]) ;
                    for (int lo = run[count] - 1, hi = k; ++lo < --hi; ) {
                        char t = a[lo];
                        a[lo] = a[hi];
                        a[hi] = t;
                    }
                } else { // equal
                    for (int m = MAX_RUN_LENGTH; ++k <= right && a[k - 1] == a[k]; ) {
                        if (--m == 0) {
                            sort(a, left, right, true);
                            return;
                        }
                    }
                }

            /*
             * The array is not highly structured,
             * use Quicksort instead of merge sort.
             */
                if (++count == MAX_RUN_COUNT) {
                    sort(a, left, right, true);
                    return;
                }
            }

            // Check special cases
            // Implementation note: variable "right" is increased by 1.
            if (run[count] == right++) { // The last run contains one element
                run[++count] = right;
            } else if (count == 1) { // The array is already sorted
                return;
            }

            // Determine alternation base for merge
            byte odd = 0;
            for (int n = 1; (n <<= 1) < count; odd ^= 1) ;

            // Use or create temporary array b for merging
            char[] b;                 // temp array; alternates with a
            int ao, bo;              // array offsets from 'left'
            int blen = right - left; // space needed for b
            if (work == null || workLen < blen || workBase + blen > work.length) {
                work = new char[blen];
                workBase = 0;
            }
            if (odd == 0) {
                System.arraycopy(a, left, work, workBase, blen);
                b = a;
                bo = 0;
                a = work;
                ao = workBase - left;
            } else {
                b = work;
                ao = 0;
                bo = workBase - left;
            }

            // Merging
            for (int last; count > 1; count = last) {
                for (int k = (last = 0) + 2; k <= count; k += 2) {
                    int hi = run[k], mi = run[k - 1];
                    for (int i = run[k - 2], p = i, q = mi; i < hi; ++i) {
                        if (q >= hi || p < mi && a[p + ao] <= a[q + ao]) {
                            b[i + bo] = a[p++ + ao];
                        } else {
                            b[i + bo] = a[q++ + ao];
                        }
                    }
                    run[++last] = hi;
                }
                if ((count & 1) != 0) {
                    for (int i = right, lo = run[count - 1]; --i >= lo;
                         b[i + bo] = a[i + ao]
                            )
                        ;
                    run[++last] = right;
                }
                char[] t = a;
                a = b;
                b = t;
                int o = ao;
                ao = bo;
                bo = o;
            }
        }

        /**
         * Sorts the specified range of the array by Dual-Pivot Quicksort.
         *
         * @param a        the array to be sorted
         * @param left     the index of the first element, inclusive, to be sorted
         * @param right    the index of the last element, inclusive, to be sorted
         * @param leftmost indicates if this part is the leftmost in the range
         */
        private static void sort(char[] a, int left, int right, boolean leftmost) {
            int length = right - left + 1;

            // Use insertion sort on tiny arrays
            if (length < INSERTION_SORT_THRESHOLD) {
                if (leftmost) {
                /*
                 * Traditional (without sentinel) insertion sort,
                 * optimized for server VM, is used in case of
                 * the leftmost part.
                 */
                    for (int i = left, j = i; i < right; j = ++i) {
                        char ai = a[i + 1];
                        while (ai < a[j]) {
                            a[j + 1] = a[j];
                            if (j-- == left) {
                                break;
                            }
                        }
                        a[j + 1] = ai;
                    }
                } else {
                /*
                 * Skip the longest ascending sequence.
                 */
                    do {
                        if (left >= right) {
                            return;
                        }
                    } while (a[++left] >= a[left - 1]);

                /*
                 * Every element from adjoining part plays the role
                 * of sentinel, therefore this allows us to avoid the
                 * left range check on each iteration. Moreover, we use
                 * the more optimized algorithm, so called pair insertion
                 * sort, which is faster (in the context of Quicksort)
                 * than traditional implementation of insertion sort.
                 */
                    for (int k = left; ++left <= right; k = ++left) {
                        char a1 = a[k], a2 = a[left];

                        if (a1 < a2) {
                            a2 = a1;
                            a1 = a[left];
                        }
                        while (a1 < a[--k]) {
                            a[k + 2] = a[k];
                        }
                        a[++k + 1] = a1;

                        while (a2 < a[--k]) {
                            a[k + 1] = a[k];
                        }
                        a[k + 1] = a2;
                    }
                    char last = a[right];

                    while (last < a[--right]) {
                        a[right + 1] = a[right];
                    }
                    a[right + 1] = last;
                }
                return;
            }

            // Inexpensive approximation of length / 7
            int seventh = (length >> 3) + (length >> 6) + 1;

        /*
         * Sort five evenly spaced elements around (and including) the
         * center element in the range. These elements will be used for
         * pivot selection as described below. The choice for spacing
         * these elements was empirically determined to work well on
         * a wide variety of inputs.
         */
            int e3 = (left + right) >>> 1; // The midpoint
            int e2 = e3 - seventh;
            int e1 = e2 - seventh;
            int e4 = e3 + seventh;
            int e5 = e4 + seventh;

            // Sort these elements using insertion sort
            if (a[e2] < a[e1]) {
                char t = a[e2];
                a[e2] = a[e1];
                a[e1] = t;
            }

            if (a[e3] < a[e2]) {
                char t = a[e3];
                a[e3] = a[e2];
                a[e2] = t;
                if (t < a[e1]) {
                    a[e2] = a[e1];
                    a[e1] = t;
                }
            }
            if (a[e4] < a[e3]) {
                char t = a[e4];
                a[e4] = a[e3];
                a[e3] = t;
                if (t < a[e2]) {
                    a[e3] = a[e2];
                    a[e2] = t;
                    if (t < a[e1]) {
                        a[e2] = a[e1];
                        a[e1] = t;
                    }
                }
            }
            if (a[e5] < a[e4]) {
                char t = a[e5];
                a[e5] = a[e4];
                a[e4] = t;
                if (t < a[e3]) {
                    a[e4] = a[e3];
                    a[e3] = t;
                    if (t < a[e2]) {
                        a[e3] = a[e2];
                        a[e2] = t;
                        if (t < a[e1]) {
                            a[e2] = a[e1];
                            a[e1] = t;
                        }
                    }
                }
            }

            // Pointers
            int less = left;  // The index of the first element of center part
            int great = right; // The index before the first element of right part

            if (a[e1] != a[e2] && a[e2] != a[e3] && a[e3] != a[e4] && a[e4] != a[e5]) {
            /*
             * Use the second and fourth of the five sorted elements as pivots.
             * These values are inexpensive approximations of the first and
             * second terciles of the array. Note that pivot1 <= pivot2.
             */
                char pivot1 = a[e2];
                char pivot2 = a[e4];

            /*
             * The first and the last elements to be sorted are moved to the
             * locations formerly occupied by the pivots. When partitioning
             * is complete, the pivots are swapped back into their final
             * positions, and excluded from subsequent sorting.
             */
                a[e2] = a[left];
                a[e4] = a[right];

            /*
             * Skip elements, which are less or greater than pivot values.
             */
                while (a[++less] < pivot1) ;
                while (a[--great] > pivot2) ;

            /*
             * Partitioning:
             *
             *   left part           center part                   right part
             * +--------------------------------------------------------------+
             * |  < pivot1  |  pivot1 <= && <= pivot2  |    ?    |  > pivot2  |
             * +--------------------------------------------------------------+
             *               ^                          ^       ^
             *               |                          |       |
             *              less                        k     great
             *
             * Invariants:
             *
             *              all in (left, less)   < pivot1
             *    pivot1 <= all in [less, k)     <= pivot2
             *              all in (great, right) > pivot2
             *
             * Pointer k is the first index of ?-part.
             */
                outer:
                for (int k = less - 1; ++k <= great; ) {
                    char ak = a[k];
                    if (ak < pivot1) { // Move a[k] to left part
                        a[k] = a[less];
                    /*
                     * Here and below we use "a[i] = b; i++;" instead
                     * of "a[i++] = b;" due to performance issue.
                     */
                        a[less] = ak;
                        ++less;
                    } else if (ak > pivot2) { // Move a[k] to right part
                        while (a[great] > pivot2) {
                            if (great-- == k) {
                                break outer;
                            }
                        }
                        if (a[great] < pivot1) { // a[great] <= pivot2
                            a[k] = a[less];
                            a[less] = a[great];
                            ++less;
                        } else { // pivot1 <= a[great] <= pivot2
                            a[k] = a[great];
                        }
                    /*
                     * Here and below we use "a[i] = b; i--;" instead
                     * of "a[i--] = b;" due to performance issue.
                     */
                        a[great] = ak;
                        --great;
                    }
                }

                // Swap pivots into their final positions
                a[left] = a[less - 1];
                a[less - 1] = pivot1;
                a[right] = a[great + 1];
                a[great + 1] = pivot2;

                // Sort left and right parts recursively, excluding known pivots
                sort(a, left, less - 2, leftmost);
                sort(a, great + 2, right, false);

            /*
             * If center part is too large (comprises > 4/7 of the array),
             * swap internal pivot values to ends.
             */
                if (less < e1 && e5 < great) {
                /*
                 * Skip elements, which are equal to pivot values.
                 */
                    while (a[less] == pivot1) {
                        ++less;
                    }

                    while (a[great] == pivot2) {
                        --great;
                    }

                /*
                 * Partitioning:
                 *
                 *   left part         center part                  right part
                 * +----------------------------------------------------------+
                 * | == pivot1 |  pivot1 < && < pivot2  |    ?    | == pivot2 |
                 * +----------------------------------------------------------+
                 *              ^                        ^       ^
                 *              |                        |       |
                 *             less                      k     great
                 *
                 * Invariants:
                 *
                 *              all in (*,  less) == pivot1
                 *     pivot1 < all in [less,  k)  < pivot2
                 *              all in (great, *) == pivot2
                 *
                 * Pointer k is the first index of ?-part.
                 */
                    outer:
                    for (int k = less - 1; ++k <= great; ) {
                        char ak = a[k];
                        if (ak == pivot1) { // Move a[k] to left part
                            a[k] = a[less];
                            a[less] = ak;
                            ++less;
                        } else if (ak == pivot2) { // Move a[k] to right part
                            while (a[great] == pivot2) {
                                if (great-- == k) {
                                    break outer;
                                }
                            }
                            if (a[great] == pivot1) { // a[great] < pivot2
                                a[k] = a[less];
                            /*
                             * Even though a[great] equals to pivot1, the
                             * assignment a[less] = pivot1 may be incorrect,
                             * if a[great] and pivot1 are floating-point zeros
                             * of different signs. Therefore in float and
                             * double sorting methods we have to use more
                             * accurate assignment a[less] = a[great].
                             */
                                a[less] = pivot1;
                                ++less;
                            } else { // pivot1 < a[great] < pivot2
                                a[k] = a[great];
                            }
                            a[great] = ak;
                            --great;
                        }
                    }
                }

                // Sort center part recursively
                sort(a, less, great, false);

            } else { // Partitioning with one pivot
            /*
             * Use the third of the five sorted elements as pivot.
             * This value is inexpensive approximation of the median.
             */
                char pivot = a[e3];

            /*
             * Partitioning degenerates to the traditional 3-way
             * (or "Dutch National Flag") schema:
             *
             *   left part    center part              right part
             * +-------------------------------------------------+
             * |  < pivot  |   == pivot   |     ?    |  > pivot  |
             * +-------------------------------------------------+
             *              ^              ^        ^
             *              |              |        |
             *             less            k      great
             *
             * Invariants:
             *
             *   all in (left, less)   < pivot
             *   all in [less, k)     == pivot
             *   all in (great, right) > pivot
             *
             * Pointer k is the first index of ?-part.
             */
                for (int k = less; k <= great; ++k) {
                    if (a[k] == pivot) {
                        continue;
                    }
                    char ak = a[k];
                    if (ak < pivot) { // Move a[k] to left part
                        a[k] = a[less];
                        a[less] = ak;
                        ++less;
                    } else { // a[k] > pivot - Move a[k] to right part
                        while (a[great] > pivot) {
                            --great;
                        }
                        if (a[great] < pivot) { // a[great] <= pivot
                            a[k] = a[less];
                            a[less] = a[great];
                            ++less;
                        } else { // a[great] == pivot
                        /*
                         * Even though a[great] equals to pivot, the
                         * assignment a[k] = pivot may be incorrect,
                         * if a[great] and pivot are floating-point
                         * zeros of different signs. Therefore in float
                         * and double sorting methods we have to use
                         * more accurate assignment a[k] = a[great].
                         */
                            a[k] = pivot;
                        }
                        a[great] = ak;
                        --great;
                    }
                }

            /*
             * Sort left and right parts recursively.
             * All elements from center part are equal
             * and, therefore, already sorted.
             */
                sort(a, left, less - 1, leftmost);
                sort(a, great + 1, right, false);
            }
        }

        /**
         * The number of distinct byte values.
         */
        private static final int NUM_BYTE_VALUES = 1 << 8;

        /**
         * Sorts the specified range of the array.
         *
         * @param a     the array to be sorted
         * @param left  the index of the first element, inclusive, to be sorted
         * @param right the index of the last element, inclusive, to be sorted
         */
        static void sort(byte[] a, int left, int right) {
            // Use counting sort on large arrays
            if (right - left > COUNTING_SORT_THRESHOLD_FOR_BYTE) {
                int[] count = new int[NUM_BYTE_VALUES];

                for (int i = left - 1; ++i <= right;
                     count[a[i] - Byte.MIN_VALUE]++
                        )
                    ;
                for (int i = NUM_BYTE_VALUES, k = right + 1; k > left; ) {
                    while (count[--i] == 0) ;
                    byte value = (byte) (i + Byte.MIN_VALUE);
                    int s = count[i];

                    do {
                        a[--k] = value;
                    } while (--s > 0);
                }
            } else { // Use insertion sort on small arrays
                for (int i = left, j = i; i < right; j = ++i) {
                    byte ai = a[i + 1];
                    while (ai < a[j]) {
                        a[j + 1] = a[j];
                        if (j-- == left) {
                            break;
                        }
                    }
                    a[j + 1] = ai;
                }
            }
        }

        /**
         * Sorts the specified range of the array using the given
         * workspace array slice if possible for merging
         *
         * @param a        the array to be sorted
         * @param left     the index of the first element, inclusive, to be sorted
         * @param right    the index of the last element, inclusive, to be sorted
         * @param work     a workspace array (slice)
         * @param workBase origin of usable space in work array
         * @param workLen  usable size of work array
         */
        static void sort(float[] a, int left, int right,
                         float[] work, int workBase, int workLen) {
        /*
         * Phase 1: Move NaNs to the end of the array.
         */
            while (left <= right && Float.isNaN(a[right])) {
                --right;
            }
            for (int k = right; --k >= left; ) {
                float ak = a[k];
                if (ak != ak) { // a[k] is NaN
                    a[k] = a[right];
                    a[right] = ak;
                    --right;
                }
            }

        /*
         * Phase 2: Sort everything except NaNs (which are already in place).
         */
            doSort(a, left, right, work, workBase, workLen);

        /*
         * Phase 3: Place negative zeros before positive zeros.
         */
            int hi = right;

        /*
         * Find the first zero, or first positive, or last negative element.
         */
            while (left < hi) {
                int middle = (left + hi) >>> 1;
                float middleValue = a[middle];

                if (middleValue < 0.0f) {
                    left = middle + 1;
                } else {
                    hi = middle;
                }
            }

        /*
         * Skip the last negative value (if any) or all leading negative zeros.
         */
            while (left <= right && Float.floatToRawIntBits(a[left]) < 0) {
                ++left;
            }

        /*
         * Move negative zeros to the beginning of the sub-range.
         *
         * Partitioning:
         *
         * +----------------------------------------------------+
         * |   < 0.0   |   -0.0   |   0.0   |   ?  ( >= 0.0 )   |
         * +----------------------------------------------------+
         *              ^          ^         ^
         *              |          |         |
         *             left        p         k
         *
         * Invariants:
         *
         *   all in (*,  left)  <  0.0
         *   all in [left,  p) == -0.0
         *   all in [p,     k) ==  0.0
         *   all in [k, right] >=  0.0
         *
         * Pointer k is the first index of ?-part.
         */
            for (int k = left, p = left - 1; ++k <= right; ) {
                float ak = a[k];
                if (ak != 0.0f) {
                    break;
                }
                if (Float.floatToRawIntBits(ak) < 0) { // ak is -0.0f
                    a[k] = 0.0f;
                    a[++p] = -0.0f;
                }
            }
        }

        /**
         * Sorts the specified range of the array.
         *
         * @param a        the array to be sorted
         * @param left     the index of the first element, inclusive, to be sorted
         * @param right    the index of the last element, inclusive, to be sorted
         * @param work     a workspace array (slice)
         * @param workBase origin of usable space in work array
         * @param workLen  usable size of work array
         */
        private static void doSort(float[] a, int left, int right,
                                   float[] work, int workBase, int workLen) {
            // Use Quicksort on small arrays
            if (right - left < QUICKSORT_THRESHOLD) {
                sort(a, left, right, true);
                return;
            }

        /*
         * Index run[i] is the start of i-th run
         * (ascending or descending sequence).
         */
            int[] run = new int[MAX_RUN_COUNT + 1];
            int count = 0;
            run[0] = left;

            // Check if the array is nearly sorted
            for (int k = left; k < right; run[count] = k) {
                if (a[k] < a[k + 1]) { // ascending
                    while (++k <= right && a[k - 1] <= a[k]) ;
                } else if (a[k] > a[k + 1]) { // descending
                    while (++k <= right && a[k - 1] >= a[k]) ;
                    for (int lo = run[count] - 1, hi = k; ++lo < --hi; ) {
                        float t = a[lo];
                        a[lo] = a[hi];
                        a[hi] = t;
                    }
                } else { // equal
                    for (int m = MAX_RUN_LENGTH; ++k <= right && a[k - 1] == a[k]; ) {
                        if (--m == 0) {
                            sort(a, left, right, true);
                            return;
                        }
                    }
                }

            /*
             * The array is not highly structured,
             * use Quicksort instead of merge sort.
             */
                if (++count == MAX_RUN_COUNT) {
                    sort(a, left, right, true);
                    return;
                }
            }

            // Check special cases
            // Implementation note: variable "right" is increased by 1.
            if (run[count] == right++) { // The last run contains one element
                run[++count] = right;
            } else if (count == 1) { // The array is already sorted
                return;
            }

            // Determine alternation base for merge
            byte odd = 0;
            for (int n = 1; (n <<= 1) < count; odd ^= 1) ;

            // Use or create temporary array b for merging
            float[] b;                 // temp array; alternates with a
            int ao, bo;              // array offsets from 'left'
            int blen = right - left; // space needed for b
            if (work == null || workLen < blen || workBase + blen > work.length) {
                work = new float[blen];
                workBase = 0;
            }
            if (odd == 0) {
                System.arraycopy(a, left, work, workBase, blen);
                b = a;
                bo = 0;
                a = work;
                ao = workBase - left;
            } else {
                b = work;
                ao = 0;
                bo = workBase - left;
            }

            // Merging
            for (int last; count > 1; count = last) {
                for (int k = (last = 0) + 2; k <= count; k += 2) {
                    int hi = run[k], mi = run[k - 1];
                    for (int i = run[k - 2], p = i, q = mi; i < hi; ++i) {
                        if (q >= hi || p < mi && a[p + ao] <= a[q + ao]) {
                            b[i + bo] = a[p++ + ao];
                        } else {
                            b[i + bo] = a[q++ + ao];
                        }
                    }
                    run[++last] = hi;
                }
                if ((count & 1) != 0) {
                    for (int i = right, lo = run[count - 1]; --i >= lo;
                         b[i + bo] = a[i + ao]
                            )
                        ;
                    run[++last] = right;
                }
                float[] t = a;
                a = b;
                b = t;
                int o = ao;
                ao = bo;
                bo = o;
            }
        }

        /**
         * Sorts the specified range of the array by Dual-Pivot Quicksort.
         *
         * @param a        the array to be sorted
         * @param left     the index of the first element, inclusive, to be sorted
         * @param right    the index of the last element, inclusive, to be sorted
         * @param leftmost indicates if this part is the leftmost in the range
         */
        private static void sort(float[] a, int left, int right, boolean leftmost) {
            int length = right - left + 1;

            // Use insertion sort on tiny arrays
            if (length < INSERTION_SORT_THRESHOLD) {
                if (leftmost) {
                /*
                 * Traditional (without sentinel) insertion sort,
                 * optimized for server VM, is used in case of
                 * the leftmost part.
                 */
                    for (int i = left, j = i; i < right; j = ++i) {
                        float ai = a[i + 1];
                        while (ai < a[j]) {
                            a[j + 1] = a[j];
                            if (j-- == left) {
                                break;
                            }
                        }
                        a[j + 1] = ai;
                    }
                } else {
                /*
                 * Skip the longest ascending sequence.
                 */
                    do {
                        if (left >= right) {
                            return;
                        }
                    } while (a[++left] >= a[left - 1]);

                /*
                 * Every element from adjoining part plays the role
                 * of sentinel, therefore this allows us to avoid the
                 * left range check on each iteration. Moreover, we use
                 * the more optimized algorithm, so called pair insertion
                 * sort, which is faster (in the context of Quicksort)
                 * than traditional implementation of insertion sort.
                 */
                    for (int k = left; ++left <= right; k = ++left) {
                        float a1 = a[k], a2 = a[left];

                        if (a1 < a2) {
                            a2 = a1;
                            a1 = a[left];
                        }
                        while (a1 < a[--k]) {
                            a[k + 2] = a[k];
                        }
                        a[++k + 1] = a1;

                        while (a2 < a[--k]) {
                            a[k + 1] = a[k];
                        }
                        a[k + 1] = a2;
                    }
                    float last = a[right];

                    while (last < a[--right]) {
                        a[right + 1] = a[right];
                    }
                    a[right + 1] = last;
                }
                return;
            }

            // Inexpensive approximation of length / 7
            int seventh = (length >> 3) + (length >> 6) + 1;

        /*
         * Sort five evenly spaced elements around (and including) the
         * center element in the range. These elements will be used for
         * pivot selection as described below. The choice for spacing
         * these elements was empirically determined to work well on
         * a wide variety of inputs.
         */
            int e3 = (left + right) >>> 1; // The midpoint
            int e2 = e3 - seventh;
            int e1 = e2 - seventh;
            int e4 = e3 + seventh;
            int e5 = e4 + seventh;

            // Sort these elements using insertion sort
            if (a[e2] < a[e1]) {
                float t = a[e2];
                a[e2] = a[e1];
                a[e1] = t;
            }

            if (a[e3] < a[e2]) {
                float t = a[e3];
                a[e3] = a[e2];
                a[e2] = t;
                if (t < a[e1]) {
                    a[e2] = a[e1];
                    a[e1] = t;
                }
            }
            if (a[e4] < a[e3]) {
                float t = a[e4];
                a[e4] = a[e3];
                a[e3] = t;
                if (t < a[e2]) {
                    a[e3] = a[e2];
                    a[e2] = t;
                    if (t < a[e1]) {
                        a[e2] = a[e1];
                        a[e1] = t;
                    }
                }
            }
            if (a[e5] < a[e4]) {
                float t = a[e5];
                a[e5] = a[e4];
                a[e4] = t;
                if (t < a[e3]) {
                    a[e4] = a[e3];
                    a[e3] = t;
                    if (t < a[e2]) {
                        a[e3] = a[e2];
                        a[e2] = t;
                        if (t < a[e1]) {
                            a[e2] = a[e1];
                            a[e1] = t;
                        }
                    }
                }
            }

            // Pointers
            int less = left;  // The index of the first element of center part
            int great = right; // The index before the first element of right part

            if (a[e1] != a[e2] && a[e2] != a[e3] && a[e3] != a[e4] && a[e4] != a[e5]) {
            /*
             * Use the second and fourth of the five sorted elements as pivots.
             * These values are inexpensive approximations of the first and
             * second terciles of the array. Note that pivot1 <= pivot2.
             */
                float pivot1 = a[e2];
                float pivot2 = a[e4];

            /*
             * The first and the last elements to be sorted are moved to the
             * locations formerly occupied by the pivots. When partitioning
             * is complete, the pivots are swapped back into their final
             * positions, and excluded from subsequent sorting.
             */
                a[e2] = a[left];
                a[e4] = a[right];

            /*
             * Skip elements, which are less or greater than pivot values.
             */
                while (a[++less] < pivot1) ;
                while (a[--great] > pivot2) ;

            /*
             * Partitioning:
             *
             *   left part           center part                   right part
             * +--------------------------------------------------------------+
             * |  < pivot1  |  pivot1 <= && <= pivot2  |    ?    |  > pivot2  |
             * +--------------------------------------------------------------+
             *               ^                          ^       ^
             *               |                          |       |
             *              less                        k     great
             *
             * Invariants:
             *
             *              all in (left, less)   < pivot1
             *    pivot1 <= all in [less, k)     <= pivot2
             *              all in (great, right) > pivot2
             *
             * Pointer k is the first index of ?-part.
             */
                outer:
                for (int k = less - 1; ++k <= great; ) {
                    float ak = a[k];
                    if (ak < pivot1) { // Move a[k] to left part
                        a[k] = a[less];
                    /*
                     * Here and below we use "a[i] = b; i++;" instead
                     * of "a[i++] = b;" due to performance issue.
                     */
                        a[less] = ak;
                        ++less;
                    } else if (ak > pivot2) { // Move a[k] to right part
                        while (a[great] > pivot2) {
                            if (great-- == k) {
                                break outer;
                            }
                        }
                        if (a[great] < pivot1) { // a[great] <= pivot2
                            a[k] = a[less];
                            a[less] = a[great];
                            ++less;
                        } else { // pivot1 <= a[great] <= pivot2
                            a[k] = a[great];
                        }
                    /*
                     * Here and below we use "a[i] = b; i--;" instead
                     * of "a[i--] = b;" due to performance issue.
                     */
                        a[great] = ak;
                        --great;
                    }
                }

                // Swap pivots into their final positions
                a[left] = a[less - 1];
                a[less - 1] = pivot1;
                a[right] = a[great + 1];
                a[great + 1] = pivot2;

                // Sort left and right parts recursively, excluding known pivots
                sort(a, left, less - 2, leftmost);
                sort(a, great + 2, right, false);

            /*
             * If center part is too large (comprises > 4/7 of the array),
             * swap internal pivot values to ends.
             */
                if (less < e1 && e5 < great) {
                /*
                 * Skip elements, which are equal to pivot values.
                 */
                    while (a[less] == pivot1) {
                        ++less;
                    }

                    while (a[great] == pivot2) {
                        --great;
                    }

                /*
                 * Partitioning:
                 *
                 *   left part         center part                  right part
                 * +----------------------------------------------------------+
                 * | == pivot1 |  pivot1 < && < pivot2  |    ?    | == pivot2 |
                 * +----------------------------------------------------------+
                 *              ^                        ^       ^
                 *              |                        |       |
                 *             less                      k     great
                 *
                 * Invariants:
                 *
                 *              all in (*,  less) == pivot1
                 *     pivot1 < all in [less,  k)  < pivot2
                 *              all in (great, *) == pivot2
                 *
                 * Pointer k is the first index of ?-part.
                 */
                    outer:
                    for (int k = less - 1; ++k <= great; ) {
                        float ak = a[k];
                        if (ak == pivot1) { // Move a[k] to left part
                            a[k] = a[less];
                            a[less] = ak;
                            ++less;
                        } else if (ak == pivot2) { // Move a[k] to right part
                            while (a[great] == pivot2) {
                                if (great-- == k) {
                                    break outer;
                                }
                            }
                            if (a[great] == pivot1) { // a[great] < pivot2
                                a[k] = a[less];
                            /*
                             * Even though a[great] equals to pivot1, the
                             * assignment a[less] = pivot1 may be incorrect,
                             * if a[great] and pivot1 are floating-point zeros
                             * of different signs. Therefore in float and
                             * double sorting methods we have to use more
                             * accurate assignment a[less] = a[great].
                             */
                                a[less] = a[great];
                                ++less;
                            } else { // pivot1 < a[great] < pivot2
                                a[k] = a[great];
                            }
                            a[great] = ak;
                            --great;
                        }
                    }
                }

                // Sort center part recursively
                sort(a, less, great, false);

            } else { // Partitioning with one pivot
            /*
             * Use the third of the five sorted elements as pivot.
             * This value is inexpensive approximation of the median.
             */
                float pivot = a[e3];

            /*
             * Partitioning degenerates to the traditional 3-way
             * (or "Dutch National Flag") schema:
             *
             *   left part    center part              right part
             * +-------------------------------------------------+
             * |  < pivot  |   == pivot   |     ?    |  > pivot  |
             * +-------------------------------------------------+
             *              ^              ^        ^
             *              |              |        |
             *             less            k      great
             *
             * Invariants:
             *
             *   all in (left, less)   < pivot
             *   all in [less, k)     == pivot
             *   all in (great, right) > pivot
             *
             * Pointer k is the first index of ?-part.
             */
                for (int k = less; k <= great; ++k) {
                    if (a[k] == pivot) {
                        continue;
                    }
                    float ak = a[k];
                    if (ak < pivot) { // Move a[k] to left part
                        a[k] = a[less];
                        a[less] = ak;
                        ++less;
                    } else { // a[k] > pivot - Move a[k] to right part
                        while (a[great] > pivot) {
                            --great;
                        }
                        if (a[great] < pivot) { // a[great] <= pivot
                            a[k] = a[less];
                            a[less] = a[great];
                            ++less;
                        } else { // a[great] == pivot
                        /*
                         * Even though a[great] equals to pivot, the
                         * assignment a[k] = pivot may be incorrect,
                         * if a[great] and pivot are floating-point
                         * zeros of different signs. Therefore in float
                         * and double sorting methods we have to use
                         * more accurate assignment a[k] = a[great].
                         */
                            a[k] = a[great];
                        }
                        a[great] = ak;
                        --great;
                    }
                }

            /*
             * Sort left and right parts recursively.
             * All elements from center part are equal
             * and, therefore, already sorted.
             */
                sort(a, left, less - 1, leftmost);
                sort(a, great + 1, right, false);
            }
        }

        /**
         * Sorts the specified range of the array using the given
         * workspace array slice if possible for merging
         *
         * @param a        the array to be sorted
         * @param left     the index of the first element, inclusive, to be sorted
         * @param right    the index of the last element, inclusive, to be sorted
         * @param work     a workspace array (slice)
         * @param workBase origin of usable space in work array
         * @param workLen  usable size of work array
         */
        static void sort(double[] a, int left, int right,
                         double[] work, int workBase, int workLen) {
        /*
         * Phase 1: Move NaNs to the end of the array.
         */
            while (left <= right && Double.isNaN(a[right])) {
                --right;
            }
            for (int k = right; --k >= left; ) {
                double ak = a[k];
                if (ak != ak) { // a[k] is NaN
                    a[k] = a[right];
                    a[right] = ak;
                    --right;
                }
            }

        /*
         * Phase 2: Sort everything except NaNs (which are already in place).
         */
            doSort(a, left, right, work, workBase, workLen);

        /*
         * Phase 3: Place negative zeros before positive zeros.
         */
            int hi = right;

        /*
         * Find the first zero, or first positive, or last negative element.
         */
            while (left < hi) {
                int middle = (left + hi) >>> 1;
                double middleValue = a[middle];

                if (middleValue < 0.0d) {
                    left = middle + 1;
                } else {
                    hi = middle;
                }
            }

        /*
         * Skip the last negative value (if any) or all leading negative zeros.
         */
            while (left <= right && Double.doubleToRawLongBits(a[left]) < 0) {
                ++left;
            }

        /*
         * Move negative zeros to the beginning of the sub-range.
         *
         * Partitioning:
         *
         * +----------------------------------------------------+
         * |   < 0.0   |   -0.0   |   0.0   |   ?  ( >= 0.0 )   |
         * +----------------------------------------------------+
         *              ^          ^         ^
         *              |          |         |
         *             left        p         k
         *
         * Invariants:
         *
         *   all in (*,  left)  <  0.0
         *   all in [left,  p) == -0.0
         *   all in [p,     k) ==  0.0
         *   all in [k, right] >=  0.0
         *
         * Pointer k is the first index of ?-part.
         */
            for (int k = left, p = left - 1; ++k <= right; ) {
                double ak = a[k];
                if (ak != 0.0d) {
                    break;
                }
                if (Double.doubleToRawLongBits(ak) < 0) { // ak is -0.0d
                    a[k] = 0.0d;
                    a[++p] = -0.0d;
                }
            }
        }

        /**
         * Sorts the specified range of the array.
         *
         * @param a        the array to be sorted
         * @param left     the index of the first element, inclusive, to be sorted
         * @param right    the index of the last element, inclusive, to be sorted
         * @param work     a workspace array (slice)
         * @param workBase origin of usable space in work array
         * @param workLen  usable size of work array
         */
        private static void doSort(double[] a, int left, int right,
                                   double[] work, int workBase, int workLen) {
            // Use Quicksort on small arrays
            if (right - left < QUICKSORT_THRESHOLD) {
                sort(a, left, right, true);
                return;
            }

        /*
         * Index run[i] is the start of i-th run
         * (ascending or descending sequence).
         */
            int[] run = new int[MAX_RUN_COUNT + 1];
            int count = 0;
            run[0] = left;

            // Check if the array is nearly sorted
            for (int k = left; k < right; run[count] = k) {
                if (a[k] < a[k + 1]) { // ascending
                    while (++k <= right && a[k - 1] <= a[k]) ;
                } else if (a[k] > a[k + 1]) { // descending
                    while (++k <= right && a[k - 1] >= a[k]) ;
                    for (int lo = run[count] - 1, hi = k; ++lo < --hi; ) {
                        double t = a[lo];
                        a[lo] = a[hi];
                        a[hi] = t;
                    }
                } else { // equal
                    for (int m = MAX_RUN_LENGTH; ++k <= right && a[k - 1] == a[k]; ) {
                        if (--m == 0) {
                            sort(a, left, right, true);
                            return;
                        }
                    }
                }

            /*
             * The array is not highly structured,
             * use Quicksort instead of merge sort.
             */
                if (++count == MAX_RUN_COUNT) {
                    sort(a, left, right, true);
                    return;
                }
            }

            // Check special cases
            // Implementation note: variable "right" is increased by 1.
            if (run[count] == right++) { // The last run contains one element
                run[++count] = right;
            } else if (count == 1) { // The array is already sorted
                return;
            }

            // Determine alternation base for merge
            byte odd = 0;
            for (int n = 1; (n <<= 1) < count; odd ^= 1) ;

            // Use or create temporary array b for merging
            double[] b;                 // temp array; alternates with a
            int ao, bo;              // array offsets from 'left'
            int blen = right - left; // space needed for b
            if (work == null || workLen < blen || workBase + blen > work.length) {
                work = new double[blen];
                workBase = 0;
            }
            if (odd == 0) {
                System.arraycopy(a, left, work, workBase, blen);
                b = a;
                bo = 0;
                a = work;
                ao = workBase - left;
            } else {
                b = work;
                ao = 0;
                bo = workBase - left;
            }

            // Merging
            for (int last; count > 1; count = last) {
                for (int k = (last = 0) + 2; k <= count; k += 2) {
                    int hi = run[k], mi = run[k - 1];
                    for (int i = run[k - 2], p = i, q = mi; i < hi; ++i) {
                        if (q >= hi || p < mi && a[p + ao] <= a[q + ao]) {
                            b[i + bo] = a[p++ + ao];
                        } else {
                            b[i + bo] = a[q++ + ao];
                        }
                    }
                    run[++last] = hi;
                }
                if ((count & 1) != 0) {
                    for (int i = right, lo = run[count - 1]; --i >= lo;
                         b[i + bo] = a[i + ao]
                            )
                        ;
                    run[++last] = right;
                }
                double[] t = a;
                a = b;
                b = t;
                int o = ao;
                ao = bo;
                bo = o;
            }
        }

        /**
         * Sorts the specified range of the array by Dual-Pivot Quicksort.
         *
         * @param a        the array to be sorted
         * @param left     the index of the first element, inclusive, to be sorted
         * @param right    the index of the last element, inclusive, to be sorted
         * @param leftmost indicates if this part is the leftmost in the range
         */
        private static void sort(double[] a, int left, int right, boolean leftmost) {
            int length = right - left + 1;

            // Use insertion sort on tiny arrays
            if (length < INSERTION_SORT_THRESHOLD) {
                if (leftmost) {
                /*
                 * Traditional (without sentinel) insertion sort,
                 * optimized for server VM, is used in case of
                 * the leftmost part.
                 */
                    for (int i = left, j = i; i < right; j = ++i) {
                        double ai = a[i + 1];
                        while (ai < a[j]) {
                            a[j + 1] = a[j];
                            if (j-- == left) {
                                break;
                            }
                        }
                        a[j + 1] = ai;
                    }
                } else {
                /*
                 * Skip the longest ascending sequence.
                 */
                    do {
                        if (left >= right) {
                            return;
                        }
                    } while (a[++left] >= a[left - 1]);

                /*
                 * Every element from adjoining part plays the role
                 * of sentinel, therefore this allows us to avoid the
                 * left range check on each iteration. Moreover, we use
                 * the more optimized algorithm, so called pair insertion
                 * sort, which is faster (in the context of Quicksort)
                 * than traditional implementation of insertion sort.
                 */
                    for (int k = left; ++left <= right; k = ++left) {
                        double a1 = a[k], a2 = a[left];

                        if (a1 < a2) {
                            a2 = a1;
                            a1 = a[left];
                        }
                        while (a1 < a[--k]) {
                            a[k + 2] = a[k];
                        }
                        a[++k + 1] = a1;

                        while (a2 < a[--k]) {
                            a[k + 1] = a[k];
                        }
                        a[k + 1] = a2;
                    }
                    double last = a[right];

                    while (last < a[--right]) {
                        a[right + 1] = a[right];
                    }
                    a[right + 1] = last;
                }
                return;
            }

            // Inexpensive approximation of length / 7
            int seventh = (length >> 3) + (length >> 6) + 1;

        /*
         * Sort five evenly spaced elements around (and including) the
         * center element in the range. These elements will be used for
         * pivot selection as described below. The choice for spacing
         * these elements was empirically determined to work well on
         * a wide variety of inputs.
         */
            int e3 = (left + right) >>> 1; // The midpoint
            int e2 = e3 - seventh;
            int e1 = e2 - seventh;
            int e4 = e3 + seventh;
            int e5 = e4 + seventh;

            // Sort these elements using insertion sort
            if (a[e2] < a[e1]) {
                double t = a[e2];
                a[e2] = a[e1];
                a[e1] = t;
            }

            if (a[e3] < a[e2]) {
                double t = a[e3];
                a[e3] = a[e2];
                a[e2] = t;
                if (t < a[e1]) {
                    a[e2] = a[e1];
                    a[e1] = t;
                }
            }
            if (a[e4] < a[e3]) {
                double t = a[e4];
                a[e4] = a[e3];
                a[e3] = t;
                if (t < a[e2]) {
                    a[e3] = a[e2];
                    a[e2] = t;
                    if (t < a[e1]) {
                        a[e2] = a[e1];
                        a[e1] = t;
                    }
                }
            }
            if (a[e5] < a[e4]) {
                double t = a[e5];
                a[e5] = a[e4];
                a[e4] = t;
                if (t < a[e3]) {
                    a[e4] = a[e3];
                    a[e3] = t;
                    if (t < a[e2]) {
                        a[e3] = a[e2];
                        a[e2] = t;
                        if (t < a[e1]) {
                            a[e2] = a[e1];
                            a[e1] = t;
                        }
                    }
                }
            }

            // Pointers
            int less = left;  // The index of the first element of center part
            int great = right; // The index before the first element of right part

            if (a[e1] != a[e2] && a[e2] != a[e3] && a[e3] != a[e4] && a[e4] != a[e5]) {
            /*
             * Use the second and fourth of the five sorted elements as pivots.
             * These values are inexpensive approximations of the first and
             * second terciles of the array. Note that pivot1 <= pivot2.
             */
                double pivot1 = a[e2];
                double pivot2 = a[e4];

            /*
             * The first and the last elements to be sorted are moved to the
             * locations formerly occupied by the pivots. When partitioning
             * is complete, the pivots are swapped back into their final
             * positions, and excluded from subsequent sorting.
             */
                a[e2] = a[left];
                a[e4] = a[right];

            /*
             * Skip elements, which are less or greater than pivot values.
             */
                while (a[++less] < pivot1) ;
                while (a[--great] > pivot2) ;

            /*
             * Partitioning:
             *
             *   left part           center part                   right part
             * +--------------------------------------------------------------+
             * |  < pivot1  |  pivot1 <= && <= pivot2  |    ?    |  > pivot2  |
             * +--------------------------------------------------------------+
             *               ^                          ^       ^
             *               |                          |       |
             *              less                        k     great
             *
             * Invariants:
             *
             *              all in (left, less)   < pivot1
             *    pivot1 <= all in [less, k)     <= pivot2
             *              all in (great, right) > pivot2
             *
             * Pointer k is the first index of ?-part.
             */
                outer:
                for (int k = less - 1; ++k <= great; ) {
                    double ak = a[k];
                    if (ak < pivot1) { // Move a[k] to left part
                        a[k] = a[less];
                    /*
                     * Here and below we use "a[i] = b; i++;" instead
                     * of "a[i++] = b;" due to performance issue.
                     */
                        a[less] = ak;
                        ++less;
                    } else if (ak > pivot2) { // Move a[k] to right part
                        while (a[great] > pivot2) {
                            if (great-- == k) {
                                break outer;
                            }
                        }
                        if (a[great] < pivot1) { // a[great] <= pivot2
                            a[k] = a[less];
                            a[less] = a[great];
                            ++less;
                        } else { // pivot1 <= a[great] <= pivot2
                            a[k] = a[great];
                        }
                    /*
                     * Here and below we use "a[i] = b; i--;" instead
                     * of "a[i--] = b;" due to performance issue.
                     */
                        a[great] = ak;
                        --great;
                    }
                }

                // Swap pivots into their final positions
                a[left] = a[less - 1];
                a[less - 1] = pivot1;
                a[right] = a[great + 1];
                a[great + 1] = pivot2;

                // Sort left and right parts recursively, excluding known pivots
                sort(a, left, less - 2, leftmost);
                sort(a, great + 2, right, false);

            /*
             * If center part is too large (comprises > 4/7 of the array),
             * swap internal pivot values to ends.
             */
                if (less < e1 && e5 < great) {
                /*
                 * Skip elements, which are equal to pivot values.
                 */
                    while (a[less] == pivot1) {
                        ++less;
                    }

                    while (a[great] == pivot2) {
                        --great;
                    }

                /*
                 * Partitioning:
                 *
                 *   left part         center part                  right part
                 * +----------------------------------------------------------+
                 * | == pivot1 |  pivot1 < && < pivot2  |    ?    | == pivot2 |
                 * +----------------------------------------------------------+
                 *              ^                        ^       ^
                 *              |                        |       |
                 *             less                      k     great
                 *
                 * Invariants:
                 *
                 *              all in (*,  less) == pivot1
                 *     pivot1 < all in [less,  k)  < pivot2
                 *              all in (great, *) == pivot2
                 *
                 * Pointer k is the first index of ?-part.
                 */
                    outer:
                    for (int k = less - 1; ++k <= great; ) {
                        double ak = a[k];
                        if (ak == pivot1) { // Move a[k] to left part
                            a[k] = a[less];
                            a[less] = ak;
                            ++less;
                        } else if (ak == pivot2) { // Move a[k] to right part
                            while (a[great] == pivot2) {
                                if (great-- == k) {
                                    break outer;
                                }
                            }
                            if (a[great] == pivot1) { // a[great] < pivot2
                                a[k] = a[less];
                            /*
                             * Even though a[great] equals to pivot1, the
                             * assignment a[less] = pivot1 may be incorrect,
                             * if a[great] and pivot1 are floating-point zeros
                             * of different signs. Therefore in float and
                             * double sorting methods we have to use more
                             * accurate assignment a[less] = a[great].
                             */
                                a[less] = a[great];
                                ++less;
                            } else { // pivot1 < a[great] < pivot2
                                a[k] = a[great];
                            }
                            a[great] = ak;
                            --great;
                        }
                    }
                }

                // Sort center part recursively
                sort(a, less, great, false);

            } else { // Partitioning with one pivot
            /*
             * Use the third of the five sorted elements as pivot.
             * This value is inexpensive approximation of the median.
             */
                double pivot = a[e3];

            /*
             * Partitioning degenerates to the traditional 3-way
             * (or "Dutch National Flag") schema:
             *
             *   left part    center part              right part
             * +-------------------------------------------------+
             * |  < pivot  |   == pivot   |     ?    |  > pivot  |
             * +-------------------------------------------------+
             *              ^              ^        ^
             *              |              |        |
             *             less            k      great
             *
             * Invariants:
             *
             *   all in (left, less)   < pivot
             *   all in [less, k)     == pivot
             *   all in (great, right) > pivot
             *
             * Pointer k is the first index of ?-part.
             */
                for (int k = less; k <= great; ++k) {
                    if (a[k] == pivot) {
                        continue;
                    }
                    double ak = a[k];
                    if (ak < pivot) { // Move a[k] to left part
                        a[k] = a[less];
                        a[less] = ak;
                        ++less;
                    } else { // a[k] > pivot - Move a[k] to right part
                        while (a[great] > pivot) {
                            --great;
                        }
                        if (a[great] < pivot) { // a[great] <= pivot
                            a[k] = a[less];
                            a[less] = a[great];
                            ++less;
                        } else { // a[great] == pivot
                        /*
                         * Even though a[great] equals to pivot, the
                         * assignment a[k] = pivot may be incorrect,
                         * if a[great] and pivot are floating-point
                         * zeros of different signs. Therefore in float
                         * and double sorting methods we have to use
                         * more accurate assignment a[k] = a[great].
                         */
                            a[k] = a[great];
                        }
                        a[great] = ak;
                        --great;
                    }
                }

            /*
             * Sort left and right parts recursively.
             * All elements from center part are equal
             * and, therefore, already sorted.
             */
                sort(a, left, less - 1, leftmost);
                sort(a, great + 1, right, false);
            }
        }
    }

```
