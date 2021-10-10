Disruptor 有单生产者与多生产者两种不同的生产方式，单生产者的实现相对简单，且生产效率比多生产者高，首先看看单生产者的源码。

### 生产者

单生产者的实现主要在 SingleProducerSequencer 实现类中，下面先看看如何申请序号：

```java
// SingleProducerSequencer.java
// 申请生产序号
@Override
public long next(int n) {
		if (n < 1) {
				throw new IllegalArgumentException("n must be > 0");
    }
  	// 上次生产到哪个序号
		long nextValue = this.nextValue;
  	// 本次需要申请的最大生产序号
		long nextSequence = nextValue + n;
  	// 用于检查上一轮的生产的元素是否被消费，相对位置的第一位，为了方便，就称为包装点（上一轮最小的生产点序号）
		long wrapPoint = nextSequence - bufferSize;
  	// 记录消费者上次最小的消费序号
    long cachedGatingSequence = this.cachedValue;
  	// 包装点大于消费者上次的消费序号，说明消费者可能还没消费完
		if (wrapPoint > cachedGatingSequence || cachedGatingSequence > nextValue) {
      	// 使用 Unsafe 设置 cursor 变量，cursor 用于记录当前生产的序号，供其他 API 使用
      	// volatile 会产生一个内存屏障，使读写的数据可见
 				cursor.setVolatile(nextValue);  // StoreLoad fence
				long minSequence;
      	// 若包装点大于消费者当前最小的消费序号，则进入等待
				while (wrapPoint > (minSequence = Util.getMinimumSequence(gatingSequences, nextValue))) {
						LockSupport.parkNanos(1L); 
        }
      	// 序号变得可用，更新本次最小的消费序号
				this.cachedValue = minSequence;
		}
  	// 更新最新的生产序号
		this.nextValue = nextSequence;
		return nextSequence;
}

// 发布数据，通知消费者消费
@Override
public void publish(long sequence)
{
  	// 更新当前生产序号
		cursor.set(sequence);
  	// 唤醒消费者
		waitStrategy.signalAllWhenBlocking();
}
```

代码中有几处需要再说明：

第一个是 wrapPoint 变量，这里举个例子来说明。假设环形队列长度为 4（这里已经忽略了缓存行填充），前期，消费者生产的 0，1，2，3 序号都被消费者及时消费，生产 4，5，6，7 时，消费者因为某些原因阻塞，未能及时消费这些序号；根据源码，nextValue = 7，nextSequence = 8，wrapPoint = 4，cachedGatingSequence = 3（因为前面假设，消费者消费到 3）；这时候再尝试生产 8 的时候，满足 wrapPoint > cachedGatingSequence 的条件，便进入 if 逻辑中，若消费者仍然没有消费并更新 gatingSequences，则进入等待；

第二个是 cursor.setVolatile(nextValue) 的作用，方法内部使用了 Unsafe 动态的给 cursor 加上 volatile 修饰符。为什么不直接在声明变量时加入 voltile 呢？主要是出于性能考虑，这两者的区别后续再做进一步探究；

第三个是 gatingSequences，这个变量表示消费者的当前的消费序号，消费者在获取可消费的序号时，会依赖该消费序号进行消费 - 更新；

### 消费者

对应的，消费者如何拿到有效的序列号进行消费，消费者的实现都在 BatchEventProcessor 中：

```java
// BatchEventProcessor.java
private void processEvents() {
		T event = null;
  	// sequence 初始值为 -1，标识当前消费者消费到的序号，也就是上文所说的 gatingSequences
		long nextSequence = sequence.get() + 1L;
		while (true) {
				try {
          	// 获取可用的消费序号，不同的生产者有不同的实现逻辑，见下面分析
						final long availableSequence = sequenceBarrier.waitFor(nextSequence);
						if (batchStartAware != null) {
            		batchStartAware.onBatchStart(availableSequence - nextSequence + 1);
            }
          	// 消费序号小于生产序号
						while (nextSequence <= availableSequence) {
              	// 从环形队列中取出元素
            		event = dataProvider.get(nextSequence);
              	// 调用消费者逻辑消费
              	// nextSequence == availableSequence 标识当前是否是最后一个可消费的序列，可用于批量处理
                eventHandler.onEvent(event, nextSequence, nextSequence == availableSequence);
              	// 为取出下一个消费序号准备  
              	nextSequence++;
            }
          	// 本轮消费结束，设置当前消费序号
						sequence.set(availableSequence);
        } catch (final TimeoutException e) {
            notifyTimeout(sequence.get());
        } catch (final AlertException ex) {
            if (running.get() != RUNNING) {
                break;
            }
        } catch (final Throwable ex) {
            handleEventException(ex, nextSequence, event);
            sequence.set(nextSequence);
            nextSequence++;
        }
    }
}
```

继续看看代码中所调用的 sequenceBarrier.waitFor(nextSequence) 逻辑：

```java
@Override
public long waitFor(final long sequence) throws AlertException, InterruptedException, TimeoutException {
		checkAlert();
  	// 根据等待机制，获取可用的消费序号，cursorSequence 在 publish 中被更新
		long availableSequence = waitStrategy.waitFor(sequence, cursorSequence, dependentSequence, this);
		// 若消费序号小于传入的 sequence，则返回
  	if (availableSequence < sequence) {
				return availableSequence;
		}
  	// 否则根据生产者的实现，返回最高的序号，单生产者直接返回 availableSequence
		return sequencer.getHighestPublishedSequence(sequence, availableSequence);
}
```

理解各个 sequence 的含义，上面几段逻辑会清晰许多。