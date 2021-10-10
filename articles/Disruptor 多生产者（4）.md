### 生产者

多生产者使用 CAS 来保证多线程可正确的并发申请可用序号；不同于单生产者，多生产者引入了一个与队列长度一致的 int 数组（availableBuffer），来标记多生产者的生产情况，并保证消费者逻辑的正确性。

```java
// MultiProducerSequencer.java
// 申请生产序号
@Override
public long next(int n) {
		if (n < 1) {
				throw new IllegalArgumentException("n must be > 0");
		}
		long current;
		long next;
		do {
      	// 以下与单生产者逻辑类似
				current = cursor.get();
        next = current + n;
				long wrapPoint = next - bufferSize;
        long cachedGatingSequence = gatingSequenceCache.get();
				if (wrapPoint > cachedGatingSequence || cachedGatingSequence > current) {
 						long gatingSequence = Util.getMinimumSequence(gatingSequences, current);
						if (wrapPoint > gatingSequence) {
   							LockSupport.parkNanos(1); 
                continue;
            }
						gatingSequenceCache.set(gatingSequence);
        // 通过 CAS 解决多线程并发申请序号的问题
        } else if (cursor.compareAndSet(current, next)) {
            break;
        }
    } while (true);
    return next;
}

// 发布数据，通知消费者消费
@Override
public void publish(final long sequence) {
  	// 更新缓冲数组的值
		setAvailable(sequence);
    waitStrategy.signalAllWhenBlocking();
}
```

对应的 publish 中，调用了 setAvailable 方法来更新缓存数组的值，为消费者的逻辑做准备；

### 消费者

消费者仍然是下面这段逻辑：

```java
@Override
public long waitFor(final long sequence) throws AlertException, InterruptedException, TimeoutException {
		checkAlert();
  	// 根据等待机制，获取可用的消费序号，cursorSequence 在生产者的 CAS 被更新
		long availableSequence = waitStrategy.waitFor(sequence, cursorSequence, dependentSequence, this);
		// 若消费序号小于传入的 sequence，则返回
  	if (availableSequence < sequence) {
				return availableSequence;
		}
  	// 否则根据生产者的实现，返回最高的序号
		return sequencer.getHighestPublishedSequence(sequence, availableSequence);
}
```

这里举个例子会比较好理解，多生产者的情况下，生产序号很快就申请到比较高的值（availableSequence 值很高），但实际上这只是完成了序号的分配，生产者并没有把数据写入到对应的元素中，此时消费者是不能消费的，还需要额外的逻辑来判断以申请的序号中哪些**连续的序号**是可消费的，我们再到 getHighestPublishedSequence 方法中看看具体逻辑，int 数组（availableBuffer）即将派上用场：

```java
// MultiProducerSequencer.java
@Override
public long getHighestPublishedSequence(long lowerBound, long availableSequence) {
  	// 判断一段连续的、已申请生产的序号中，哪些可用
		for (long sequence = lowerBound; sequence <= availableSequence; sequence++) {
      	// 当前遍历的序号不可用，减 1 后返回
				if (!isAvailable(sequence)) {
            return sequence - 1;
        }
    }
  	// 全部可用，直接返回最高序号
		return availableSequence;
}

// 判断该序号是否已经完成生产
@Override
public boolean isAvailable(long sequence) {
  	// 序号经过转换之后在 availableBuffer 中的索引
		int index = calculateIndex(sequence);
  	// 序号若完成生产之后，会有对应的值，类似标记是第几次循环
		int flag = calculateAvailabilityFlag(sequence);
  	// 计算内存位置，获取当前的 flag
		long bufferAddress = (index * SCALE) + BASE;
  	// 对比两个 flag，若相等，说明该位置完成生产，反之，只是分配了序号，但生产者未完成生产
		return UNSAFE.getIntVolatile(availableBuffer, bufferAddress) == flag;
}
```

再用一张图举个例子说明：

![multi](https://github.com/luckyMax-dev/blog/blob/main/images/disruptor/multi.png)

多生产者引入一个 availableBuffer 数组（类似标记元素的属于第几代），解决了消费者消费未完成生产的元素的问题，这是个巧妙的设计。

