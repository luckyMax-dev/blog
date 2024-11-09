# ThreadLocal 源码阅读 2

先尝试回答上一篇文章中提出的问题，

问：既然有了 remove 方法，为什么作者还要在 set、get、remove 方法里写逻辑去检测数组中哪些元素被回收并释放 value ？

答：

1. 一方面是减少因为人为操作而出现的内存泄漏问题，如果使用者在写业务代码时忘记写 remove 方法，那作者的检测逻辑就能回收一部分已经内存泄漏的 value，但这种方法只能推迟内存泄漏发生的时间，最终还是会发生内存泄漏导致 JVM 进程无法服务的问题；
2. 另外一个方面就是维护数组的均匀散列与查改效率，将旧值移除，将哈希冲突键移动，都可以让性能提高；

接下来继续看几个方法的源码。

### expungeStaleEntry 方法

expungeStaleEntry 顾名思义，移除陈旧的 Entry，这个方法有两个逻辑：

1. 用于线性检测 Entry 数组的一组 key 是否已被回收，若已被回收则释放 value，
2. 当 key 不在预期的、经过哈希计算出来的位置时，将其移动至预期位置；

源码如下：

```java
#0
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;
    // 首先清除传进来的位置的引用关系
    tab[staleSlot].value = null;	// 取消值引用
    tab[staleSlot] = null;				// 释放 ThreadLOocal
    size--;
    // 线性探测，遇到 Entry 为 null 才停止
    Entry e;
    int i;
    for (i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
        if (k == null) {	// 若 ThreadLocal 已经被回收，释放 key 和 value
            e.value = null;
            tab[i] = null;
            size--;
        } else {
          	// ThreadLocal 没有被回收，计算预期的索引位置
            int h = k.threadLocalHashCode & (len - 1);
          	// 与当前所在的位置不相等，说明当前遍历到的 key 不属于该位置，移动它
            if (h != i) {
              	// 先释放当前位置的 ThreadLocal
                tab[i] = null;
                // 线性探测 key 预计所在的索引位置，找到一个为 null 的位置
                while (tab[h] != null)
                    h = nextIndex(h, len);
                tab[h] = e;	// 移动该 Entry
            }
        }
    }
    return i;
}
```

第一个逻辑比较好理解，即线性探测某个索引后续的 ThreadLocal 并释放 value，附图理解：

![release](https://github.com/notayessir/blog/blob/main/images/threadlocal/release.png)

第二个逻辑的作用是什么？根据源码的介绍，这段逻辑用来保证整个 Entry 数组的哈希有序，这样在使用 ThreadLocal 方法时，可以快速定位 Entry，附图理解：

![swap](https://github.com/notayessir/blog/blob/main/images/threadlocal/swap.png)

### cleanSomeSlots 方法

在 set 方法中，还出现了 cleanSomeSlots 方法，我们可以**大概的**将该方法约等同于 expungeStaleEntry，根据 Entry 数组的长度，进行对数级别的检测已被回收的 ThreadLocal 并调用 expungeStaleEntry 释放引用关系；源码如下：

```java
#0
// 参数 i 所在的 entry 一定没有被回收
// 参数 n 的大小决定扫描次数  
private boolean cleanSomeSlots(int i, int n) {
    boolean removed = false;
    Entry[] tab = table;
    int len = tab.length;
    do {
      	// 遍历 i 之后的元素
        i = nextIndex(i, len);
        Entry e = tab[i];
      	// 有 ThreadLocal 被释放
        if (e != null && e.get() == null) {
            n = len;
            removed = true;
          	// 清除 i 所持有的引用关系
            i = expungeStaleEntry(i);
        }
    // 将 n 进行位移操作，即进行 log2(n) 次探测
    } while ( (n >>>= 1) != 0);	
    return removed;
}
```

例如，当 n = 32，则需要在数组的 32，16，8，4，2，1 索引上进行检测，附图理解：

![cleanSomeSlots](https://github.com/notayessir/blog/blob/main/images/threadlocal/cleanSomeSlots.png)

### replaceStaleEntry 方法

replaceStaleEntry 方法，使用新的 ThreadLocal 和 value 替换指定位置的旧的 ThreadLocal 和 value；此外，该方法还会从指定位置开始检测左右两侧的元素，将已经被回收的 ThreadLocal 所在位置执行释放操作；

```java
private void replaceStaleEntry(ThreadLocal<?> key, Object value, int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;
    Entry e;
  	// 往指定位置的左边遍历，找到一个 Entry 使用过，但 ThreadLocal 已经被回收的元素位置
    int slotToExpunge = staleSlot;
    for (int i = prevIndex(staleSlot, len); (e = tab[i]) != null; i = prevIndex(i, len))
        if (e.get() == null)
            slotToExpunge = i;
    // 往指定位置的右边遍历，线性探测
    for (int i = nextIndex(staleSlot, len); (e = tab[i]) != null; i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
      	// 线性探测后找到了因为哈希冲突 key 所在的位置
        if (k == key) {
          	// 设置新的 value
            e.value = value;
          	// 交换因为哈希冲突导致位置不符合预期的两个 Entry
          	// 先将 staleSlot 放在正确位置，保证数组的哈希有序
            tab[i] = tab[staleSlot];
            tab[staleSlot] = e;
            // 决定 slotToExpunge 位置，如果在指定位置的左边找到被回收的索引，就从该位置开始清理和重新哈希
          	// 如果左边没有找到，则从右边开始清理，起始位置为 i
            if (slotToExpunge == staleSlot)
                slotToExpunge = i;
          	// 执行探测清理和重新散列
            cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
            return;
        }
				// 如果 key 为 null，并且 staleSlot 左边没找到已失效的 ThreadLocal
        if (k == null && slotToExpunge == staleSlot)
            slotToExpunge = i;
    }
    // 如果 key 没找到，就在哈希预期的位置设置新的 ThreadLocal 和 value
    tab[staleSlot].value = null;
    tab[staleSlot] = new Entry(key, value);
    // 清理已经被回收的 Entry
    if (slotToExpunge != staleSlot)
        cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
}
```

replaceStaleEntry 附带的清理逻辑有点复杂，附图理解：

情况一，直接替换：

![case1](https://github.com/notayessir/blog/blob/main/images/threadlocal/case1.png)

情况二：1）在预期位置左边**找到**已释放的 ThreadLocal；2）预期位置右边**找到** key 所在位置；

![case2](https://github.com/notayessir/blog/blob/main/images/threadlocal/case2.png)

情况三：1）在预期位置左边**没找到**已释放的 ThreadLocal；2）预期位置右边**找到**已被释放的 ThreadLocal；

![case3](https://github.com/notayessir/blog/blob/main/images/threadlocal/case3.png)

### rehash 方法

rehash 方法进行无效引用回收，源码如下：

```java
#0
private void rehash() {
  	// 见 #1
    expungeStaleEntries();
    // 探测之后，比较回收结果，再决定是否进行扩容
    if (size >= threshold - threshold / 4)
        resize();	// 真正扩容
}

#1，遍历整个 Entry 数组，释放已经被回收的 ThreadLocal 和 value
private void expungeStaleEntries() {
    Entry[] tab = table;
    int len = tab.length;
    for (int j = 0; j < len; j++) {
        Entry e = tab[j];
      	// 若元素已被回收，释放 value
        if (e != null && e.get() == null)
            expungeStaleEntry(j);
    }
}
```

### resize 方法

resize 方法的作用，将数组扩容两倍，并将元素重新散列到新数组上，源码如下：

```java
private void resize() {
    Entry[] oldTab = table;
    int oldLen = oldTab.length;
    int newLen = oldLen * 2;	// 新数组长度为原来的两倍
    Entry[] newTab = new Entry[newLen];
    int count = 0;
  	// 遍历旧数组
    for (int j = 0; j < oldLen; ++j) {
        Entry e = oldTab[j];
        if (e != null) {
            ThreadLocal<?> k = e.get();
          	// ThreadLocal 为 null，释放 value
            if (k == null) {
                e.value = null; // Help the GC
            } else {
              	// 将旧的 entry 均匀的散列到新数组上
                int h = k.threadLocalHashCode & (newLen - 1);
                while (newTab[h] != null)
                    h = nextIndex(h, newLen);
                newTab[h] = e;
                count++;
            }
        }
    }
  	// 设置新的阈值
    setThreshold(newLen);
    size = count;
    table = newTab;
}
```

### 小结

以上几个方法，保证了 ThreadLocal 增删查的效率。

- expungeStaleEntry 方法是基础，在移除引用关系的同时，还会对不在预期位置的 Entry 进行重散列，移到正确位置上；
- replaceStaleEntry 方法，在替换键值的同时，会找到一个已被释放的位置进行顺序清理；
- rehash 和 resize 方法进行数组的扩容，与无效键值释放；