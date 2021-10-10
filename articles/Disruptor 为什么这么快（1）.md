Disruptor 是交易公司 LMAX 开源的，目标是达到**低延迟高吞吐**的队列框架，官方文档上有充分的介绍和说明，这里仅作为自己对该框架的一些总结，以及框架中值得借鉴的实现细节。

### 为什么快

- **无锁**。Disruptor 使用的环形队列，数据的放入和取出都没有使用锁，这一点减少了线程对资源的竞争；Java 内置的队列，例如 ABQ，大量使用了锁机制来保护数据的安全性，性能会有所下降；
- **volatile 可见性**。多线程操作同一数据，数据的可见性影响后来线程的逻辑，Disruptor 直接或间接（Unsafe 操作）使用 volatile 关键字来保证数据的可见性；
- **有效的缓存行**。利用 CPU 缓存的特性，避免了数据伪共享的情况，使用 padding 的办法对没有需要共享的变量进行声明，使得数据不会频繁的从主内存中加载到 CPU 缓存；
- **预分配内存**。为了达到低延迟的目标，Disruptor 使用的环形队列，在框架初始化的时候，就已经完成了队列中元素的初始化，避免了垃圾回收带来的停顿（STW）;

这里说的快，体现在数据的出队与出队，并不是指数据走完所有的消费逻辑（取决于处理逻辑的复杂程度）。

### 环形队列

#### 计算缓存行

Disruptor 使用的环形队列，是基于 new Object[] 实现的，为什么不用链表或图、树的数据结构来实现？一个主要的原因是**数组在内存中的布局是连续的**，相对于其他数据结构，数组的读写只需计算相对的元素的偏移量即可，并且该数组是可以**重复使用**的；下面是一个长度为 8 的环形队列抽象图与内存布局图（使用 jol-core 打印内存布局）：

![layout](https://github.com/luckyMax-dev/blog/blob/master/images/disruptor/layout.png)

结合这两张图会发现，队列的头尾（64 bytes 或 128 bytes）有一部分是未使用的，这就是有效缓存行的填充原理，CPU 缓存行的大小一般是 64 bytes 或 128 bytes，为了兼容这两种缓存行，Disruptor 预留了足够的位置，主要是让其他变量与元素隔离开，达到有效缓存的目的，缓存行在框架中的 Sequence.java 中也有应用。下面根据源码看看，它是如何计算头尾填充以及元素定位的：

```java
// RingBuffer.java
static {
  	// 使用 Unsafe 方法，获取到元素在内存的大小
		final int scale = UNSAFE.arrayIndexScale(Object[].class);
		if (4 == scale) {
      	// 设置元素的偏移量，用于根据元素序列号定位内存位置
				REF_ELEMENT_SHIFT = 2;
    } else if (8 == scale) {
        REF_ELEMENT_SHIFT = 3;
    } else {
        throw new IllegalStateException("Unknown pointer size");
    }
  	// 缓存行填充，即 32 个元素或 16 个元素（x4）
    BUFFER_PAD = 128 / scale;
    // 使用 Unsafe 方法，获取到第一个元素在该数组中的内存位置，以 scale = 4 来算，
  	// 16 bytes 的数组头部字节 + 填充字节，REF_ARRAY_BASE = 144 bytes
    REF_ARRAY_BASE = UNSAFE.arrayBaseOffset(Object[].class) + (BUFFER_PAD << REF_ELEMENT_SHIFT);
}
```

Disruptor 使用了 Unsafe 的内存布局 API 对元素进行了偏移量计算：

```java
//返回数组中第一个元素的偏移地址
public native int arrayBaseOffset(Class<?> arrayClass);
//返回数组中一个元素占用的大小
public native int arrayIndexScale(Class<?> arrayClass);
```

#### 初始化

计算好填充大小 BUFFER_PAD 之后，看看队列的预初始化：

```java
// RingBuffer.java
RingBufferFields(EventFactory<E> eventFactory, Sequencer sequencer) {
		this.sequencer = sequencer;
		this.bufferSize = sequencer.getBufferSize();
    // bufferSize 必须是 2 的 n 次方，方便位运算定位
		if (bufferSize < 1) {
				throw new IllegalArgumentException("bufferSize must not be less than 1");
    } if (Integer.bitCount(bufferSize) != 1) {
				throw new IllegalArgumentException("bufferSize must be a power of 2");
    }
  	// 索引掩码
    this.indexMask = bufferSize - 1;
  	// 环形队列大小，例如我们声明大小为 8 的队列，实际会创建 8 + 64 = 72 大小的队列
  	// 2 * BUFFER_PAD 代表了在队列的头尾对缓存行进行填充
		this.entries = new Object[sequencer.getBufferSize() + 2 * BUFFER_PAD];
  	// 初始化
		fill(eventFactory);
}

// 根据我们定义的元素工厂类，对元素进行实例化
private void fill(EventFactory<E> eventFactory) {
		for (int i = 0; i < bufferSize; i++) {
      	// 从有效缓存行的位置开始初始化
				entries[BUFFER_PAD + i] = eventFactory.newInstance();
		}
}
```

#### 元素定位

接下来，环形数组的元素定位，结合 Unsafe 使用偏移量定位元素：

```java
// 根据元素序列号定位元素位置
protected final E elementAt(long sequence) {
  	// 使用偏移量从内存中获取元素
		return (E) UNSAFE.getObject(entries, REF_ARRAY_BASE + ((sequence & indexMask) << REF_ELEMENT_SHIFT));
}
```

需要注意元素序列号和元素索引的区别，可以理解为元素序列号是元素索引的另一种表现形式。

### 参考

- Disruptor Github：https://github.com/LMAX-Exchange/disruptor
- Disruptor 论文：https://lmax-exchange.github.io/disruptor/disruptor.html
- Java Unsafe 介绍：https://tech.meituan.com/2019/02/14/talk-about-java-magic-class-unsafe.html
- 内存布局介绍：https://www.baeldung.com/java-memory-layout





  

