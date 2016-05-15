# HashMap与ConcurrentHashMap旧版源码笔记

来源:[http://www.jianshu.com/p/b2f5033b60a1](http://www.jianshu.com/p/b2f5033b60a1)

## 前言

最近处理了一个内存泄漏的问题, 主要原因在于多线程下对HashMap的操作没有同步, 使用ConcurrentHashMap可以轻松解决问题, 这引起了我对ConcurrentHashMap的兴趣, 想看看它的实现机制, 不过我看的是Android中的早期版本的实现, 最新的JDK中HashMap和ConcurrentHashMap的实现有了显著的变化, 均引入了红黑树, 在链表数量超过阈值时将其转为红黑树存储, ConcurrentHashMap取消锁分段, 改为使用CAS操作, 这两个类的代码量都增加不少. 这篇笔记只针对Android中HashMap和ConcurrentHashMap的早期版本的源码做分析. 注意Android中Java包中的类和JDK中的并非完全相同, 有一些小修改.

## HashMap原理

源码版本: Android 22, 应该是基于JDK 7-b147修改得来.

要想知道ConcurrentHashMap的实现原理, 以及它对多线程的处理, 首先要知道HashMap的原理才好对比, 所以先从HashMap开始.

## 存储形式

HashMap中存储的是键值对`HashMapEntry<K, V>`, 它的存储很简单, 就是把HashMapEntry放进数组table里

```
transient HashMapEntry<K, V>[] table;
```

HashMapEntry有四个成员变量

```
final K key;
V value;
final int hash;
HashMapEntry<K, V> next;
```

名字已经说明了变量的用途, 其中hash用来存储HashMapEntry对应的hash值, 这个hash值决定了HashMapEntry应该放在table的哪个位置上, 如果经过计算, 发现有两个HashMapEntry都需要放在table的同一个位置上, 那么它们将以链表的形式存储, next字段就是用来干这个的.

所以一般来讲HashMap中的存储形式如下:

```
table[0]->null
table[1]->Entry1->null
table[2]->null
table[3]->Entry2->Entry3->null
```

## HashMap创建过程

HashMap提供了四个构造方法:

```
public HashMap()
public HashMap(int capacity)
public HashMap(int capacity, float loadFactor)
public HashMap(Map<? extends K, ? extends V> map)
```

第三个构造方法中的loadFactor在Android上是被忽略掉的, 保留仅仅是为了兼容.

HashMap默认的构造方法很简单

```
private static final int MINIMUM_CAPACITY = 4;

private static final Entry[] EMPTY_TABLE
        = new HashMapEntry[MINIMUM_CAPACITY >>> 1];

public HashMap() {
    table = (HashMapEntry<K, V>[]) EMPTY_TABLE;
    threshold = -1; // Forces first put invocation to replace EMPTY_TABLE
}
```

默认是将table指向一个大小为2的**EMPTY_TABLE**, 这个**EMPTY_TABLE**是被所有的size为0的HashMap所共享的, 大家都指向它. threshold则是阈值, 一般都是table数组大小的3/4, 一旦HashMap中存储的HashMapEntry的数量即将大于threshold + 1, HashMap就会进行扩容操作, 将table数组的大小翻倍, 此处将threshold初始化为-1, 只要一进行put操作, HashMap就会扩容, table就不再指向EMPTY_TABLE. 所以HashMap的EMPTY_TABLE中的元素永远为null.

调用`HashMap(int capacity)`和H`ashMap(int capacity, float loadFactor)`构造, 在Android平台上效果是一样的. 如果capacity传0, 和调用无参构造方法效果相同. 如果capacity大于0时, HashMap并非创建一个大小为capacity的table.

```
private static final int MAXIMUM_CAPACITY = 1 << 30;

public HashMap(int capacity) {
    ......

    if (capacity < MINIMUM_CAPACITY) {
        capacity = MINIMUM_CAPACITY;
    } else if (capacity > MAXIMUM_CAPACITY) {
        capacity = MAXIMUM_CAPACITY;
    } else {
        capacity = Collections.roundUpToPowerOfTwo(capacity);
    }
    makeTable(capacity);
}
```

HashMap会检查capacity的范围, 注意最大值**MAXIMUM_CAPACITY**是`1 << 30`, Java中的int是带符号的32位二进制数表示的, **MAXIMUM_CAPACITY**是int能表示的最大的2的幂, 之所以选这个值, 是因为HashMap要求table的大小必须是2的幂, 主要还是为了计算方便, 下面会提到.

capacity在规定范围中时, 会使用Collections.roundUpToPowerOfTwo获取大于等于capacity的最小的2的幂. 这个方法非常有趣:

```
public static int roundUpToPowerOfTwo(int i) {
    i--; // If input is a power of two, shift its high-order bit right.

    // "Smear" the high-order bit all the way to the right.
    i |= i >>>  1;
    i |= i >>>  2;
    i |= i >>>  4;
    i |= i >>>  8;
    i |= i >>> 16;

    return i + 1;
}
```

乍一看这个方法不明觉厉, 其实也很好理解, 先看那一串移位操作

```
i |= i >>>  1;
i |= i >>>  2;
i |= i >>>  4;
i |= i >>>  8;
i |= i >>> 16;
```

这个方法其实就是把i的最高位1右边所有的位都变成1.

比如我们取一个最简单的例子, 假如i是1000 0000

```
1000 0000执行i |= i >>>  1;变成1100 0000
1100 0000执行i |= i >>>  2;变成1111 0000
1111 0000执行i |= i >>>  4;变成1111 1111
```

对于一个8位的数, 这三步就足够了, Java中的int是32位, 所以一共进行了5次移位的操作.

最后再加上1, 假如传入的i是01XX, 位运算完成后i变成0111, 加上1变成1000, 这样我们得到的是大于i的最小的2的幂, 所以我们在位移运算前先减一, 这样我们得到的是大于i - 1的最小的2的幂, 综合起来看就是大于等于i的最小的2的幂.

这样我们的capacity就转换成了大于等于capacity的最小的2的幂, 这样一个数既满足HashMap对table大小的要求, 又能装得下capacity数量的Entry. 然后调用makeTable真正创建table.

```
private HashMapEntry<K, V>[] makeTable(int newCapacity) {
    @SuppressWarnings("unchecked") HashMapEntry<K, V>[] newTable
            = (HashMapEntry<K, V>[]) new HashMapEntry[newCapacity];
    table = newTable;
    threshold = (newCapacity >> 1) + (newCapacity >> 2); // 3/4 capacity
    return newTable;
}
```

这里对threshold进行了赋值, 右边的位运算其实就是计算`newCapacity * 3/4`, 只是位运算的效率要比直接乘浮点数再取整的效率高, 而且因为*newCapacity*是大于4的一个2的幂, 所以这个位运算实际上是能得到精确值的.

最后一个构造方法就不介绍了, 都是正常逻辑, 需要说明的上面也说清楚了.

## put操作
put方法一开始就先进行判空

```
@Override public V put(K key, V value) {
if (key == null) {
	return putValueForNullKey(value);
}
......
```

HashMap允许null作为key, 这个特殊的HashMapEntry没有放到table里存储, 而是单独用一个成员变量表示

```
transient HashMapEntry<K, V> entryForNullKey;
```

该成员变量为null时, 代表HashMap中没有以null为key的Entry, 当它不为null时, 所指向的HashMapEntry的key是null, hash是0.

对于这个特殊的HashMapEntry不必多做分析, 逻辑上和其他的put操作是一样的, 只是经常要特殊照顾一下, 我们继续看put操作:

```
@Override public V put(K key, V value) {
    ......
    int hash = Collections.secondaryHash(key);
    HashMapEntry<K, V>[] tab = table;
    int index = hash & (tab.length - 1);
    ......
}
```

这里HashMap并没有直接使用key的hash, 而是对key.hashCode()进行了二次hash, 二次hash的核心代码还是一连串位运算.

拿到新的hash后, 通过hash & (tab.length - 1)取得该hash值在table中对应的下标index.

这一句位运算算然看起来怪怪的, 其本质还是取模.

之前说过, table的大小是2的幂, 所以table.length在内存中的形式一定是形如1000....0这样的, 又因为table.length最大值是1 << 30, 所以table的最高位1肯定不是符号位.

举个例子, 假如tab.length == 16, 在内存中就是0001 0000, tab.length - 1在内存中就是0000 1111, hash和这个数做&运算, 只有低4位上的值能保留下来, hash高位上的数全被抹掉了, 低4位上的数范围是0~15, 被抹掉那部分数肯定能被16(0001 0000)整除, 说白了就是取模, hash & (tab.length - 1)实际上就是hash % tab.length, 这也是因为tab.length必须是2的幂, 所以才能靠位运算迅速完成模运算.

我没有找到对这个二次hash算法的权威说明, 如果有知道的同学欢迎在评论区补充. 但这个二次hash是很有必要的. Java中Object的hashcode是内存地址, 这些hashcode的高位基本都是相同的, 某些情况下低位也有可能完全相同, 从刚才的取模运算来看, 如果几个hashcode的低位完全相同, 会导致严重的哈希碰撞, 效果是多个HashMapEntry被分配到table的同一个位置上, 最严重时整个HashMap退化成一个链表, 出于性能的考虑, 我们希望HashMap中的Entry尽量平均分布在table中, 所以HashMap对key的hashcode进行二次hash. 据说这个哈希算法可以做到输入的数只变1位的情况下hash值变化一半以上的位.

继续看put方法

```
@Override public V put(K key, V value) {
    ......
    for (HashMapEntry<K, V> e = tab[index]; e != null; e = e.next) {
        if (e.hash == hash && key.equals(e.key)) {
            preModify(e);
            V oldValue = e.value;
            e.value = value;
            return oldValue;
        }
    }
    ......
}
```

刚才我们通过二次哈希和取模拿到了对应的index, 此时我们要做的就是到table[index]对应的链表中找是否已经存在这个HashMapEntry, 如果已经存在, 把它的value替换成新的就好, 注意这里不仅判断了key的equals方法, 还会比较hash值.

如果链表中还没有存储有相同key的Entry, 那么就新建一个HashMapEntry插入:

```
transient int size;

transient int modCount;

@Override public V put(K key, V value) {
    ......
    // No entry for (non-null) key is present; create one
    modCount++;
    if (size++ > threshold) {
        tab = doubleCapacity();
        index = hash & (tab.length - 1);
    }
    addNewEntry(key, value, hash, index);
	return null;
}
```

这里对modCount加1, 每次发生对HashMap中结构的修改时, 都会将它增加1, 这个变量是HashMap迭代器检测并发修改用的, 但实际上这个检测手段并不能保证检测到并发修改, 后面再说.

size用来记录HashMap中Entry的数量, size在put中也会加1.

如果满足条件, HashMap就会进行扩容, 扩容等会儿再说. 扩容后根据新的table大小重新计算index, 最后调用addNewEntry将Entry插入HashMap.

```
void addNewEntry(K key, V value, int hash, int index) {
	table[index] = new HashMapEntry<K, V>(key, value, hash, table[index]);
}
```

这个方法将构建一个HashMapEntry, 第四个参数是next字段, 它指向table[index], 再将这个HashMapEntry赋给table[index], 这就是让新建的HashMapEntry指向table[index]指向链表的头部, 新建的HashMapEntry作为链表新的头部, 让table[index]指向它, 这样完成插入到链表头部的操作.

## doubleCapacity扩容

```
private HashMapEntry<K, V>[] doubleCapacity() {
    HashMapEntry<K, V>[] oldTable = table;
    int oldCapacity = oldTable.length;
    if (oldCapacity == MAXIMUM_CAPACITY) {
        return oldTable;
    }
    int newCapacity = oldCapacity * 2;
    HashMapEntry<K, V>[] newTable = makeTable(newCapacity);
    if (size == 0) {
        return newTable;
    }

    ......
}
```

正如方法名所提示的, 这个扩容操作是将table的容量扩大一倍. 如果发现当前HashMap的size为0, 那么直接返回newTable, 否则就继续进行下面的rehash操作:

```
private HashMapEntry<K, V>[] doubleCapacity() {
    ......
    for (int j = 0; j < oldCapacity; j++) {
        /*
         * Rehash the bucket using the minimum number of field writes.
         * This is the most subtle and delicate code in the class.
         */
        HashMapEntry<K, V> e = oldTable[j];
        if (e == null) {
            continue;
        }
        int highBit = e.hash & oldCapacity;
        HashMapEntry<K, V> broken = null;
        newTable[j | highBit] = e;
        for (HashMapEntry<K, V> n = e.next; n != null; e = n, n = n.next) {
            int nextHighBit = n.hash & oldCapacity;
            if (nextHighBit != highBit) {
                if (broken == null)
                    newTable[j | nextHighBit] = n;
                else
                    broken.next = n;
                broken = e;
                highBit = nextHighBit;
            }
        }
        if (broken != null)
            broken.next = null;
    }
    return newTable;
}
```


不要看这里位运算又花里胡哨的, 其实本质上就是使用新的table.length重新计算各个Entry的hash值对应的index, 只是这里有个小的trick.

```
int highBit = e.hash & oldCapacity;
```

这句是用hash和旧的table.length做&, 还是举例子说明比较好懂.

假如旧table容量为4 (oldCapacity = 0100), 存有3个Entry, hash分别是1, 5, 9.

他们对应的index应该是

```
0001 & 0011 = 0001
0101 & 0011 = 0001
1001 & 0011 = 0001
```

也就是都在index == 1的地方.

```
table[0] -> null
table[1] -> hash_1 -> hash_5 -> hash_9 -> null
table[2] -> null
table[3] -> null
```

当用它们的hash分别和oldCapacity (0100)做&, 得到结果如下

```
0001 & 0100 = 0000
0101 & 0100 = 0100
1001 & 0100 = 0000
```

这就是源码中计算的highBit. 这个怎么用, 比如其中有1, 代表重新hash后index将发生变化, index是什么呢? 用highBit | oldIndex就是.

这东西要说也简单, 之前计算index是hash和0011(4 - 1)做&运算, 扩容后计算index是hash和0111 (8 - 1)做&运算, 也就是说取模的时候多算一位, 这一位就是highBit.

拿highBit和旧index做|运算, 自然就和用hash和(新length - 1)做&运算的结果相同.

broken变量用来代表被截断的一方, rehash时是把一条链表拆开, 拿刚才的例子来说, 把`hash_1 -> hash_5 -> hash_9 -> nullrehash`到长度为8的table中:

第一轮

```
table[0] -> null
table[1] -> hash_1 -> hash_5 -> hash_9 -> null
table[2] -> null
table[3] -> null
table[4] -> null
table[5] -> null
table[6] -> null
table[7] -> null
```

此时broken == null

第二轮

```
table[0] -> null
table[1] -> hash_1 -> (仍指向hash_5) <===broken
table[2] -> null
table[3] -> null
table[4] -> null
table[5] -> hash_5 -> hash_9 -> null
table[6] -> null
table[7] -> null
```

第三轮

```
table[0] -> null
table[1] -> hash_1 -> hash_9 -> null
table[2] -> null
table[3] -> null
table[4] -> null
table[5] -> hash_5 -> (仍指向hash_9) <===broken
table[6] -> null
table[7] -> null
```

最后broken.next置null:

```
table[0] -> null
table[1] -> hash_1 -> hash_9 -> null
table[2] -> null
table[3] -> null
table[4] -> null
table[5] -> hash_5 -> null <===broken
table[6] -> null
table[7] -> null
```

rehash完毕.

## remove操作

和put操作类似, 先二次hash再取模获取index, 到table[index]里找到对应的Entry, 从链表中删除. size和modCount均减一.

## clear操作

将table中的元素全部置null, entryForNullKey置null, size置0, modCount加一.

## HashIterator

HashMap中所有的Iterator都是继承自HashIterator, 每次调用iterator都会重新new一个对应的Iterator, 创建HashIterator时会将当前HashMap中modCount的值存到HashIterator成员变量expectedModCount中, 每次调用Iterator.next, 都会调到HashIterator.nextEntry, 在HashIterator.nextEntry中会比较modCount和expectedModCount, 如果不相等, 抛出ConcurrentModificationException. 如果调用了Iterator.remove, expectedModCount会重新赋值. 程序里不要通过这个异常来判断有没有并发修改HashMap, 因为它极有可能检测不到并发修改. 这个后面说.

## 多线程问题

看了HashMap的源码, 很明显的感觉就是在多线程下不加同步, 很容易出大乱子, 比较明显的是HashMap扩容, 还有比如对size的修改并非原子性的, 多线程下可能出现size错乱的情况. 最简单粗暴的方法是像HashTable那样, 对每个公开的方法进行同步, 这样当然是可以解决问题的, 但是整个HashTable的性能也因此大打折扣. 之前遇到的一个并发操作HashMap导致的内存泄露的问题, 在换用ConcurrentHashMap之后就消失了.

虽然有些地方喜欢称呼ConcurrentHashMap为免锁容器, 但实际上JDK6中的ConcurrentHashMap内部还是有锁的, 只是将锁的粒度变得更细了. HashTable相当于是锁住整个table, 而ConcurrentHashMap则将锁分段(Segment), 当操作发生在不同的Segment上时, 可以并行操作. 另外有些操作可以不锁住Segment进行, 这些就需要Java内存模型的知识了.

## Java内存模型(Java Memory Model)
影响多线程安全的不只是我们直观理解的执行顺序的问题, 还有重排序和内存可见性.

### 内存可见性

内存可见性比较直观的例子如下:

```
public class NoVisibility {
    private static boolean ready;

    private static class ReaderThread extends Thread {
        public void run() {
            while (!ready) {
                Thread.yield();
            }
        }
    }

    public static void main(String[] args) {
        new ReaderThread().start();
        ready = true;
    }
}
```

启动一个ReaderThread反复读ready的值, 发现它变成true后线程就可以结束了, 但这个线程可能会持续循环下去, 因为ReaderThread可能永远都看不到ready的值变为true了.

如果缺少同步, 将会有很多因素使线程无法立即看到, 甚至永远看不到另一个线程的操作结果, 编译器可能将变量保存在寄存器中, 缓存可能改变将写入变量提交到主存的次序, 保存在处理器本地缓存中的值对其他处理器不可见, 等等.

### 重排序

重排序的一个相当经典的例子是双重检查锁(Double-checked locking)

```
class DoubleCheckedLocking {
    private static Instance instance;

    public static Instance getInstance() {
        if (instance == null) {
            synchronized(DoubleCheckedLocking.class) {
                if (instance == null) {
                    instance = new Instance();
                }
            }
        }
        return instance;
    }
}
```

单例模式中, 一方面我们想getInstance能尽快返回, 不对它加锁, 另一方面又希望不要创建多个实例出来, 所以就有人想出了双重检查锁(DCL)这么一个看似牛逼的解决方案. 很多书上还对这种实现赞不绝口.

至今还有人在讨论单例模式时认为上面代码实现的DCL非常棒, 我第一次见DCL是在一本叫剑指Offer的书上, 这本书也成功误导了我. 我发现大部分语言中, DCL都是不安全的, 除非给instance变量加上volatile修饰.

这里的问题在于重排序可能会导致instance先指向了一个没有完成初始化的实例, 而第二个线程直接拿着一个没有初始化完毕的实例去用了, 这是完全有可能的, Java语言规范要求JVM在线程中维护一种类似串行的语义: 只要程序最终结果与在严格串行环境中执行的结果相同, 那么重排序, 缓存等等优化操作都是允许的. 上面说的那种重排序并不影响单线程中的执行, 所以是允许的.

在多线程环境中, 维护程序的串行性将导致很大的开销, 所以多个线程共享数据时, 仅在必要时才协调它们之间的操作.

假设一个线程为变量a赋值:

```
a = 1;
```

在什么条件下, 读取a的线程将看到这个值为1?

这就是Java内存模型(JMM)需要解决的问题.

### Happens-Before关系

我们不需要关心不同架构上内存模型之间的差异, Java提供了自己的内存模型, 屏蔽了底层的内存模型. JMM中定义了一系列操作, 这些操作间有一个Happens-Before关系, 如果想保证执行操作B的线程看到操作A的结果, A和B之间必须满足Happens-Before关系, 否则JVM可以对它们任意的重排序, 也不保证可见性. 其中几个跟ConcurrentHashMap原理联系紧密的如下:

> 程序顺序规则: 同一个线程中, 任意操作happens-before任意后续操作<br/>
> 监视器锁规则: 对监视器锁上的解锁操作happens-before后续加锁操作<br/>
> volatile变量规则: 对volatile变量的写happens-before后续对其读操作<br/>
> 传递性规则: A happens-before B, B happens-before C 则 A happens-before C<br/>
> 另外happens-before关系仅仅是一个操作对另一个操作可见, 并不是说一定要谁先执行谁后执行

另一个需要知道的是final的语义.

### final的语义

对象O的一个字段被声明为final, 不仅仅是说这个字段不可变, Java还保证只要这个对象能安全构造, 即便不安全发布, 其他线程也能看到O对final的写入, 但是普通的字段就必须同时做到安全构造和安全发布才能有这个保证.

安全构造就是说不要在构造方法还未完成前就把对象的引用this泄漏出去了. 比如在构造方法中将自身注册到某个类中去, 即便这个注册调用是在构造方法的最后一行, 也是提前泄露了this, 提前泄漏this会导致其他线程看到一个没有完全初始化的对象.

安全发布, 则是DCL中存在的情形, 对象可能在没有完成构造的情况下就被发布并使用了.

知道这些也就足够分析JDK 6中ConcurrentHashMap的实现了

## ConcurrentHashMap原理
源码版本: Android 15, 应该是基于JDK 6-b27修改得来.

### 存储形式
ConcurrentHashMap内部也是有用锁来同步的, 只是将锁的粒度缩小为段(Segment), 一个segment的结构其实和HashMap是类似的, 可以理解为ConcurrentHashMap里面有很多个HashMap, 每个HashMap都有一把锁来同步对它的部分操作, 当操作发生在不同的HashMap, 自然就能并发执行. segment也是以数组的形式存储的.

```
segment[0] -> table0[]
segment[1] -> table1[]
segment[2] -> table2[]
......
```

### ConcurrentHashMap创建过程

ConcurrentHashMap提供了五个构造方法

```
public ConcurrentHashMap()
public ConcurrentHashMap(int initialCapacity)
public ConcurrentHashMap(int initialCapacity, float loadFactor)
public ConcurrentHashMap(int initialCapacity,
                             float loadFactor, int concurrencyLevel)
public ConcurrentHashMap(Map<? extends K, ? extends V> m)
```

直接看参数最多的就好了:

initialCapacity是初始大小, 用来决定Segment中的table数组初始大小, 默认16. segment中的成员和HashMap类似.

loadFactor是用来初始化segment中的threshold, segment中entry数量超过threshold, 同样会有扩容操作. 默认0.75f

concurrencyLevel用来决定segment数组初始化大小, 默认16.

```
static final int MAX_SEGMENTS = 1 << 16;

final int segmentMask;

final int segmentShift;

final Segment<K,V>[] segments;

public ConcurrentHashMap(int initialCapacity,
                         float loadFactor, int concurrencyLevel) {
    if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();

    if (concurrencyLevel > MAX_SEGMENTS)
        concurrencyLevel = MAX_SEGMENTS;

    // Find power-of-two sizes best matching arguments
    int sshift = 0;
    int ssize = 1;
    while (ssize < concurrencyLevel) {
        ++sshift;
        ssize <<= 1;
    }
    segmentShift = 32 - sshift;
    segmentMask = ssize - 1;
    this.segments = Segment.newArray(ssize);

    ......
}
```

开头仍然是一些防御性代码, 这里concurrencyLevel也不是直接用的, 仍然是找到大于等于它的最小的2的幂, 这里用的位移操作就比之前在HashMap中看到的要易懂一点, 得到的结果存入ssize.

segmentShift中保存了移位相关的数据, segmentMask则保存了ssize - 1, 做掩码使用, 这个可以猜到是决定使用哪个segment时做位运算用的.

然后调用Segment.newArray创建ssize大小的segment数组

```
@SuppressWarnings("unchecked")
static final <K,V> Segment<K,V>[] newArray(int i) {
    return new Segment[i];
}
```

继续看构造方法的代码:

```
public ConcurrentHashMap(int initialCapacity,
                         float loadFactor, int concurrencyLevel) {
    ......

    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    int c = initialCapacity / ssize;
    if (c * ssize < initialCapacity)
        ++c;
    int cap = 1;
    while (cap < c)
        cap <<= 1;

    for (int i = 0; i < this.segments.length; ++i)
        this.segments[i] = new Segment<K,V>(cap, loadFactor);
}
```

同样是找到initialCapacity的对应的2的幂, 接着将segments数组中的所有的元素都new出来, 至此构造方法完毕.

### put操作

ConcurrentHashMap不允许使用null做key, 也不允许使用null做value. 和put相关的操作一共有三个:

```
public V put(K key, V value)
public V putIfAbsent(K key, V value)
public void putAll(Map<? extends K, ? extends V> m)
```

其中第三个其实就是循环调用第一个, 直接看第一个和第二个的代码就好:

```
public V put(K key, V value) {
    if (value == null)
        throw new NullPointerException();
    int hash = hash(key.hashCode());
    return segmentFor(hash).put(key, hash, value, false);
}

public V putIfAbsent(K key, V value) {
    if (value == null)
        throw new NullPointerException();
    int hash = hash(key.hashCode());
    return segmentFor(hash).put(key, hash, value, true);
}
```

两个方法的代码几乎一样, 而且都很简单, 一个是put, 一个是仅在对应的key不存在时put, 都是对`key.hashCode()`做一次hash之后, 传入segmentFor方法, 拿到一个对象之后调用它的put就结束了.

我看的这个旧版本的代码中hash方法用的算法和HashMap中的二次hash是一样的, 不管这个, 看segmentFor方法:

```
final Segment<K,V> segmentFor(int hash) {
	return segments[(hash >>> segmentShift) & segmentMask];
}
```

这里就是之前保存的segmentShift和segmentMask派上用场的地方, ConcurrentHashMap是使用hash的高位来决定一个Entry应该放入哪个segment. 与HashMap相同, hash的低位用来决定Entry具体放在Segment中table的哪个位置. 因为ConcurrentHashMap对hash的高位和低位都比较依赖, 更加不能直接用Object.hashCode()的值.

其实不光put, remove, replace等方法都是调用Segment对应的方法, 所以我们得先看看Segment到底是个什么东西.

## Segment原理

先看看Segment是个什么东西:

```
static final class Segment<K,V> extends ReentrantLock
            implements Serializable
```

Segment继承ReentrantLock, 也就是说每个Segment自己就是一把锁, 对所有会改变自己持有的table的操作, 可以通过自己这把锁来同步.

### Segment构造过程

```
transient int threshold;

transient volatile HashEntry<K,V>[] table;

final float loadFactor;

Segment(int initialCapacity, float lf) {
    loadFactor = lf;
    setTable(HashEntry.<K,V>newArray(initialCapacity));
}

void setTable(HashEntry<K,V>[] newTable) {
    threshold = (int)(newTable.length * loadFactor);
    table = newTable;
}
```

这里仅仅是赋值和初始化对应大小的table, threshold是当前table的容量乘上loadFactor, table是HashEntry数组, 这个和HashMap中用的HashMapEntry还有点不一样.

## HashEntry与HashMapEntry

HashMap中使用的HashMapEntry

```
static class HashMapEntry<K, V> implements Entry<K, V> {
    final K key;
    V value;
    final int hash;
    HashMapEntry<K, V> next;

    ......
}
```

ConcurrentHashMap中使用的HashEntry

```
static final class HashEntry<K,V> {
    final K key;
    final int hash;
    volatile V value;
    final HashEntry<K,V> next;

    ......
}
```

成员虽然相近, 但HashEntry中除了value, 均使用final修饰, 而value则使用volatile修饰. 前面说过final在JMM中的语义, 只要这个HashEntry安全构造, 其他线程访问它的这三个final域, 能看到正确写入的值. value则没有保证, 这一点等会儿有讨论.

### Segment.put操作

知道了Segment怎么创建的, 就来看它的put操作.

```
transient volatile int count;

V put(K key, int hash, V value, boolean onlyIfAbsent) {
    lock();
    try {
        int c = count;
        if (c++ > threshold) // ensure capacity
            rehash();
        ......
    } finally {
        unlock();
    }
}
```

segment.put第一句就是lock, 因为put基本上都是会改变table中一些元素的结构的, 所以采取了最保守的做法来同步, 要想在segment持有的table中进行一些结构上的更改, 必须先拿到锁.

这里先拿到count, 存入局部变量c, 自增也是对c进行, 因为这个方法不一定会导致table中新增一个HashEntry, 不过不论会不会新增一个HashEntry, c++ > threshold满足时rehash方法是会调用的, 这个rehash就是我们常说的扩容, 扩容等会儿再说, 先继续看.

```
transient int modCount;

V put(K key, int hash, V value, boolean onlyIfAbsent) {
    lock();
    try {
        ......
        HashEntry<K,V>[] tab = table;
        int index = hash & (tab.length - 1);
        HashEntry<K,V> first = tab[index];
        HashEntry<K,V> e = first;
        while (e != null && (e.hash != hash || !key.equals(e.key)))
            e = e.next;

        V oldValue;
        if (e != null) {
            oldValue = e.value;
            if (!onlyIfAbsent)
                e.value = value;
        }
        else {
            oldValue = null;
            ++modCount;
            tab[index] = new HashEntry<K,V>(key, hash, first, value);
            count = c; // write-volatile
        }
        return oldValue;
    } finally {
        unlock();
    }
}
```

后面的逻辑就很清晰了, 和HashMap类似, 先找到对应的Entry, 之后就是决定要不要覆盖它的值, 如果没找到, 就新建一个HashEntry插入, 插入前先将modCount加1, 这个modCount和HashMap中的modCount是同一个意思, 但是因为这里代码的特殊性, 它功能更强, 尽管它不是用来抛ConcurrentModificationException的. 注意这里将c赋值给了count, 旁边还有注释// write-volatile, 这有什么深意呢? 等会儿再说.

### Segment.rehash扩容

Segment的扩容和HashMap的扩容目的是一致的, 但是实际细节有所不同, 主要是因为HashEntry.next字段是final的, 所以链表是拆不开的, 必须要克隆一些HashEntry重新插入, 这部分的模拟过程如下:

```
table[0]->null
table[1]->hash_1 ->hash_5 ->hash_9 ->hash_13 ->hash_21
table[2]->null
table[3]->null
```

真正开始前更新threshold.

第一轮: 找到最末尾将要被hash到同一个位置的连续Entry链, 插入新的table, 这是唯一重用的Entry部分.

```
table[0]->null
table[1]->null
table[2]->null
table[3]->null
table[4]->null
table[5]->hash_13 ->hash_21
table[6]->null
table[7]->null
```

第二轮: clone剩下的链表中的Entry, 逐个插入table, 因为next是final的, 只能从链表的头部插入, 又因为是顺着链表克隆, 所以插入后的顺序是反的:

插入hash_1

```
table[0]->null
table[1]->hash_1
table[2]->null
table[3]->null
table[4]->null
table[5]->hash_13 ->hash_21
table[6]->null
table[7]->null
```

插入hash_5

```
table[0]->null
table[1]->hash_1
table[2]->null
table[3]->null
table[4]->null
table[5]->hash_5-> hash_13 ->hash_21
table[6]->null
table[7]->null
```

插入hash_9

```
table[0]->null
table[1]->hash_9-> hash_1
table[2]->null
table[3]->null
table[4]->null
table[5]->hash_5-> hash_13 ->hash_21
table[6]->null
table[7]->null
```

rehash完毕.

### Segment.remove操作

看完put再来看remove

```
V remove(Object key, int hash, Object value) {
    lock();
    try {
        int c = count - 1;
        ......

        V oldValue = null;
        if (e != null) {//e是通过我们找到的Entry
            V v = e.value;
            if (value == null || value.equals(v)) {
                ......
                ++modCount;
                ......
                tab[index] = newFirst;
                count = c; // write-volatile
            }
        }
        return oldValue;
    } finally {
        unlock();
    }
}
```

这个方法细节都被我省略了, 直接说明就好, 和put一样, 这个方法也有很大可能改变table中元素的结构, 一上来就调用lock, 找到对应的Entry存入e中, 如果e不为null就进入删除逻辑, 这都很简单, 具体的删除逻辑因为HashEntry.next字段是final的, 所以和rehash中类似, 将被删除节点后面的链表直接接在table的对应位置, 然后将被删除节点之前的Entry全部克隆后挨个插到链表头部即可.

这里的判断条件`(value == null || value.equals(v))`可能会令人费解, 主要因为代码都被我省略了, 这个方法的第三个参数value在为null时, 方法会删除key对应的Entry, 如果不为null, 仅在key和value都匹配时才删除.

在真的删除节点发生前, 同样会对modCount自增, 注意看这里在删除逻辑中, 我们之前见过的注释// write-volatile又出现了. 我们再看看其他地方还有没有这个注释

### Segment.replace操作

Segment.replace方法有两种形式, 只看一种就好, 大同小异

```
V replace(K key, int hash, V newValue) {
    lock();
    try {
        HashEntry<K,V> e = getFirst(hash);
        while (e != null && (e.hash != hash || !key.equals(e.key)))
            e = e.next;

        V oldValue = null;
        if (e != null) {
            oldValue = e.value;
            e.value = newValue;
        }
        return oldValue;
    } finally {
        unlock();
    }
}
```

虽说replace不改变table中Entry的数量, 它还是在锁的保护下进行的, 这个方法就没有那个奇怪的注释// write-volatile, 它的逻辑也是很清晰的, 找到Entry, 如果存在就替换它的value, 没必要多说.

### Segment.clear操作

到目前为止我们还没见过不加锁的操作, clear操作也是加锁的, 只是它是先检查再加锁, 这里就稍微有些问题了.

```
void clear() {
    if (count != 0) {
        lock();
        try {
            HashEntry<K,V>[] tab = table;
            for (int i = 0; i < tab.length ; i++)
                tab[i] = null;
            ++modCount;
            count = 0; // write-volatile
        } finally {
            unlock();
        }
    }
}
```

clear首先判断count != 0, 满足才去加锁处理, 如果`count == 0`自然就结束了.

考虑这样一种情况, A线程线调用put, B线程再调用clear, 尽管put操作比clear在时间上早那么一点, 但最终结果很有可能ConcurrentHashMap中仍有HashEntry存在. 这是多线程下的竟态条件, 虽说count是个volatile值, B线程读取时它必定是最新的, 但因为B在不加锁的情况下读取count, A此时可能还没有执行到给count赋值的那一步. B读取count时, 它的最新值的确是0, B发现`count == 0`, clear操作完成, 之后A才把count赋值为1.

### Segment.get操作

现在再介绍相当常用的get操作

```
V get(Object key, int hash) {
    if (count != 0) { // read-volatile
        HashEntry<K,V> e = getFirst(hash);
        while (e != null) {
            if (e.hash == hash && key.equals(e.key)) {
                V v = e.value;
                if (v != null)
                    return v;
                return readValueUnderLock(e); // recheck
            }
            e = e.next;
        }
    }
    return null;
}
```

和clear类似, 这个操作也是先检查count, 它和clear有同样的问题. 此外, 即便通过检查, get也没有加锁, 先通过hash获取table中对应的链表头, 然后遍历找到自己要找的那个Entry, 如果这个Entry的value为null, 会加锁重试一次.

之前说了, 多线程下, 不仅仅是通常理解的执行顺序可能会影响程序的结果, 还有重排序和内存可见性, HashMap中不加同步的情况下, 即便是A线程put操作执行完毕了, B线程去get同一个Entry也可能会get到一个null. 我们也说了, ConcurrentHashMap保证get操作能看到已经完成的put操作, 换句话说A线程put方法完成后, B线程再去get同一个Entry, 保证B能拿到, 不怕因为缓存等等问题put进去的东西对B不可见, ConcurrentHashMap中的get方法实际上调用Segment.get, 他俩都不加同步, 如何保证?

注意方法体第一行的注释// read-volatile, 这个注释和我们之前看到的// write-volatile遥相呼应, 必定有深意.

## volatile变量count

回顾一下上文说的JMM的happens-before关系:

> 程序顺序规则: 同一个线程中, 任意操作happens-before任意后续操作<br/>
> 监视器锁规则: 对监视器锁上的解锁操作happens-before后续加锁操作<br/>
> volatile变量规则: 对volatile变量的写happens-before后续对其读操作<br/>
> 传递性规则: A happens-before B, B happens-before C 则 A happens-before C<br/>
> 每次注释都和count变量一起出现, 看看count是怎么定义的

```
transient volatile int count;
```

get方法不加锁, 所以监视器锁规则是没法利用了, 但程序顺序规则, volatile规则, 传递性规则三条联合起来就不一样了.

我们的前提是保证get能看到已经完成的put, 现在我们看一下put和get方法

```
V put(K key, int hash, V value, boolean onlyIfAbsent) {
    lock();
    try {
        ......
        if (e != null) {
            ......
        }
        else {
            ......// 下面这句代码之前的操作统称A操作
            count = c; // write-volatile B操作
        }
        return oldValue;
    } finally {
        unlock();
    }
}
```

我们把`count = c;`之前所有的操作称为A, `count = c;`称为B. 如果`A happens-before B`, 记为`hb(A, B)`.

根据程序顺序规则, hb(A, B)成立.

```
V get(Object key, int hash) {
    if (count != 0) { // read-volatile C操作
        ......// 把count != 0之后的操作称为D操作
    }
    return null;
}
```

我们把count != 0称为C, 之后的操作称为D, 同样根据程序顺序规则有`hb(C, D)`, 而B是一个volatile写, C是一个volatile读, 根据volatile变量规则, 有hb(B, C).

现在`hb(A, B), hb(B, C), hb(C, D)`, 根据传递性规则`hb(A, D)`也成立, 也就是说我们在get中能看到之前put放入的Entry. 所以ConcurrentHashMap是能保证同一个table上, get能看到已经完成的put这一点的, 但是对于同一个table上get一个正在发生的put, 并不做保证.

这里利用一个volatile变量, 保证了普通变量的可见性, 作者对JMM中规则的利用令人惊叹.

### readValueUnderLock是否必须

之前在Segment.get方法中, 我们注意到有一个加锁重试调用:

```
V readValueUnderLock(HashEntry<K,V> e) {
    lock();
    try {
        return e.value;
    } finally {
        unlock();
    }
}
```

注意这个方法是在拿到HashEntry, 且HashEntry.value == null时调用的, 不是拿不到HashEntry时调用的.

这个方法本身是很让人疑惑的. 毕竟ConcurrentHashMap不允许value为null, 所以get出来的HashEntry.value为null, 一定是出了什么问题, 但构造HashEntry时value就已经赋值了, 它还是volatile类型, 整个HashEntry是安全构造的, get方法都拿到HashEntry了, 为什么会出现value为null的情况?

这就需要抠字眼了, 前面说对象O的final字段, Java保证安全构造后, 不需要安全发布, 其他线程在拿到O的引用的情况下访问这个final字段, 能看到正确的值, 而ConcurrentHashMap中的HashEntry也的确满足这个条件, HashEntry是安全构造的. 但要注意HashEntry不是安全发布的, 双重检查锁(DCL)也是因为没有安全发布单例, 导致它是不安全的. Java虽然保证对象的final字段在安全构造, 不安全发布的情况下其他线程能读到它初始化的值而非默认值(0, null, false), 但从来没有保证volatile字段也享受这个待遇! 所以这里仍然有可能遇到get方法拿到了一个没有完全构造好的HashEntry对象, 因此它要加锁重试.

在这一点上, ConcurrentHashMap的作者Doug Lea也认为这一句加锁重试可能永远都不会被调用, 但JLS/JMM (JLS: Java Language Specification)没有说清楚的事情, 谁也不能保证, 万一哪天JLS/JMM明确指出说volatile字段也需要安全发布才能访问, 现有的ConcurrentHashMap仍然是安全的

从get方法的代码中, 可以看出我们如果put一个值之后, 如果put操作还没有完成, 就去get, 八成get出来是null, 尤其是在count == 0的情况下更容易重现, 一般我们说ConcurrentHashMap是弱一致性的, 说的就是这种问题. ConcurrentHashMap对我们的保证是, put完成之后, 再去get, 那么多线程下get方法能看到之前put的值.

所以如果我们需要强一致性的保证, 还是老老实实同步的好.

### isEmpty操作

Segment中比较重要, 值得分析的东西都看完了, 接下来继续看ConcurrentHashMap中的方法isEmpty.

看代码之前首先要明确一点, ConcurrentHashMap没有一个全局的锁来锁住所有的Segment, 也不能有, 否则不仅性能受影响, 还需要避免死锁, 没这个必要, 但是对于size, isEmpty这类方法, 就有些麻烦了.

因为这类方法一定是遍历segments数组, 怎么保证遍历的时候, 没有其他线程操作segment, 保证结果有效呢?

ConcurrentHashMap采取的策略是检查两次, 如果两次检查间table上没有对应的修改, 且两次检查结果均发现满足empty, 就返回true, 否则返回false.

```
public boolean isEmpty() {
    final Segment<K,V>[] segments = this.segments;
    /*
     * We keep track of per-segment modCounts to avoid ABA
     * problems in which an element in one segment was added and
     * in another removed during traversal, in which case the
     * table was never actually empty at any point. Note the
     * similar use of modCounts in the size() and containsValue()
     * methods, which are the only other methods also susceptible
     * to ABA problems.
     */
    int[] mc = new int[segments.length];
    int mcsum = 0;
    for (int i = 0; i < segments.length; ++i) {
        if (segments[i].count != 0)
            return false;
        else
            mcsum += mc[i] = segments[i].modCount;
    }
    // If mcsum happens to be zero, then we know we got a snapshot
    // before any modifications at all were made.  This is
    // probably common enough to bother tracking.
    if (mcsum != 0) {
        for (int i = 0; i < segments.length; ++i) {
            if (segments[i].count != 0 ||
                    mc[i] != segments[i].modCount)
                return false;
        }
    }
    return true;
}

```

代码本身不长, 注释很多, 但是也好理解.

mc数组用来记录每个segment的modCount, mcsum用来累加每个segment.modCount, mcsum相当于一个小优化, 第一轮检查发现每个segment.size都为0的情况下, 如果mcsum == 0, 表示ConcurrentHashMap在创建之后还没有往里面塞过HashEntry, 则认为这个结果可信, 直接返回true就好. 如果mcsum不为0, 则再检查一次, 大部分情况下都是不为0的, 因为modCount是只增不减的. 第二次检查就会先检查segment.count是否仍然为0, 再检查segment.modCount和之前保存的是否相等, 只有所有的都检查一遍之后, 都满足, 才返回true, 否则直接就返回false了.

其实这里也只是尽量给一个相对准确的返回值, 这也是弱一致性的体现.

### modCount的用处和可见性
接下来有个问题: 为什么要用modCount, 两次检查只检查size不行吗?

这是为了避免ABA问题, 多线程下假如我们想不加锁的情况下知道一个segment是不是空的, 一般想到的就是读取两遍size, 如果均为0, 就说明这个segment是空的, 但是如果我们第一次读取size为0, 中途另一个线程先往其中put了一个Entry, 随后又remove掉, 我们第二次读取size的确还是0, 此时我们无法准确判断, 所以引入一个版本号, 每次put和remove的时候不仅修改size, 还要将版本号加1, 这样我们两次读取size, 不仅要读取的size相等, 而且还要版本号相等, 这样我们认为结果可信.

还有一个问题: modCount是普通int变量, 如何保证可见性?

这里的原理仍然是利用count是volatile变量, 细心的同学可能注意到了, put, remove这类代码中, 每次修改modCount的代码, 都写在修改size的代码的前面, 而isEmpty这类需要读取modCount的代码, 都会在读取size之后才被执行.

根据前面的volatile和happens-before规则, 自然就保证了modCount的可见性, 总结一句话就是: volatile写前的代码, 不能重排序到volatile写后, volatile读后的代码不能重排序到volatile读前.

### size方法

获取ConcurrentHashMap的size, 和isEmpty方法的思想类似, 也是先把所有segment.size加一遍, 同时记录modCount, 然后再把segment.size加一遍, 同时比较modCount, modCount均相同, 且size的和两次都相同, 则直接返回size和就好, 唯一不同的是, 这里如果第一次两步计算发现segments变了, 有个重试的机制, 默认是重试一次, 如果重试后能得到结果就返回了, 如果重试还不能得到结果, 就对每个segment进行加锁, 锁完之后一起算一个size和出来, 再挨个解锁, 最后返回.

代码比较长, 但逻辑很清晰, 就不贴了.

## 尾声

上面这一长串分析下来, 也就把早期版本的HashMap和ConcurrentHashMap中需要注意的点说完了, 主要是一些神奇的位运算, 内部的插入删除方式, ConcurrentHashMap中的锁分段机制, 无锁操作的可见性处理, 以及背后所依靠的Java内存模型相关的知识.

最后, 我也是才看的源码, 刚了解的Java内存模型, 如果文中有错, 欢迎指出, 我会及时更正.