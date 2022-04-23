ThreadLocal 的使用场景之一有，在执行 SQL 之前进行数据库选择或进行读写分离。当在谈论 ThreadLocal 时，最能突出其特点的是**线程独享**这个功能，正确使用可以避免线程竞争带来的衍生问题。为了弄清它的运行机制，在阅读部分源码后，将几点实现细节记录于此。

### ThreadLocal 的组成

![struct](/Users/nuc/Desktop/struct.png)

如图，ThreadLocal 由几个部分参与组成。每个线程的内部，都会持有一个 ThreadLocalMap，存储结构是一个有限长的 Entry 数组；Entry 继承自 WeakReference，持有 ThreadLocal 和 Value，平时使用中，对 ThreadLocal 的 set/get/remove 操作都是对 Entry 数组进行操作，下面先看看 Entry 的源码。

### Entry

Entry 源码很简单：

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
	/** The value associated with this ThreadLocal. */
	Object value;

	Entry(ThreadLocal<?> k, Object v) {
    	super(k);
        value = v;
    }
}
```

通过继承 WeakReference，把引用关系，或者说是纽带设置为 ThreadLocal，然后内部持有一个 value 对象，就是平常在代码里操作的值。虽然代码只有短短几行，但已经说明了因为不当使用导致的**内存泄漏**问题。

回顾一下 WeakReference 的特点，每次 GC 发生时，WeakReference 引用的对象都会被回收，考虑下面这段伪代码，当 GC 发生之后，哪些对象被回收了？

```java
Entry [] entries = new Entry[10];
ThreadLocal t1 = new ThreadLocal();
Object v1 = new Object();
Entry e1 = new Entry(t1, v1);
entries[0] = e1;

ThreadLocal t2 = new ThreadLocal();
Object v2 = new Object();
Entry e2 = new Entry(t2, v2);
entries[1] = e2;
System.gc();
```

答案是，GC 之后，由于弱引用的关系，t1 和 t2 被回收，v1 和 v2 是强引用，没有被回收。这样问题就暴露出来，如果我们将值成功设置到 ThreadLocal，结束业务逻辑时却没有及时调用值移除的 API，就会导致内存泄漏！

### 哈希散列

ThreadLocalMap 内部使用 Entry 数组保存每一个 ThreadLocal，并且使用一个简单的哈希算法，将 ThreadLocal 均匀的散列到数组中，源码如下：

```java
public class ThreadLocal<T> {
    
  	// 每创建一个 ThreadLocal，都会生成新的 hashCode
    private final int threadLocalHashCode = nextHashCode();

    private static AtomicInteger nextHashCode = new AtomicInteger();
		// 固定增量
    private static final int HASH_INCREMENT = 0x61c88647;

    private static int nextHashCode() {
        return nextHashCode.getAndAdd(HASH_INCREMENT);
    }
  	// ignore ...
}
```

可以看出，每创建一个 ThreadLocal 对象，该对象的 hashCode 都会加上 0x61c88647，然后在与 Entry 数组的长度做与运算，定位对象在该数组的位置：

```java
int pos = hashCode & (INITIAL_CAPACITY - 1);
```

例如，若 Entry 数组长度为 16，散列 16 次后的数组索引：

```
0,7,14,5,12,3,10,1,8,15,6,13,4,11,2,9
```

散列 32 次：

```
// round 1
0,7,14,5,12,3,10,1,8,15,6,13,4,11,2,9
// round 2
0,7,14,5,12,3,10,1,8,15,6,13,4,11,2,9
```

若 Entry 数组长度为 32，散列 32 次后的数组索引：

```
0,7,14,21,28,3,10,17,24,31,6,13,20,27,2,9,16,23,30,5,12,19,26,1,8,15,22,29,4,11,18,25
```

可以看出一些规律，两个索引之间间隔都是 7，每一个轮次，索引都不会冲突。关于 hashCode 的增量为什么是 0x61c88647，后续再做探究。

### set 方法

set 源码如下：

```java
#0
public void set(T value) {
    Thread t = Thread.currentThread();
  	// 获取当前线程持有的 ThreadLocalMap
    ThreadLocalMap map = getMap(t);
    if (map != null)
      	// 设置键值对，请看 #1
        map.set(this, value);
    else
      	// 若 map 为空，则构造一个 ThreadLocalMap
        createMap(t, value);
}

#1
private void set(ThreadLocal<?> key, Object value) {
    Entry[] tab = table;
    int len = tab.length;
  	// 根据 ThreadLocal 的 hashCode，定位其在 table 中的索引位置，我们也称为槽位的起始点
    int i = key.threadLocalHashCode & (len-1);
  	// 从槽位的起始点遍历，若 Entry(WeakReference) 对象不为 null，需要持续后移，即进行哈希冲突的探测
    for (Entry e = tab[i]; e != null; e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();
      	// 若 ThreadLocal 是同一个对象，直接覆盖值
        if (k == key) {
            e.value = value;
            return;
        }
      	// 若 ThreadLocal 为 null，说明 GC 时已经将该 Entry 的 ThreadLocal 回收
        if (k == null) {
          	// 在该位置上设置 key 和并使用新的 value 替代旧的 value 
            replaceStaleEntry(key, value, i);
            return;
        }
    }
  	// 槽位起始点为空，创建键值对
    tab[i] = new Entry(key, value);
    int sz = ++size;
  	// 如果 size 超过数组的散列上限（散列因子），则将数组长度 *2 扩容并将原数组上的对象重新散列
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
      	// 重新散列并视情况扩容
        rehash();
}

```

将 set 方法化简，可以看作是根据哈希计算出索引，并在该索引上设置键值，若有冲突，则进行线性探测，放在索引后的某个位置；

set 中还有几个重要方法，如 replaceStaleEntry、cleanSomeSlots、rehash，方法内会扫描 Entry 数组，将已经被回收的 ThreadLocal 进行释放。这里提一个问题，既然有了 remove 方法，为什么作者还要在 set 方法里写逻辑去检测数组中哪些元素被回收并释放 value ？

### get 方法

get 源码如下：

```java
#1
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
      	// 见 #2
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
  	// 如果在 get 之前没有使用 set 设置 value，那就将 value 设置为 null 并返回
    return setInitialValue();
}

#2
private Entry getEntry(ThreadLocal<?> key) {
  	// 根据 hashCode 计算所在位置
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
  	// 若键相等，则返回 value
    if (e != null && e.get() == key)
        return e;
    else
      	// 有可能发生了冲突，进行线性探测，见 #3
        return getEntryAfterMiss(key, i, e);
}

#3
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;
  	// 对 i 后续的位置进行探测
    while (e != null) {
        ThreadLocal<?> k = e.get();
        if (k == key)	  // 命中则返回
            return e;
        if (k == null)	// 该 key 已经被 GC 回收，清除无效 value，避免内存泄漏
            expungeStaleEntry(i);
        else						// 线性探测
            i = nextIndex(i, len);
        e = tab[i];
    }
    return null;
}
```

简单来看，get 方法计算 key 所在的位置是否有效，或进行线性探测拿到最终的 value；

另外，expungeStaleEntry 也是一个重点方法，会检查索引 i 后续的元素是否被回收并释放 value，根据条件还会将元素重新哈希，使数组的散列更均匀。

### remove 方法

remove 源码如下：

```java
#0
private void remove(ThreadLocal<?> key) {
    Entry[] tab = table;
    int len = tab.length;
  	// 计算 key 所在位置
    int i = key.threadLocalHashCode & (len-1);
  	// 线性探测
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        if (e.get() == key) {
            e.clear();	// 解除弱引用
            expungeStaleEntry(i);	// 释放 value
            return;
        }
    }
}
```

remove 方法进行了弱引用的解除，以及调用 expungeStaleEntry 方法进行一系列元素的内存释放。

### 小结

- 方法层面，set、get、remove 依靠一个有效的 hash 方法得出索引，对数组进行增删查操作；
- Entry 的实现，正确理解 WeakReference 的特性可以避免内存泄漏，set、get、remove 是一套组合拳；
- 每次调用 set、get、remove，会进行无效 key 的探测与 value 引用解除，间接释放内存；