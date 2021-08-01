

[AndroidNote](https://github.com/zhpanvip/AndroidNote/wiki)是笔者花费了大量精力整理的Java基础和Android进阶常问面试题集合。最近在整理SparseArray这一知识点的时候，发现网上大多数SparseArray原理分析的文章都存在很多问题（可以说很多作者并没有读懂SparseArray的源码），也正因此，才有了这篇SparseArray原理的文章。我们知道，SparseArray与ArrayMap是Android中高效存储K-V的数据结构，也是是Android面试中的常客，弄懂它们的实现原理是很有必要的，本篇文章就以SparseArray的源码为例进行深入分析。


## 一、SparseArray的类结构

SparseArray可以翻译为**稀疏数组**，从字面上可以理解为松散不连续的数组。虽然叫做Array，但它却是存储K-V的一种数据结构。其中Key只能是int类型，而Value是Object类型。我们来看下它的类结构：

```java
public class SparseArray<E> implements Cloneable {
    // 用来标记此处的值已被删除
    private static final Object DELETED = new Object();
    // 用来标记是否有元素被移除
    private boolean mGarbage = false;
    // 用来存储key的集合
    private int[] mKeys;
    // 用来存储value的集合
    private Object[] mValues;
    // 存入的元素个数
    private int mSize;
    
    // 默认初始容量为10
    public SparseArray() {
        this(10);
    }

    public SparseArray(int initialCapacity) {
        if (initialCapacity == 0) {
            mKeys = EmptyArray.INT;
            mValues = EmptyArray.OBJECT;
        } else {
            mValues = ArrayUtils.newUnpaddedObjectArray(initialCapacity);
            mKeys = new int[mValues.length];
        }
        mSize = 0;
    }
    
    // ...省略其他代码

}
```

可以看到SparseArray仅仅实现了Cloneable接口并没有实现Map接口，并且SparseArray内部维护了一个int数组和一个Object数组。在无参构造方法中调用了有参构造，并将其初始容量设置为了10。

## 二、SparseArray的remove()方法

是不是觉得很奇怪？作为一个容器类，不先讲put方法怎么先将remove呢？这是因为remove方法的一些操作会影响到put的操作。只有先了解了remove才能更容易理解put方法。我们来看remove的代码：

```java

// SparseArray
public void remove(int key) {
    delete(key);
}

public void delete(int key) {
    int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

    if (i >= 0) {
        if (mValues[i] != DELETED) {
            mValues[i] = DELETED;
            mGarbage = true;
        }
    }
}
```
可以看到remove方法直接调用了delete方法。而在delete方法中会先通过二分查找（二分查找代码后边分析）找到key所在的位置，然后将这一位置的value值置为DELETE，注意，这里还将mGarbage设置为了true来标记集合中存在删除元素的情况。想象一下，在删除多个元素后这个集合中是不是就可能会出现不连续的情况？大概这也是SparseArray名字的由来吧。


## 三、SparseArray的put()方法

作为一个存储K-V类型的数据结构，put方法是key和value的入口。也是SparseArray中最重要的一个方法。先来看下put方法的代码：

```java
// SparseArray
public void put(int key, E value) {
    int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

    if (i >= 0) { // 意味着之前mKeys中已经有对应的key存在了,第i个位置对应的就是key。
        mValues[i] = value; // 直接更新value
    } else { // 返回负数说明未在mKeys中查找到key
      
        // 取反得到待插入key的位置
        i = ~i;
      
	// 如果插入位置小于size,并且这个位置的value刚好是被删除掉的，那么直接将key和value分别插入mKeys和mValues的第i个位置
        if (i < mSize && mValues[i] == DELETED) {
            mKeys[i] = key;
            mValues[i] = value;
            return;
        }
	// mGarbage为true说明有元素被移除了，此时mKeys已经满了，但是mKeys内部有被标记为DELETE的元素
        if (mGarbage && mSize >= mKeys.length) {
            // 调用gc方法移动mKeys和mValues中的元素，这个方法可以后边分析
            gc();
					
            // 由于gc方法移动了数组，因此插入位置可能有变化，所以需要重新计算插入位置
            i = ~ContainerHelpers.binarySearch(mKeys, mSize, key);
        } 
	// GrowingArrayUtils的insert方法将会将插入位置之后的所有数据向后移动一位，然后将key和value分别插入到mKeys和mValue对应的第i个位置，如果数组空间不足还会开启扩容，后边分析这个insert方法
        mKeys = GrowingArrayUtils.insert(mKeys, mSize, i, key);
        mValues = GrowingArrayUtils.insert(mValues, mSize, i, value);
        mSize++;
    }
}
```
虽然这个方法只有寥寥数行，但是想要完全理解却并非易事，即使写了很详细的注释也不容易读懂。我们不妨来详细分析一下。第一行代码通过二分查找得到了一个index。看下二分查找的代码：


```java
// ContainerHelpers
static int binarySearch(int[] array, int size, int value) {
    int lo = 0;
    int hi = size - 1;

    while (lo <= hi) {
        final int mid = (lo + hi) >>> 1;
        final int midVal = array[mid];

        if (midVal < value) {
            lo = mid + 1;
        } else if (midVal > value) {
            hi = mid - 1;
        } else {
            return mid;  // value found
        }
    }
    return ~lo;  // value not present
}
```
关于二分查找相信大家都是比较熟悉的，这一算法用于在一组有序数组中查找某一元素所在位置的。如果数组中存在这一元素，则将这个元素对应的位置返回。如果不存在那么此时的lo就是这个元素的最佳存储位置。上述代码中将lo取反作为了返回值。因为lo一定是大于等于0的数，因此取反后的返回值必定小于等于0.明白了这一点，再来看put方法中的这个if...else是不是很容易理解了？

```
// SparseArray
public void put(int key, E value) {
 
    if (i >= 0) { 
        mValues[i] = value; // 直接更新value
    } else { 
        i = ~i;
        // ... 省略其它代码
    }
}
```
如果i>=0,意味着当前的这个key已经存在于mKeys中了，那么此时put只需要将最新的value更新到mValues中即可。而如果i<=0就意味着mKeys中之前没有对应的key。因此就需要将key和value分别插入到mKeys和mValues中。而插入的最佳位置就是对i取反。

得到插入位置之后，如果这个位置是被标记为删除的元素，那么久可以直接将其覆盖掉了，因此有以下代码：
```java
public void put(int key, E value) {
    // ...
    if (i >= 0) {
        // ...
    } else {    
        // 如果i对应的位置是被删除掉的，可以直接将其覆盖
        if (i < mSize && mValues[i] == DELETED) {
            mKeys[i] = key;
            mValues[i] = value;
            return;
        }
        // ...
    }
    
}
```
如果上边条件不满足，那么继续往下看：

```java
public void put(int key, E value) {
    // ...
    if (i >= 0) {
        // ...
    } else {    
	// mGarbage为true说明有元素被移除了，此时mKeys已经满了，但是mKeys内部有被标记为DELETE的元素
        if (mGarbage && mSize >= mKeys.length) {
            // 调用gc方法移动mKeys和mValues中的元素，这个方法可以后边分析
            gc();
					
            // 由于gc方法移动了数组，因此插入位置可能有变化，所以需要重新计算插入位置
            i = ~ContainerHelpers.binarySearch(mKeys, mSize, key);
        } 
        // ...
    }
    
}
```
上边我们已经知道，在remove元素的时候mGarbage会被置为true，这段代码意味着有被移除的元素，被移除的位置并不是要插入的位置，并且如果mKeys已经满了，那么就调用gc方法来移动元素填充被移除的位置。由于mKeys中元素位置发生了变化，因此key插入的位置也可能改变，因此需要再次调用二分法来查找key的插入位置。




以上代码最终会确定key被插入的位置，接下来调用GrowingArrayUtils的insert方法来进行key的插入操作：

```java
// SparseArray
public void put(int key, E value) {
    int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

    if (i >= 0) { 
        // ...
    } else { 
         // ...

	// GrowingArrayUtils的insert方法将会将插入位置之后的所有数据向后移动一位，然后将key和value分别插入到mKeys和mValue对应的第i个位置，如果数组空间不足还会开启扩容，后边分析这个insert方法
        mKeys = GrowingArrayUtils.insert(mKeys, mSize, i, key);
        mValues = GrowingArrayUtils.insert(mValues, mSize, i, value);
        mSize++;
    }
}
```

GrowingArrayUtils的insert方法代码如下：

```java
// GrowingArrayUtils
public static <T> T[] insert(T[] array, int currentSize, int index, T element) {
    assert currentSize <= array.length;
    // 如果插入后数组size小于数组长度，能进行插入操作
    if (currentSize + 1 <= array.length) {
        // 将index之后的所有元素向后移动一位
        System.arraycopy(array, index, array, index + 1, currentSize - index);
        // 将key插入到index的位置
        array[index] = element;
        return array;
    }

    // 来到这里说明数组已满，需需要进行扩容操作。newArray即为扩容后的数组
    T[] newArray = ArrayUtils.newUnpaddedArray((Class<T>)array.getClass().getComponentType(),
            growSize(currentSize));
    System.arraycopy(array, 0, newArray, 0, index);
    newArray[index] = element;
    System.arraycopy(array, index, newArray, index + 1, array.length - index);
    return newArray;
}

// 返回扩容后的size
public static int growSize(int currentSize) {
    return currentSize <= 4 ? 8 : currentSize * 2;
}
```
insert方法的代码比较容易理解，如果数组容量足够，那么就将index之后的元素向后移动一位，然后将key插入index的位置。如果数组容量不足，那么则需要进行扩容，然后再进行插入操作。


### 四、SparseArray的gc()方法

这个方法其实很容易理解，我们知道Java虚拟机在内存不足时会进行GC操作，标记清除法在回收垃圾对象后为了避免内存碎片化，会将存活的对象向内存的一端移动。而SparseArray中的这个gc方法其实就是借鉴了垃圾收集整理碎片空间的思想。



关于mGarbage这个参数上边已经有提到过了，这个变量会在删除元素的时候被置为true。如下：

```Java
// SparseArray中所有移除元素的方法中都将mGarbage置为true

public E removeReturnOld(int key) {
    int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

    if (i >= 0) {
        if (mValues[i] != DELETED) {
            final E old = (E) mValues[i];
            mValues[i] = DELETED;
            mGarbage = true;
            return old;
        }
    }
    return null;
}

public void delete(int key) {
    int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

    if (i >= 0) {
        if (mValues[i] != DELETED) {
            mValues[i] = DELETED;
            mGarbage = true;
        }
    }
}


public void removeAt(int index) {
    if (index >= mSize && UtilConfig.sThrowExceptionForUpperArrayOutOfBounds) {
        throw new ArrayIndexOutOfBoundsException(index);
    }
    if (mValues[index] != DELETED) {
        mValues[index] = DELETED;
        mGarbage = true;
    }
}


```


而SparseArray中所有插入和查找元素的方法中都会判断如果mGarbage为true，并且mSize >= mKeys.length时调用gc,以append方法为例，代码如下：

```java
public void append(int key, E value) {

    if (mGarbage && mSize >= mKeys.length) {
        gc();
    }
  
   // ... 省略无关代码
}
```

源码中调用gc方法的地方多达8处，都是与添加和查找元素相关的方法。例如put()、keyAt()、setValueAt()等方法中。gc的实现其实比较简单，就是将删除位置后的所有数据向前移动一下，代码如下：

```java
private void gc() {
    // Log.e("SparseArray", "gc start with " + mSize);

    int n = mSize;
    int o = 0;
    int[] keys = mKeys;
    Object[] values = mValues;

    for (int i = 0; i < n; i++) {
        Object val = values[i];

        if (val != DELETED) {
            if (i != o) {
                keys[o] = keys[i];
                values[o] = val;
                values[i] = null;
            }

            o++;
        }
    }

    mGarbage = false;
    mSize = o;

    // Log.e("SparseArray", "gc end with " + mSize);
}
```


### 五、SparseArray的get()方法

这个方法就比较简单了，因为put的时候是维持了一个有序数组，因此通过二分查找可以直接确定key在数组中的位置。

```Java
public E get(int key, E valueIfKeyNotFound) {
    int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

    if (i < 0 || mValues[i] == DELETED) {
        return valueIfKeyNotFound;
    } else {
        return (E) mValues[i];
    }
}
```

### 六、总结

可见SparseArray是一个使用起来很简单的数据结构，但是它的原理理解起来似乎却没那么容易。这也是网上大部分文章对应SparseArray的解析都是含糊不清的原因。相信通过本篇文章的学习一定对SparseArray的实现有了新的认识！
