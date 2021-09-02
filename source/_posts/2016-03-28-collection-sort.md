---
layout: post
title: "Collections.sort 原理探究"
date: 2016-03-28
tags: Java
categories: Java
---

 从代码的角度一步步深入，首先

```java
List<String> list = new ArrayList<String>();
Collections.sort(list);
```

里面是这样
```java
@SuppressWarnings("unchecked")
public static <T extends Comparable<? super T>> void sort(List<T> list) {
    list.sort(null);
}
```

list 的泛型需要继承 Comparable 接口，否则需要使用 
```java
@SuppressWarnings({"unchecked", "rawtypes"})
public static <T> void sort(List<T> list, Comparator<? super T> c) {
    list.sort(c);
}
```
list.sort 长这样
```java
    @SuppressWarnings({"unchecked", "rawtypes"})
    default void sort(Comparator<? super E> c) {
        Object[] a = this.toArray();
        Arrays.sort(a, (Comparator) c);
        ListIterator<E> i = this.listIterator();
        for (Object e : a) {
            i.next();
            i.set((E) e);
        }
    }
```
1. 转成数组
2. 调用 Arrays.sort 排序
3. 通过迭代器赋值回原来集合

然后关键来了，JDK 默认的的 sort 方法到底有多神奇呢？
```java
    public static <T> void sort(T[] a, Comparator<? super T> c) {
        if (c == null) {
            sort(a);
        } else {
            if (LegacyMergeSort.userRequested)
                legacyMergeSort(a, c);
            else
                TimSort.sort(a, 0, a.length, c, null, 0, 0);
        }
    }
    
    public static void sort(Object[] a) {
        if (LegacyMergeSort.userRequested)
            legacyMergeSort(a);
        else
            ComparableTimSort.sort(a, 0, a.length, null, 0, 0);
    }
```
再继续扒皮...
```java
private static void legacyMergeSort(Object[] a) {
        Object[] aux = a.clone();
        mergeSort(aux, a, 0, a.length, 0);
    }
```
这里用了一个 clone() 方法，mergeSort 看来已经是归并排序了，是不是一会儿还得赋值回来？继续看...
```java
@SuppressWarnings({"unchecked", "rawtypes"})
    private static void mergeSort(Object[] src,
                                  Object[] dest,
                                  int low,
                                  int high,
                                  int off) {
        int length = high - low;

        //这里这个常量是7，长度小于7直接冒泡解决，比较的是 dest 数组
        if (length < INSERTIONSORT_THRESHOLD) {
            for (int i=low; i<high; i++)
                for (int j=i; j>low &&
                         ((Comparable) dest[j-1]).compareTo(dest[j])>0; j--)
                    swap(dest, j, j-1);
            return;
        }

        // Recursively sort halves of dest into src
        int destLow  = low;
        int destHigh = high;
        //从上面看下来off 是0，这里可以忽略
        low  += off;
        high += off;
        //新技能 Get
        int mid = (low + high) >>> 1;
        mergeSort(dest, src, low, mid, -off);
        mergeSort(dest, src, mid, high, -off);

        // 如果列表已经有序, 只需要从 src 拷贝到 dest. 下面是优化过的拷贝算法
        if (((Comparable)src[mid-1]).compareTo(src[mid]) <= 0) {
            System.arraycopy(src, low, dest, destLow, length);
            return;
        }

        // 归并算法的最后，拷贝到 dest
        for(int i = destLow, p = low, q = mid; i < destHigh; i++) {
            if (q >= high || p < mid && ((Comparable)src[p]).compareTo(src[q])<=0)
                dest[i] = src[p++];
            else
                dest[i] = src[q++];
        }
    }
```
等等，好像什么地方漏了。。。。  
```java
TimSort.sort(a, 0, a.length, c, null, 0, 0);
```
看方法上的注释说这是自1.8开始后才使用的，尽可能使用给定的数据空间，为了提升性能而设计
```java
    static <T> void sort(T[] a, int lo, int hi, Comparator<? super T> c,
                         T[] work, int workBase, int workLen) {
        assert c != null && a != null && lo >= 0 && lo <= hi && hi <= a.length;
        
        int nRemaining  = hi - lo;
        if (nRemaining < 2)
            return;  // Arrays of size 0 and 1 are always sorted

        // 如果数组长度小(这里是32), 使用0拷贝的"mini-TimSort"方式
        if (nRemaining < MIN_MERGE) {
            int initRunLen = countRunAndMakeAscending(a, lo, hi, c);
            binarySort(a, lo, hi, lo + initRunLen, c);
            return;
        }

        /**
         * March over the array once, left to right, finding natural runs,
         * extending short natural runs to minRun elements, and merging runs
         * to maintain stack invariant.
         */
        TimSort<T> ts = new TimSort<>(a, c, work, workBase, workLen);
        int minRun = minRunLength(nRemaining);
        do {
            // Identify next run
            int runLen = countRunAndMakeAscending(a, lo, hi, c);

            // If run is short, extend to min(minRun, nRemaining)
            if (runLen < minRun) {
                int force = nRemaining <= minRun ? nRemaining : minRun;
                binarySort(a, lo, lo + force, lo + runLen, c);
                runLen = force;
            }

            // Push run onto pending-run stack, and maybe merge
            ts.pushRun(lo, runLen);
            ts.mergeCollapse();

            // Advance to find next run
            lo += runLen;
            nRemaining -= runLen;
        } while (nRemaining != 0);

        // Merge all remaining runs to complete sort
        assert lo == hi;
        ts.mergeForceCollapse();
        assert ts.stackSize == 1;
    }
```
擦，还得继续扒皮 binarySort
```java
@SuppressWarnings("fallthrough")
    private static <T> void binarySort(T[] a, int lo, int hi, int start,
                                       Comparator<? super T> c) {
        assert lo <= start && start <= hi;
        if (start == lo)
            start++;
        for ( ; start < hi; start++) {
            T pivot = a[start];

            // Set left (and right) to the index where a[start] (pivot) belongs
            int left = lo;
            int right = start;
            assert left <= right;
            /*
             * Invariants:
             *   pivot >= all in [lo, left).
             *   pivot <  all in [right, start).
             */
            while (left < right) {
                int mid = (left + right) >>> 1;
                if (c.compare(pivot, a[mid]) < 0)
                    right = mid;
                else
                    left = mid + 1;
            }
            assert left == right;

            /*
             * The invariants still hold: pivot >= all in [lo, left) and
             * pivot < all in [left, start), so pivot belongs at left.  Note
             * that if there are elements equal to pivot, left points to the
             * first slot after them -- that's why this sort is stable.
             * Slide elements over to make room for pivot.
             */
            int n = start - left;  // The number of elements to move
            // Switch is just an optimization for arraycopy in default case
            switch (n) {
                case 2:  a[left + 2] = a[left + 1];
                case 1:  a[left + 1] = a[left];
                         break;
                default: System.arraycopy(a, left, a, left + 1, n);
            }
            a[left] = pivot;
        }
    }
```
这是一个二分的插入排序算法。然后研究了下下面这个函数，返回连续升序数据的数据量，如果前 n 个数据是连续降序排列的，翻转此序列，并返回 n 。为了防止最坏的情况
```java
private static <T> int countRunAndMakeAscending(T[] a, int lo, int hi,
                                                    Comparator<? super T> c) {
	assert lo < hi;
	int runHi = lo + 1;
	if (runHi == hi)
		return 1;

	// Find end of run, and reverse range if descending
	if (c.compare(a[runHi++], a[lo]) < 0) { // Descending
	while (runHi < hi && c.compare(a[runHi], a[runHi - 1]) < 0)
		runHi++;
		reverseRange(a, lo, runHi);
	} else {                              // Ascending
		while (runHi < hi && c.compare(a[runHi], a[runHi - 1]) >= 0) {
			runHi++;
		}
	}
	return runHi - lo;
}
```
TimeSort的逻辑还是挺复杂，这里引用一个 demo 的解释

http://blog.sina.com.cn/s/blog_8e6f1b330101h7fa.html

>5. Demo
>这一节用一个具体的例子来演示整个算法的演进过程：
>*注意*：为了演示方便，我将TimSort中的minRun直接设置为2，否则我不能用很小的数组演示。。。同时把MIN_MERGE也改成2（默认为32），这样避免直接进入binary sort。
>
>初始数组为[7,5,1,2,6,8,10,12,4,3,9,11,13,15,16,14]
>
>=> 寻找连续的降序或升序序列 (4.3.2)
>
>[1,5,7] [2,6,8,10,12,4,3,9,11,13,15,16,14]
>
>=> 入栈 (4.3.4)   
>当前的栈区块为[3]   
>=> 进入merge循环 (4.3.5)   
>do not merge因为栈大小仅为1   
>=> 寻找连续的降序或升序序列 (4.3.2)   
>[1,5,7] [2,6,8,10,12] [4,3,9,11,13,15,16,14]   
>=> 入栈 (4.3.4)   
>当前的栈区块为[3, 5]   
>=> 进入merge循环 (4.3.5)   
>merge因为runLen[0]<=runLen[1]   
>1) gallopRight：寻找run1的第一个元素应当插入run0中哪个位置（”2”应当插入”1”之后），然后就可以忽略之前run0的>元素（都比run1的第一个元素小）   
>2) gallopLeft：寻找run0的最后一个元素应当插入run1中哪个位置（”7”应当插入”8”之前），然后就可以忽略之后 run1 的元素（都比run0的最后一个元素大）   
>这样需要排序的元素就仅剩下[5,7] [2,6]，然后进行mergeLow   
>完成之后的结果：   
>[1,2,5,6,7,8,10,12] [4,3,9,11,13,15,16,14]   
>=> 入栈 (4.3.4)   
>当前的栈区块为[8]   
>退出当前merge循环因为栈中的区块仅为1   
>=> 寻找连续的降序或升序序列 (4.3.2)   
>[1,2,5,6,7,8,10,12] [3,4] [9,11,13,15,16,14]   
>=> 入栈 (4.3.4)   
>当前的栈区块大小为[8,2]   
>=> 进入merge循环 (4.3.5)   
>do not merge因为runLen[0]>runLen[1]   
>=> 寻找连续的降序或升序序列 (4.3.2)   
>[1,2,5,6,7,8,10,12] [3,4] [9,11,13,15,16] [14]   
>=> 入栈 (4.3.4)   
>当前的栈区块为[8,2,5]   
>=>   
>do not merege run1与run2因为不满足runLen[0]<=runLen[1]+runLen[2]   
>merge run2与run3因为runLen[1]<=runLen[2]   
>1) gallopRight：发现run1和run2就已经排好序   
>完成之后的结果：   
>[1,2,5,6,7,8,10,12] [3,4,9,11,13,15,16] [14]   
>=> 入栈 (4.3.4)   
>当前入栈的区块大小为[8,7]   
>退出merge循环因为runLen[0]>runLen[1]   
>=> 寻找连续的降序或升序序列 (4.3.2)   
>最后只剩下[14]这个元素：[1,2,5,6,7,8,10,12] [3,4,9,11,13,15,16] [14]   
>=> 入栈 (4.3.4)   
>当前入栈的区块大小为[8,7,1]   
>=> 进入merge循环 (4.3.5)   
>merge因为runLen[0]<=runLen[1]+runLen[2]   
>因为runLen[0]>runLen[2]，所以将run1和run2先合并。（否则将run0和run1先合并）   
>1) gallopRight & 2) gallopLeft   
>这样需要排序的元素剩下[13,15] [14]，然后进行mergeHigh   
>完成之后的结果：   
>[1,2,5,6,7,8,10,12] [3,4,9,11,13,14,15,16] 当前入栈的区块为[8,8]   
>=>   
>继续merge因为runLen[0]<=runLen[1]   
>1) gallopRight & 2) gallopLeft   
>需要排序的元素剩下[5,6,7,8,10,12] [3,4,9,11]，然后进行mergeHigh   
>完成之后的结果：   
>[1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16] 当前入栈的区块大小为[16]   
>=>   
>不需要final merge因为当前栈大小为1   
>=>   
>结束   

