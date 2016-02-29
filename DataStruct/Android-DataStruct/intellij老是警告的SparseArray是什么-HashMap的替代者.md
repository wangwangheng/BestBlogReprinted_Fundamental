# intellij老是警告的SparseArray是什么 - HashMap的替代者

来源:[blog.zhaiyifan.cn](http://blog.zhaiyifan.cn/2015/08/28/intellij%E8%80%81%E6%98%AF%E8%AD%A6%E5%91%8A%E7%9A%84SparseArray%E6%98%AF%E4%BB%80%E4%B9%88-HashMap%E7%9A%84%E6%9B%BF%E4%BB%A3%E8%80%85/)

如果只想看比较和结论，可以直接跳到最后

## 序言
身为一个有代码洁癖的程序员，在写Android应用的时候，我总是会去注意

* 代码规范（Google Android Guideline）
* 能一行搞定的代码，绝不写两行
* 决不让编译器（intellij, as）右边滚动条有黄色
* 不重复自己

当然了，实际开发中，编译器报的warning有些不太好避免，比如有些空指针，编译器从android源码来看，觉得不会出现空指针，但是实际情况下….你懂得，部分rom手贱改坏了源码，结果就crash了，所以我们能做的，就是尽量减少warning。

扯了这么多，说回主题，有时候在HashMap申明那行，intellij会报warning，说用SparseArray更好，那么SparseArray究竟是什么东西，为什么更好，为什么提示说更省内存呢？

**本文以api 21的源码为准**

## SparseArray源码分析

```
// E对应HashMap的Value
public class SparseArray<E> implements Cloneable {
    // 用来优化删除性能，标记已经删除的对象
    private static final Object DELETED = new Object();
    // 用来优化删除性能，标记是否需要垃圾回收
    private boolean mGarbage = false;

    // 存储索引，整数索引从小到大被映射在该数组
    private int[] mKeys;
    // 存储对象
    private Object[] mValues;
    // 实际大小
    private int mSize;
```

构造函数：

```
public SparseArray(int initialCapacity) {
    if (initialCapacity == 0) {
     // EmptyArray是一个不需要数组分配的轻量级表示。
        mKeys = EmptyArray.INT;
        mValues = EmptyArray.OBJECT;
    } else {
        mValues = ArrayUtils.newUnpaddedObjectArray(initialCapacity);
        mKeys = new int[mValues.length];
    }
    mSize = 0;
}
```

这里也充分节约了内存。`newUnpaddedObjectArray`最后指向了VMRuntime的一个native方法

```
/**
 * 返回一个至少长minLength的数组，但可能更大。增长的大小来自于避免数组后的任何padding。padding的大小依赖于componentType和内存分配器的实现
 */
public native Object newUnpaddedArray(Class<?> componentType, int minLength);
```

Get方法使用了二分查找

```
/**
 * 获得指定key的映射对象，或者null如果没有该映射。
 */
public E get(int key) {
    return get(key, null);
}

@SuppressWarnings("unchecked")
public E get(int key, E valueIfKeyNotFound) {
    // 二分查找
    int i = ContainerHelpers.binarySearch(mKeys, mSize, key);
	// 如果没找到或者该value已经被标记删除
    if (i < 0 || mValues[i] == DELETED) {
        return valueIfKeyNotFound;
    } else {
        return (E) mValues[i];
    }
}
```

对应的二分查找实现这里就不赘述了，大致就是一个对有序数组一分为二比较中间值和目标值的循环查找。

其中比较巧妙的一点是在没有找到的时候会返回一个~low的index，其有两个作用：

* 告诉调用者没有找到
* 调用者可以直接用~result获得该元素应该插入的位置(-(insertion point) - 1)。

对应的put方法

```
/**
 * 添加一个指定key到指定object的映射，如果之前有一个指定key的映射则直接替换掉原映射object。
 */
public void put(int key, E value) {
    int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

    if (i >= 0) {
        // 该key原来有了，替换掉
        mValues[i] = value;
    } else {
        // 做一个负运算，获得应该插入的index
        i = ~i;

		// size足够且原value已经被标记为删除
        if (i < mSize && mValues[i] == DELETED) {
            mKeys[i] = key;
            mValues[i] = value;
            return;
        }
        // 走到这里就说么i超出了size，或者对应元素是有效的

		// 被标记为需要垃圾回收且SparseArray大小不小于keys数组长度
        if (mGarbage && mSize >= mKeys.length) {
            // 压缩空间（这里源码有点逗，竟然还有Log.e的注释留在那里，看来Android源码工程师也是要调试的），会压缩数组，把无效的值都去掉，保证连续有效值
            gc();

            // 再次查找因为索引可能改变
            i = ~ContainerHelpers.binarySearch(mKeys, mSize, key);
        }

		// 插入，如果size不够则会重新分配更大的数组，然后拷贝过去并插入；size足够则用System.arraycopy把插入位置开始的value都后移然后插入
        mKeys = GrowingArrayUtils.insert(mKeys, mSize, i, key);
        mValues = GrowingArrayUtils.insert(mValues, mSize, i, value);
        // 实际大小加1
        mSize++;
    }
}
```

在看看remove(del)方法

```
/**
 * 如果有的话，删除对应key的映射
 */
public void delete(int key) {
    // 又是二分查找
    int i = ContainerHelpers.binarySearch(mKeys, mSize, key);
	// 存在则标记对应value为DELETED，且置位mGarbage
    if (i >= 0) {
        if (mValues[i] != DELETED) {
            mValues[i] = DELETED;
            mGarbage = true;
        }
    }
}

/**
 * {@link #delete(int)}的别名.
 */
public void remove(int key) {
    delete(key);
}

/**
 * 删除指定索引的映射（这个有点暴力啊，用的应该比较少吧，直接指定位置了）
 */
public void removeAt(int index) {
	// 就是delete里面那段方法，如此说来为什么delete不调用removeAt而要重复这段代码呢
    if (mValues[index] != DELETED) {
        mValues[index] = DELETED;
        mGarbage = true;
    }
}
```

大致看了crud的这几个方法，应该有个一个初步了解了吧，SparseArray因为key是一个纯粹的整数数组，避免了auto-boxing key和额外的数据结构去映射K/V关系，从而节省了内存。

当然了，这里也有一个tradeoff(权衡)，由于key数组需要有序，所以每次都会相对简单的写操作更花费时间，要去二分查找，要在数组删除/插入元素。所以对应地为了优化，因为了mGarbage和DELETED，将可能的多次gc合并为一次，延迟到必须的时候执行。

源码注释里也提到了，该类不适合用于大数据量，上百个entry的时候差别在50%以内，尚可以接受，毕竟在移动端我们缺的，往往是内存，而不是CPU。

## HashMap源码分析
相较SparseArray，HashMap的实现就复杂一点了，因为其支持多种key（甚至null），还要实现Iterator。这里我们主要看看基本操作实现。

```
/**
 * transient关键字表示不需要序列化。
 * Hash表，null key的在下面。
 * HashMapEntry定义了K/V映射，hash值，以及next元素
 */
transient HashMapEntry<K, V>[] table;

/**
 * 该entry代表null key, 或者不存在该映射.
 */
transient HashMapEntry<K, V> entryForNullKey;

/**
 * hash map的映射数量.
 */
transient int size;

/**
 * 结构修改的时候自增，来做（最大努力的）并发修改检测
 */
transient int modCount;

/**
 * hash表会在大小超过该阙值的时候做rehash。通常该值为0.75 * capacity, 除非容量为0，即上面的EMPTY_TABLE声明。
 */
private transient int threshold;

public HashMap(int capacity) {
    if (capacity < 0) {
        throw new IllegalArgumentException("Capacity: " + capacity);
    }

    if (capacity == 0) {
	    // 和SparseArray类似，所有空的实例共享同一个EMPTY表示
        HashMapEntry<K, V>[] tab = (HashMapEntry<K, V>[]) EMPTY_TABLE;
        table = tab;
        // 强制先put()来替换EMPTY_TABLE
        threshold = -1;
        return;
    }

    if (capacity < MINIMUM_CAPACITY) {
        capacity = MINIMUM_CAPACITY;
    } else if (capacity > MAXIMUM_CAPACITY) {
        capacity = MAXIMUM_CAPACITY;
    } else {
        // 难道是为了内存padding
        capacity = Collections.roundUpToPowerOfTwo(capacity);
    }
    makeTable(capacity);
}

/**
 * 根据给定容量分配一个hash表，并设置对应阙值。
 * @param newCapacity 必须是2的次数，因为要内存对齐
 */
private HashMapEntry<K, V>[] makeTable(int newCapacity) {
// 这么做难道是为了同步性考虑
HashMapEntry<K, V>[] newTable
            = (HashMapEntry<K, V>[]) new HashMapEntry[newCapacity];
    table = newTable;
    threshold = (newCapacity >> 1) + (newCapacity >> 2); // 3/4 capacity
    return newTable;
}
```

然后是插入的代码

```
/**
 * 映射指定key到指定value，如果有原对应mapping返回其value，否则返回null。
 */
@Override public V put(K key, V value) {
    if (key == null) {
	    // key为空直接去putValueForNullKey方法
        return putValueForNullKey(value);
    }
	// 计算该key的hash值，根据key本身的hashCode做二次hash
    int hash = Collections.secondaryHash(key);
    HashMapEntry<K, V>[] tab = table;
    int index = hash & (tab.length - 1);
    for (HashMapEntry<K, V> e = tab[index]; e != null; e = e.next) {
	    // 原来有对应entry了，直接修改value，然后返回oldValue
        if (e.hash == hash && key.equals(e.key)) {
            preModify(e);
            V oldValue = e.value;
            e.value = value;
            return oldValue;
        }
    }

    // 没有现存的entry，创建一个
    modCount++;
    // size超出阙值，double容量，重新计算index
    if (size++ > threshold) {
        tab = doubleCapacity();
        index = hash & (tab.length - 1);
    }
    // 在table的index位置插入一个新的HashMapEntry，其next是自己
    addNewEntry(key, value, hash, index);
    return null;
}

private V putValueForNullKey(V value) {
    HashMapEntry<K, V> entry = entryForNullKey;
    // 和上面类似的逻辑，如果存在则替换，否则创建该HashMapEntry
    if (entry == null) {
        addNewEntryForNullKey(value);
        size++;
        modCount++;
        return null;
    } else {
        preModify(entry);
        V oldValue = entry.value;
        entry.value = value;
        return oldValue;
    }
}
```

姑且说到这里，大致可以清楚，HashMap是一个以hash为中心的实现，在size上，也只有double的逻辑，而没有remove后是否缩小capacity的逻辑。时间复杂度O(1)的代价就是耗费大量内存来存储数据。

## 比较
* HashMap -> 快，浪费内存
* SparseArray -> 会有性能损耗, 节约内存

我们做一个简单的性能测试，不带参数初始化HashMap和SparseArray，同样存储Integer->String的映射

用:符号来分割HashMap和SparseArray的耗时（毫秒）

|实验|第一次|第二次|第三次|第四次|
|:--:|:--:|:--:|:--:|:--:|
|依次put|10000个数|13:7|12:6|76:4|14:6|
|随机put|10000个数|16:83|18:85|15:76|17:78|

4次测试结果相近（有一个噪音数据76/4有待研究），HashMap在有序的key时候，更耗时，而在无需key的时候则是SparseArray更耗时，而HashMap则没有太大的性能差别。

再对随机put后的结果做了10000次get后，获得了7:3，7:3，8:3的结果，可见get操作上，SparseArray性能更好，但即便在10000个entry的时候，差别其实也并不大。

**在内存上，看到SparseArray在Allocation Tracker中为32，而HashMap则达到了69632这个可怕的数字。。。。。。**

## 结论

在key为整数的情况下，考虑到移动端往往K/V对不会太大，所以用SparseArray能更节省内存，且性能损耗在可接受的范围。

从试验结果看，SparseArray相较于HashMap，大大节省了内存，对移动端实在是不二的选择。
