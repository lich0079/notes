# Disruptor 源码解读 

本文适合对Disruptor框架和源码有初步了解的读者。

&nbsp;

Disruptor版本 [3.4.3](https://github.com/LMAX-Exchange/disruptor/tree/3.4.3)

&nbsp;

<img src="img/Disruptor.png?raw=true"  width="900">

&nbsp;

## RingBuffer

Disruptor 框架中的核心数据结构，event的容器。内部实现是

```
protected final Object[] entries;
```

初始化的时候要给出一个bufferSize，必须是2的次方（数据访问优化）。

但实际上在内部会给这个数组开出的实际空间是 bufferSize + 2*BUFFER_PAD，在我的电脑上这个PAD是32，所以最后的数组其实是这样的

```
entries=[null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null]

例子里event装的是long
```

为什么说它是ring呢，就是因为访问它的数据用的是sequence来定位，sequence是一个只会增加的long类型。

举个例子，给定bufferSize 16， cursor是实际数组的下标，那么对应关系就是

```
sequence   cursor
-----------------
0          0
1          1
2          2
3          3
 ..........
15.........15
16.........0
17.........1
18.........2
19.........3
 ..........
```
sequence 就是围绕着RingBuffer转圈。

### EventFactory<E>

eventFactory 是一次性使用的用来给RingBuffer预填充数据。

为了避免JAVA GC带来的性能影响，Disruptor采用的设计是在数组上预填充好对象，每次publish event的时候，只是通过ringBuffer.get(seq)把对象拿下来，更新对象的值，然后就发布出去了。整个过程中再也不会有event对象的创建和销毁。

&nbsp;
## Sequence

用来表达event序例号的对象，但这里为什么不直接用long呢，为了高并发下的可见性，肯定不能直接用long的，至少也是volatile long。但Disruptor觉得volatile long还是不够用，所以创造了Sequence。它的内部主要是volatile long，
```
protected volatile long value;
```
除此以外还支持以下特性：

 * CAS更新
 * order writes (Store/Store barrier) vs volatile writes (Store/Load barrier)
 * adding padding around the volatile field to solve false sharing

简单理解就是高并发下优化的long类型。 [这里有另一篇解释volatile的文章](volatile.md)

&nbsp;
## Sequencer

Sequencer是Disruptor框架的核心类。

publisher想发布event的时候首先需要一个Sequence，Sequencer就是计算和发布Sequence的。它主要有2个实现类：SingleProducerSequencer和MultiProducerSequencer。

&nbsp;

### SingleProducerSequencer

publisher 每次通过Sequencer.next(n) 来获取下面n个可以写入的数据位，然后修改数据，发布event就好了。

但因为RingBuffer是环形的，一个 size 16的RingBuffer当你拿到Sequence为16时相当于又要去写RingBuffer[0]的位置了，假如之前的数据还没被消费过就会被覆盖了。

Sequencer是这样解决这个问题的，它在内部维护了一个
```
protected volatile Sequence[] gatingSequences = new Sequence[0];
```
每个消费者会维护一个自己的Sequence对象，来记录自己已经消费到的序例。

每添加一个消费者，都会把消费者的Sequence引用添加到gatingSequences中。

通过访问gatingSequences，Sequencer可以得知消费的最慢的消费者消费到了哪个位置。
```
8个消费者的例子
gatingSequences=[7, 8, 9, 10, 3, 4, 5, 6, 11]
```

在next(n)方法里，如果计算出的下一个event的Sequence值减去bufferSize
```
long nextValue = this.nextValue;
long nextSequence = nextValue + n;
long wrapPoint = nextSequence - bufferSize;
```
得出来的wrapPoint > min(gatingSequences)，说明即将写入的位置上，之前的event还有消费者没有消费，这时SingleProducerSequencer会等待并自旋。
```
while (wrapPoint > (minSequence = Util.getMinimumSequence(gatingSequences, nextValue)))
    {
        LockSupport.parkNanos(1L);
    }
```

*举个例子，gatingSequences=[7, 8, 9, 10, 3, 4, 5, 6, 11]， RingBuffer size 16的情况下，如果算出来的nextSequence是20，wrapPoint是20-16=4， 这时gatingSequences里最小的是3。*

*说明下一个打算写入的位置是wrapPoint 4，但最慢的消费者才消费到3，你不能去覆盖之前4上的数据，这时只能等待，等消费者把之前的4消费掉。*

*为什么wrapPoint = nextSequence - bufferSize，而不是bufferSize的n倍呢，因为消费者只能落后生产者一圈，不然就已经存在数据覆盖了。*

等到SingleProducerSequencer自旋到下一个位置所有人都消费过的时候，它就可以从next方法中返回，publisher可以继续去发布。

&nbsp;

### MultiProducerSequencer

MultiProducerSequencer 是在多个publisher的场合使用的，多个publisher的情况下存在竞争，导致它的实现更加复杂。

```
    private final int[] availableBuffer;
    private final int indexMask;
    private final int indexShift;

    public MultiProducerSequencer(int bufferSize, final WaitStrategy waitStrategy)
    {
        super(bufferSize, waitStrategy);
        availableBuffer = new int[bufferSize];
        indexMask = bufferSize - 1;
        indexShift = Util.log2(bufferSize);
        initialiseAvailableBuffer();
    }
```

数据结构上主要多出来的就是这个 availableBuffer，用来记录RingBuffer上哪些位置有数据可以读。

&nbsp;

还是从Sequencer.next(n)说起，计算下一个数据位Sequence的逻辑是一样的，包括消费者落后导致Sequencer自旋等待的场合。不同的是因为有多个publisher同时访问Sequencer.next(n)方法，所以在确定最终位置的时候用了一个CAS操作，如果失败了就自旋再来一次。
```
cursor.compareAndSet(current, next)
```

另一个不同的地方是 publish(final long sequence) 方法，SingleProducer的版本很简单，就是move了一下光标。
```
    public void publish(long sequence)
    {
        cursor.set(sequence);
        waitStrategy.signalAllWhenBlocking();
    }
```

MultiProducer的版本则是
```
    public void publish(final long sequence)
    {
        setAvailable(sequence);
        waitStrategy.signalAllWhenBlocking();
    }
```
setAvailable做了什么事呢，它去设置availableBuffer的状态位了。给定一个sequence，先计算出对应的数组下标index，在计算出在那个index上要写的value availabilityFlag，然后availableBuffer[index]=availabilityFlag。

根据 calculateAvailabilityFlag(sequence) 方法计算出来的availabilityFlag 实际上是该sequence环绕RingBuffer的圈数。
```
availableBuffer=[6, 6, 6, 6, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5]

例子：上面说明前4个已经走到第6圈。
```

availableBuffer 主要用于判断一个sequence下的数据是否可用
```
    public boolean isAvailable(long sequence)
    {
        int index = calculateIndex(sequence);
        int flag = calculateAvailabilityFlag(sequence);
        long bufferAddress = (index * SCALE) + BASE;
        return UNSAFE.getIntVolatile(availableBuffer, bufferAddress) == flag;
    }
```
作为比较，来看一下SingleProducer的方法
```
    public boolean isAvailable(long sequence)
    {
        return sequence <= cursor.get();
    }
```
在单个生产者的场景下，publishEvent的时候才会推进cursor，所以只要sequence<=cursor，就说明数据是可用的。

多个生产者的场景下，在next(n)方法中，就已经通过 cursor.compareAndSet(current, next) 移动cursor了，此时event还没有publish，所以cursor所在的位置不能保证event一定可用。在publish方法中是去setAvailable(sequence)了，所以availableBuffer是数据是否可用的标志。那为什么要写成圈数呢，应该是避免把上一轮的数据当成这一轮的数据，错误判断sequence是否可用。

&nbsp;

另一个值得一提的是getHighestPublishedSequence方法，这个是消费者用来查询可用event数据，
```
    public long getHighestPublishedSequence(long lowerBound, long availableSequence)
    {
        for (long sequence = lowerBound; sequence <= availableSequence; sequence++)
        {
            if (!isAvailable(sequence))
            {
                return sequence - 1;
            }
        }

        return availableSequence;
    }
```
给定一段range，依次查询availableBuffer，直到发现不可用的Sequence，那么该Sequence之前的都是可用的。或全部都是可用的。

单生产者的版本:
```
    public long getHighestPublishedSequence(long lowerBound, long availableSequence)
    {
        return availableSequence;
    }
```

&nbsp;

&nbsp;

说完了生产者，下面来看看消费者

&nbsp;
## EventProcessor & BatchEventProcessor<T> 

EventProcessor extends Runnable, 可以理解它是一个消费者线程。

EventProcessor 本身也只是个interface。

&nbsp;
### BatchEventProcessor<T>

主要属性有
```
DataProvider<T> dataProvider;   // 就是RingBuffer， event容器
SequenceBarrier sequenceBarrier; // 获取可用event的sequence
EventHandler<? super T> eventHandler;  // 真正处理event的代码
Sequence sequence = new Sequence(-1);      // 该消费线程消费完的sequence位置
```


ProcessingSequenceBarrier 内部持有Sequencer的cursor引用，知道生产者生产到哪个位置了。BatchEventProcessor.sequence 是当前消费线程消费到的位置。sequence + 1 就是下一个打算消费的位置，sequenceBarrier.waitFor(nextSequence) 会去获取下一个可以消费的availableSequence。

拿到的availableSequence可能比要求的nextSequence大，意味着生产者生产出了很多可以消费的event。这时就会一个个去消费，并且更新BatchEventProcessor的sequence至availableSequence。

调用 sequenceBarrier.waitFor(nextSequence) 时可能不会立即有新的event，这时的行为由 waitStrategy 决定，有多种实现，比如 BlockingWaitStrategy。 

Sequencer在构造的时候就会传入一个 waitStrategy，sequenceBarrier 是由 Sequencer 创建的，创建的时候把 Sequencer 的 waitStrategy 传递给 sequenceBarrier。Sequencer和SequenceBarrier持有同样的waitStrategy，相当于在两者间起到了传递信息和回调的作用。

当消费者调用 sequenceBarrier.waitFor(nextSequence) 时因为没有新event，方法会被 waitStrategy 调用
```
processorNotifyCondition.await();
```
来停止。

这时如果生产者调用Sequencer.publish 会调用 waitStrategy.signalAllWhenBlocking 然后间接调用
```
processorNotifyCondition.signalAll();
```
从而唤醒 sequenceBarrier.waitFor 方法继续执行，看有没有新event可以用。有的话就可以继续消费。没有就再次陷入等待。