# HashMap和HashTable的区别

我们先看2个类的定义

```
public class Hashtable  
    extends Dictionary  
    implements Map, Cloneable, java.io.Serializable  
public class Hashtable
    extends Dictionary
    implements Map, Cloneable, java.io.Serializable 
```

```
public class HashMap  
    extends AbstractMap  
    implements Map, Cloneable, Serializable  
public class HashMap
    extends AbstractMap
    implements Map, Cloneable, Serializable 
```

可见Hashtable 继承自 Dictiionary 而 HashMap继承自AbstractMap

 

Hashtable的put方法如下

```
public synchronized V put(K key, V value) {  //###### 注意这里1   
  // Make sure the value is not null   
  if (value == null) { //###### 注意这里 2   
    throw new NullPointerException();  
  }  
  // Makes sure the key is not already in the hashtable.   
  Entry tab[] = table;  
  int hash = key.hashCode(); //###### 注意这里 3   
  int index = (hash & 0x7FFFFFFF) % tab.length;  
  for (Entry e = tab[index]; e != null; e = e.next) {  
    if ((e.hash == hash) && e.key.equals(key)) {  
      V old = e.value;  
      e.value = value;  
      return old;  
    }  
  }  
  modCount++;  
  if (count >= threshold) {  
    // Rehash the table if the threshold is exceeded   
    rehash();  
    tab = table;  
    index = (hash & 0x7FFFFFFF) % tab.length;  
  }  
  // Creates the new entry.   
  Entry e = tab[index];  
  tab[index] = new Entry(hash, key, value, e);  
  count++;  
  return null;  
}  
public synchronized V put(K key, V value) {  //###### 注意这里1
    // Make sure the value is not null
    if (value == null) { //###### 注意这里 2
      throw new NullPointerException();
    }
    // Makes sure the key is not already in the hashtable.
    Entry tab[] = table;
    int hash = key.hashCode(); //###### 注意这里 3
    int index = (hash & 0x7FFFFFFF) % tab.length;
    for (Entry e = tab[index]; e != null; e = e.next) {
      if ((e.hash == hash) && e.key.equals(key)) {
        V old = e.value;
        e.value = value;
        return old;
      }
    }
    modCount++;
    if (count >= threshold) {
      // Rehash the table if the threshold is exceeded
      rehash();
      tab = table;
      index = (hash & 0x7FFFFFFF) % tab.length;
    }
    // Creates the new entry.
    Entry e = tab[index];
    tab[index] = new Entry(hash, key, value, e);
    count++;
    return null;
  } 
```
 

注意1 方法是同步的
注意2 方法不允许value==null
注意3 方法调用了key的hashCode方法，如果key==null,会抛出空指针异常 HashMap的put方法如下

```
public V put(K key, V value) { //###### 注意这里 1   
  if (key == null)  //###### 注意这里 2   
    return putForNullKey(value);  
  int hash = hash(key.hashCode());  
  int i = indexFor(hash, table.length);  
  for (Entry e = table[i]; e != null; e = e.next) {  
    Object k;  
    if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {  
      V oldValue = e.value;  
      e.value = value;  
      e.recordAccess(this);  
      return oldValue;  
    }  
  }  
  modCount++;  
  addEntry(hash, key, value, i);  //###### 注意这里    
  return null;  
}  
public V put(K key, V value) { //###### 注意这里 1
	if (key == null)  //###### 注意这里 2
  		return putForNullKey(value);
	int hash = hash(key.hashCode());
	int i = indexFor(hash, table.length);
	for (Entry e = table[i]; e != null; e = e.next) {
 		 Object k;
  		if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
	    	V oldValue = e.value;
			e.value = value;
			e.recordAccess(this);
			return oldValue;
		}
	}
	modCount++;
	addEntry(hash, key, value, i);  //###### 注意这里 
	return null;
} 
```
 

注意1 方法是非同步的
注意2 方法允许key==null
注意3 方法并没有对value进行任何调用，所以允许为null 


> 注意：Hashtable 有一个 contains方法，容易引起误会，所以在HashMap里面已经去掉了,当然，2个类都用containsKey和containsValue方法。 

-----
 

|比较项目|HashMap|Hashtable|
|:--:|:--:|:--:|
|父类|AbstractMap|Dictiionary|
|是否同步|否|是|
|k，v可否null|是|否|

-----

HashMap是Hashtable的轻量级实现（非线程安全的实现），他们都完成了Map接口，主要区别在于

* HashMap允许空（null）键值（key）,由于非线程安全，效率上可能高于Hashtable。
* HashMap允许将null作为一个entry的key或者value，而Hashtable不允许。
* HashMap把Hashtable的contains方法去掉了，改成containsvalue和containsKey。因为contains方法容易让人引起误解。 
* Hashtable继承自Dictionary类，而HashMap是Java1.2引进的Map interface的一个实现。
* 最大的不同是，Hashtable的方法是Synchronize的，而HashMap不是，在多个线程访问Hashtable时，不需要自己为它的方法实现同步，而HashMap 就必须为之提供外同步(Collections.synchronizedMap)。 

Hashtable和HashMap采用的hash/rehash算法都大概一样，所以性能不会有很大的差异。


## 引用:

* [CSDN-Hashtable 和 HashMap的区别](http://blog.csdn.net/shohokuf/article/details/3932967)
