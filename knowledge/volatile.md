


# 可见性

多核执行多线程的情况下，每个core读取变量不是直接从内存读，而是从L1, L2 ...cache读，所以你在一个core中的write不一定会被其他core马上观测到。

解决这个的办法就是volatile关键字，加上它修饰后，变量在一个core中做了修改，会导致其他core的缓存立即失效，这样就会从内存中读出最新的值，保证了可见性。


# False sharing

volatile 导致其他core缓存失效会带来一个问题。假设2个volatile变量位于一个缓存单元中（cache line，通常64字节），那么一个core一直修改变量A，另一个core在使用变量B，A的不断更新导致在另一个core中处于同一个cache line中的B也会需要一直去内存拉最新的值，即使另一个完全不关心A的值。

解决这个问题的方法就是Padding，把volatile关键字的周围用其他的值来填充，保证volatile不会和其他的volatile共享一个cache line

```

class LhsPadding
{
    protected long p1, p2, p3, p4, p5, p6, p7;
}

class Value extends LhsPadding
{
    protected volatile long value;
}

class RhsPadding extends Value
{
    protected long p9, p10, p11, p12, p13, p14, p15;
}
public class Sequence extends RhsPadding
{
    public long get()
    {
        return value;
    }
}

```

# Lazyset

加了volatile的变量可以保证对它的修改的可见性，但这种保证也是有性能的代价的。那么对一个volatile变量我可不可以有时候不需要它的可见性保证来提升性能呢？

答案就是sun.misc.Unsafe

```

does not guarantee immediate visibility of the store to other threads. This method is generally only useful if the underlying field is a Java volatile (or if an array cell, one that is otherwise only accessed using volatile accesses).

public native void putOrderedLong(Object o, long offset, long x);

```

用法如下， 注意里面对 value的set 是有 volatile 和 non volatile 2个版本，分别针对性能和可见性的场合。

```

package com.lmax.disrupto;

public class Sequence extends RhsPadding
{
    static final long INITIAL_VALUE = -1L;
    private static final Unsafe UNSAFE;
    private static final long VALUE_OFFSET;

    static
    {
        UNSAFE = Util.getUnsafe();
        try
        {
            VALUE_OFFSET = UNSAFE.objectFieldOffset(Value.class.getDeclaredField("value"));
        }
        catch (final Exception e)
        {
            throw new RuntimeException(e);
        }
    }

		/**
     * Perform an ordered write of this sequence.  The intent is
     * a Store/Store barrier between this write and any previous
     * store.
     *
     * @param value The new value for the sequence.
     */
    public void set(final long value)
    {
        UNSAFE.putOrderedLong(this, VALUE_OFFSET, value);
    }

    /**
     * Performs a volatile write of this sequence.  The intent is
     * a Store/Store barrier between this write and any previous
     * write and a Store/Load barrier between this write and any
     * subsequent volatile read.
     *
     * @param value The new value for the sequence.
     */
    public void setVolatile(final long value)
    {
        UNSAFE.putLongVolatile(this, VALUE_OFFSET, value);
    }
}

```


# 性能比较

在 long 自增的测试场景中， 

non-volatile long > volatile long use lasyset(Sequence.set) > Sequence.setVolatile == volatile long with padding > volatile long without padding