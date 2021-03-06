# Disruptor 源码解读 

本文对 Disruptor 的运行机制在源码级别上做出解读, 适合对 Disruptor 框架和源码有初步了解的读者。

&nbsp;

Disruptor版本 [3.4.3](https://github.com/LMAX-Exchange/disruptor/tree/3.4.3)

转载请注明来源 https://github.com/lich0079/notes/blob/master/knowledge/disruptor.md
&nbsp;

<img src="img/Disruptor.png?raw=true"  width="900">

&nbsp;


1. [RingBuffer](#RingBuffer)
	* 1.1. [EventFactory<E>](#EventFactoryE)
2. [Sequence](#Sequence)
3. [Sequencer](#Sequencer)
	* 3.1. [SingleProducerSequencer](#SingleProducerSequencer)
	* 3.2. [MultiProducerSequencer](#MultiProducerSequencer)
4. [EventProcessor](#EventProcessor)
	* 4.1. [BatchEventProcessor<T>](#BatchEventProcessorT)
		* 4.1.1. [SequenceBarrier](#SequenceBarrier)
		* 4.1.2. [WaitStrategy](#WaitStrategy)
5. [WorkerPool](#WorkerPool)
	* 5.1. [WorkProcessor](#WorkProcessor)
    
##  1. <a name='RingBuffer'></a>RingBuffer

Disruptor 框架中的核心数据结构，event的容器。内部实现是

```
Object[] entries;
```

创建RingBuffer的时候要给出一个bufferSize，必须是2的次方（数据访问优化，bit mask）。

但实际上在内部会给这个数组开出的空间是 bufferSize + 2*BUFFER_PAD，在我的电脑上这个PAD是32，所以最后的数组其实是这样的

```
entries=[null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null]

例子里event装的是long
```

为什么说它是ring呢，就是因为访问它的数据用的是sequence来定位，sequence是一个只会增加的long类型。

举个例子，给定bufferSize 16， cursor是数组实际的下标，那么对应关系就是如下所示。

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
32.........0
33.........1
34.........2
35.........3
 ..........
```


&nbsp;
###  1.1. <a name='EventFactoryE'></a>EventFactory<E>

EventFactory 是一次性使用的类，在最开始的时候用来给RingBuffer预填充数据。

为了避免JAVA GC带来的性能影响，Disruptor采用的设计是在数组上预填充好对象，每次publish event的时候，只是通过RingBuffer.get(seq)拿到对象，更新对象的值，然后就发布出去了。整个生产消费过程中再也不会有event对象的创建和销毁。

&nbsp;
##  2. <a name='Sequence'></a>Sequence

用来表达event序例号的对象，但这里为什么不直接用long呢 ？

为了高并发下的可见性，肯定不能直接用long的，至少也是volatile long。但Disruptor觉得volatile long还是不够用，所以创造了Sequence类。它的内部实现主要是volatile long，
```
volatile long value;
```
除此以外还支持以下特性：

 * CAS更新
 * order writes (Store/Store barrier，改动不保证立即可见) vs volatile writes (Store/Load barrier，改动保证立即可见)
 * 在 volatile 字段 附近添加 padding 解决伪共享问题

简单理解就是高并发下优化的long类型。 [这里有另一篇解释volatile的文章](volatile.md)

&nbsp;

***在整个框架中可以看到在不同的类里，不同场景下对sequence的表达，有时用long，有时用的Sequence类，这其实是背后对于效率和高并发可见性的考量。***

***比如在对EventProcessor.sequence的更新中都是用的order writes，不保证立即可见，但速度快很多。在这个场景里，造成的结果是显示的消费进度可能比实际上慢，导致生产者有可能在可以生产的情况下没有去生产。但生产者看的是多个消费者中最慢的那个消费进度，所以影响可能没有那么大。***

&nbsp;
##  3. <a name='Sequencer'></a>Sequencer

Sequencer是Disruptor框架的核心类。

生产者发布event的时候首先需要预定一个sequence，Sequencer就是计算和发布sequence的。它主要有2个实现类：SingleProducerSequencer和MultiProducerSequencer。

&nbsp;

&nbsp;

###  3.1. <a name='SingleProducerSequencer'></a>SingleProducerSequencer

生产者每次通过Sequencer.next(n) 来预定下面n个可以写入的数据位，然后修改数据，发布event。

但因为RingBuffer是环形的，一个 size 16的RingBuffer当你拿到Sequence为16时相当于又要去写RingBuffer[0]的位置了，假如之前的数据还没被消费过就会被覆盖了。

Sequencer是这样解决这个问题的，它在内部维护了一个
```
volatile Sequence[] gatingSequences = new Sequence[0];
```
每个消费者会维护一个自己的Sequence对象，来记录自己已经消费到的序例位置。

每添加一个消费者，都会把消费者的Sequence引用添加到gatingSequences中。

通过访问gatingSequences，Sequencer可以得知消费的最慢的消费者消费到了哪个位置。
```
gatingSequences=[7, 8, 9, 10, 3, 4, 5, 6, 11]

8个消费者的例子，最慢的消费完了3，此时可以写seq 19的数据，但不能写seq 20。
```

&nbsp;

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

&nbsp;

*举个例子，gatingSequences=[7, 8, 9, 10, 3, 4, 5, 6, 11]， RingBuffer size 16的情况下，如果算出来的nextSequence是20，wrapPoint是20-16=4， 这时gatingSequences里最小的是3。*

*说明下一个打算写入的位置是wrapPoint 4，但最慢的消费者才消费到3，你不能去覆盖之前4上的数据，这时只能等待，等消费者把之前的4消费掉。*

*为什么wrapPoint = nextSequence - bufferSize，而不是bufferSize的n倍呢，因为消费者只能落后生产者一圈，不然就已经存在数据覆盖了。*

等到SingleProducerSequencer自旋到下一个位置所有人都消费过的时候，它就可以从next方法中返回，生产者拿着sequence就可以继续去发布。

&nbsp;

&nbsp;

###  3.2. <a name='MultiProducerSequencer'></a>MultiProducerSequencer

MultiProducerSequencer 是在多个生产者的场合使用的，多个生产者的情况下存在竞争，导致它的实现更加复杂。

```
int[] availableBuffer;
int indexMask;
int indexShift;

public MultiProducerSequencer(int bufferSize, final WaitStrategy waitStrategy)
{
    super(bufferSize, waitStrategy);
    availableBuffer = new int[bufferSize];
    indexMask = bufferSize - 1;
    indexShift = Util.log2(bufferSize);
    initialiseAvailableBuffer();
}
```

数据结构上多出来的主要就是这个 availableBuffer，用来记录RingBuffer上哪些位置有数据可以读。

&nbsp;

还是从Sequencer.next(n)说起，计算下一个数据位Sequence的逻辑是一样的，包括消费者落后导致Sequencer自旋等待的逻辑。不同的是因为有多个publisher同时访问Sequencer.next(n)方法，所以在确定最终位置的时候用了一个CAS操作，如果失败了就自旋再来一次。
```
cursor.compareAndSet(current, next)
```

另一个不同的地方是 publish(final long sequence) 方法，SingleProducer的版本很简单，就是移动了一下cursor。
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
setAvailable做了什么事呢，它去设置availableBuffer的状态位了。给定一个sequence，先计算出对应的数组下标index，然后计算出在那个index上要写的数据 availabilityFlag，最后执行 
```
availableBuffer[index]=availabilityFlag
```

&nbsp;

根据 calculateAvailabilityFlag(sequence) 方法计算出来的availabilityFlag 实际上是该sequence环绕RingBuffer的圈数。
```
availableBuffer=[6, 6, 6, 6, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5]

例子：前4个已经走到第6圈。
```

&nbsp;

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
在单个生产者的场景下，publishEvent的时候才会推进cursor，所以只要sequence<=cursor，就说明数据是可消费的。

&nbsp;

多个生产者的场景下，在next(n)方法中，就已经通过 cursor.compareAndSet(current, next) 移动cursor了，此时event还没有publish，所以cursor所在的位置不能保证event一定可用。

在publish方法中是去setAvailable(sequence)了，所以availableBuffer是数据是否可用的标志。那为什么值要写成圈数呢，应该是避免把上一轮的数据当成这一轮的数据，错误判断sequence是否可用。

&nbsp;

另一个值得一提的是getHighestPublishedSequence方法，这个是消费者用来查询最高可用event数据的位置。
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

以上就是2种生产者的介绍。

为了最佳的系统性能，如果业务允许的话最好遵守 [Single Writer Principle](https://mechanical-sympathy.blogspot.co.nz/2011/09/single-writer-principle.html)， 采用SingleProducer。在官方的[测试](https://lmax-exchange.github.io/disruptor/user-guide/index.html#_getting_started)中，SingleProducer的吞吐量大概是MultipleProducer的2-3倍。

&nbsp;

&nbsp;

说完了生产者，下面来看看消费者

&nbsp;
##  4. <a name='EventProcessor'></a>EventProcessor

EventProcessor extends Runnable, 可以理解它是一个消费者线程。

EventProcessor 本身也只是个interface。

&nbsp;
###  4.1. <a name='BatchEventProcessorT'></a>BatchEventProcessor<T>

主要属性有
```
DataProvider<T> dataProvider;   // 就是RingBuffer， event容器
SequenceBarrier sequenceBarrier; // 用来获取可用event的sequence
EventHandler<? super T> eventHandler;  // 真正消费event的业务代码
Sequence sequence = new Sequence(-1);      // 该消费线程消费完的sequence位置
```

&nbsp;

####  4.1.1. <a name='SequenceBarrier'></a>SequenceBarrier

ProcessingSequenceBarrier 内部持有Sequencer的cursor引用，知道生产者生产到哪个位置了。BatchEventProcessor.sequence 是当前消费线程消费到的位置。sequence + 1 就是下一个打算消费的位置 nextSequence，sequenceBarrier.waitFor(nextSequence) 会去获取下一个可以消费的availableSequence。

拿到的availableSequence可能比要求的nextSequence大，意味着生产者生产出了很多可以消费的event。这时就会一个个去消费，并且更新BatchEventProcessor的sequence至availableSequence。此时Sequencer上的gatingSequences因为是引用的关系也会被更新。

&nbsp;

####  4.1.2. <a name='WaitStrategy'></a>WaitStrategy

调用 sequenceBarrier.waitFor(nextSequence) 时可能不会立即有新的event，这时的行为由 waitStrategy 决定，有多种实现，比如 BlockingWaitStrategy。 

Sequencer在构造的时候就会传入一个 waitStrategy，sequenceBarrier 是由 Sequencer 创建的，创建的时候把 Sequencer 的 waitStrategy 传递给 sequenceBarrier。Sequencer和SequenceBarrier持有同样的waitStrategy，相当于在两者间起到了 ***传递信息和回调*** 的作用。

消费者在没有可消费的event时会调用waitStrategy.waitFor陷入等待，生产者会在生产出新event后调用waitStrategy.signalAllWhenBlocking来唤醒消费者。

&nbsp;

不同的 WaitStrategy 的实现会有不同的效率和性能。

#### BlockingWaitStrategy

该实现依赖Lock来设置等待和唤醒。 系统吞吐量和低延迟的表现比较差，好处是对CPU的消耗比较少。

```
Lock lock = new ReentrantLock();
Condition processorNotifyCondition = lock.newCondition();
```

```
processorNotifyCondition.await();

等待
```

```
processorNotifyCondition.signalAll();

唤醒
```
&nbsp;
#### SleepingWaitStrategy

该实现是在性能和CPU占用之间的一种折中。

```
inal int DEFAULT_RETRIES = 200;
long DEFAULT_SLEEP = 100;

int retries;
long sleepTimeNs;
```


```
if (counter > 100)
{
    --counter;
}
else if (counter > 0)
{
    --counter;
    Thread.yield();
}
else
{
    LockSupport.parkNanos(sleepTimeNs);
}

等待的实现，上面的counter就是retries。
```

该实现对负责调用唤醒方法的生产者比较友好，因为啥都不用做。相当于完全依赖消费者端的自旋retry。
```
public void signalAllWhenBlocking()
{
}
```

&nbsp;

#### YieldingWaitStrategy

该实现和SleepingWaitStrategy很类似，只是它在等待的时候会吃掉100%的CPU。
```
if (0 == counter)
{
    Thread.yield();
}
else
{
    --counter;
}

等待的实现， 只有 counter==0 的时候才让出CPU，其他时候都在自旋。
```
和SleepingWaitStrategy一样，唤醒的时候啥都不用做。
```
public void signalAllWhenBlocking()
{
}
```

&nbsp;

#### BusySpinWaitStrategy

该实现的唤醒也是啥都不做。性能最好的实现，但对部署环境的要求也最高。消费者线程数应该要少于CPU的实际物理核心数。
```
ThreadHints.onSpinWait();

等待的实现。
```


&nbsp;

&nbsp;

上面介绍的是通过 EventProcessor，EventHandler<T> 来消费event的场景，每一组 EventProcessor + EventHandler 是一个独立的消费者，会消费全量的event数据。如果你通过 
```
handleEventsWith(final EventHandler<? super T>... handlers)
```
接口添加了10个EventHandler，那么event就会被消费10遍。

&nbsp;

&nbsp;

Disruptor 还支持一种多个线程共同消费event的模式。相关的类就是 WorkerPool，WorkProcessor，WorkHandler。

&nbsp;

##  5. <a name='WorkerPool'></a>WorkerPool

```
Sequence workSequence = new Sequence(-1);
WorkProcessor<?>[] workProcessors
```

WorkerPool 内部维护了一个workSequence，代表着整个pool分配出去的event位置。<=workSequence的event已经被分配给某个workProcessors了，但是不是一定已经被消费完。这个设计和多生产者的情况下，先分配sequence到具体的某个生产者，然后再去填充，提交是一样的道理。

&nbsp;

###  5.1. <a name='WorkProcessor'></a>WorkProcessor

WorkProcessor 是基本的消费者线程，它保有workSequence的引用。

在它的run loop中，它会首先尝试CAS去抢workSequence的下一个位置，抢到了就会去消费。

如果没有可消费的event了，它就会调用 sequenceBarrier.waitFor(nextSequence) 陷入等待。但即使有了新的event被唤醒，它还是要靠CAS去抢下一个event的消费权。


```
while (true)
{
    try
    {
        // if previous sequence was processed - fetch the next sequence and set
        // that we have successfully processed the previous sequence
        // typically, this will be true
        // this prevents the sequence getting too far forward if an exception
        // is thrown from the WorkHandler
        if (processedSequence)   // 这个if里面的代码都是为了CAS拿event
        {
            processedSequence = false;
            do
            {
                nextSequence = workSequence.get() + 1L; // 拿到下一个sequence
                sequence.set(nextSequence - 1L); 
                // 更新这个WorkProcessor的消费位置，这个位置主要是反映到Sequencer的gatingSequence从而影响生产者是否继续生产。
                // 但实际上（nextSequence - 1L）这个位置很有可能不是这个WorkProcessor消费掉的
            }
            while (!workSequence.compareAndSet(nextSequence - 1L, nextSequence));
        }

        if (cachedAvailableSequence >= nextSequence) // 如果该nextSequence已经被生产出来
        {
            event = ringBuffer.get(nextSequence);
            workHandler.onEvent(event);
            processedSequence = true;
        }
        else
        {
            cachedAvailableSequence = sequenceBarrier.waitFor(nextSequence);  // 没有被生产出来就在这等待
        }
    }
    // exception handler
}
```


## 一些思考

从Disruptor的实现可以看到因为采用环形数据结构，它会覆盖老数据，生产新数据必须等到老数据都被消费后，生产速度和消费速度是紧耦合的，一旦消费者们中有一个人慢了一点、卡顿、异常，会影响整个系统的生产以至消费。

唯一可以缓冲的就是Buffer，即数组长度bufferSize。

比如，官方文档就建议生产event的时候使用 EventTranslator API。因为外层有 try finally 兜底。

```
    private void translateAndPublish(EventTranslator<E> translator, long sequence)
    {
        try
        {
            translator.translateTo(get(sequence), sequence);
        }
        finally
        {
            sequencer.publish(sequence);
        }
    }
```


&nbsp;

## 2021.6 更新

*定义Event model的时候可以加个long seq 字段，把当时disruptor分配的sequence记住，方便以后debug*


&nbsp;

## Exception Handle

&nbsp;

### Producer

如果可能在生产的时候出异常，必须在publishEvent 方法外面包一层 try catch, 否则会导致程序崩坏


```
// translateTo 方法中会出异常
public static EventTranslatorTwoArg<StubEvent, String, Integer> eventTranslator = new EventTranslatorTwoArg<StubEvent, String, Integer>() {
 
    @Override
    public void translateTo(StubEvent event, long sequence, String arg0, Integer arg1) {
        event.setSeq(sequence);
        if (sequence % 3 == 0) {
            throw new RuntimeException("translateTo exception," + arg0 + ",arg1=" + arg1 + ",seq=" + sequence);
        }
        event.setMsg(arg0);
        event.setValue(arg1);
    }
};
 
 
 
//  publishEvent 需要 try catch
for (long i = 0; i < count ; i++) {
    try {
        disruptor.publishEvent(StubEvent.eventTranslator, "msg " + i, r.nextInt(100));
    } catch (Exception e) {
        log.error("publishEvent exception", e);
    }
}

```

有一点需要注意的是 如果在生产的时候出了异常，导致event 属性的填充出了问题，你又没有clean 之前的属性的话， 因为是环形数据结构，消费者有可能消费上一次的属性。

```
比如 下面的log中， seq 6 的 event 在生产阶段出错了，消费阶段消费的是上一次的属性值



java.lang.RuntimeException: translateTo exception,msg 6,value=2,seq=6    // 生产时  seq 6 的event, value 是 2



journal StubEvent(value=84, msg=msg 2, seq=6)      // 消费seq 6的时候，  value 是 84， 其实是上一次的
```

所以最好在event上设置一个flag（比如： produceCompleted），放在 translateTo 方法的最后一行执行。 这样消费者可以在消费的时候检查这个flag，防止误把以前的event消费2次。 


```
// 生产者在开始生产的时候需要先把flag 设为false， 在最后一行设为true
@Override
public void translateTo(StubEvent event, long sequence, String arg0, Integer arg1) {
    event.setProduceCompleted(false);
    event.setSeq(sequence);
 
    if (sequence % 3 == 0) {
        throw new RuntimeException("translateTo exception, arg0=" + arg0 + ",arg1=" + arg1 + ",seq=" + sequence);
    }
 
    event.setMsg(arg0);
    event.setValue(arg1);
 
    event.setProduceCompleted(true);
}
```

&nbsp;

### Consumer

在消费端，可通过  
```
disruptor.setDefaultExceptionHandler
```
方法设定异常处理， 如果没调用过这个方法的话，默认实现是  FatalExceptionHandler, log并抛出异常， 会导致整个disruptor崩坏。
```
private ExceptionHandler<? super T> getExceptionHandler()
{
    ExceptionHandler<? super T> handler = exceptionHandler;
    if (handler == null)
    {
        return ExceptionHandlers.defaultHandler();
    }
    return handler;
}
```

框架提供了一个 **IgnoreExceptionHandler**  只会输出log， 不会抛出。建议采用或自己实现。

&nbsp;

&nbsp;

转载请注明来源 https://github.com/lich0079/notes/blob/master/knowledge/disruptor.md