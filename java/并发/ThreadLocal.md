# ThreadLocal数据结构

![ThreadLocal structure](../img/1648364800(1).jpg)

每个线程有且仅有一个独立的 ThreadLocalMap。即如果使用了多个 ThreadLocal 实例，也只是对应 Entry 数组的不同 Entry。

```java
public class Thread implements Runnable{
    ThreadLocal.ThreadLocalMap threadLocals = null;
}

public class ThreadLocal<T> {
    
    static class ThreadLocalMap {
        
        static class Entry extends WeakReference<ThreadLocal<?>> {
            Object value;
        }
        
        private static final int INITIAL_CAPACITY = 16;

        private Entry[] table; // MUST always be a power of two.
        
        private int size = 0;

        private int threshold; // Default to 0
    }
    
}
```



```java
    // class: ThreadLocal
	public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t); // -> return t.threadLocals;
        if (map != null) {
            ...
        } else {
            createMap(t, value); // -> t.threadLocals = new ThreadLocalMap(this, firstValue);
        }
    }
    
	// class: ThreadLocalMap
    ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
        table = new Entry[INITIAL_CAPACITY];
        int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
        table[i] = new Entry(firstKey, firstValue);
        size = 1;
        setThreshold(INITIAL_CAPACITY);
    }     
```

> `Thread`类有一个类型为`ThreadLocal.ThreadLocalMap`的实例变量`threadLocals`，也就是说每个线程有一个自己的`ThreadLocalMap`。
>
> `ThreadLocalMap`有自己的独立实现，可以简单地将它的`key`视作`ThreadLocal`，`value`为代码中放入的值（实际上`key`并不是`ThreadLocal`本身，而是它的一个**弱引用**）。
>
> 每个线程在往`ThreadLocal`里放值的时候，都会往自己的`ThreadLocalMap`里存，读也是以`ThreadLocal`作为引用，在自己的`map`里找对应的`key`，从而实现了**线程隔离**。
>
> `ThreadLocalMap`有点类似`HashMap`的结构，只是`HashMap`是由**数组+链表**实现的，而`ThreadLocalMap`中并没有**链表**结构。
>
> 我们还要注意`Entry`， 它的`key`是`ThreadLocal<?> k` ，继承自`WeakReference`， 也就是我们常说的弱引用类型。

# Hash算法

也就是说每当创建一个 `ThreadLocal` 对象，这个 `ThreadLocal.nextHashCode`  这个值就会增长 `0x61c88647` 。

> 这个值很特殊，它是**斐波那契数** 也叫 **黄金分割数**。`hash`增量为 这个数字，带来的好处就是 `hash` **分布非常均匀**。

```java
    # class: ThreadLocal
    private final int threadLocalHashCode = nextHashCode();
    private static int nextHashCode() {
        return nextHashCode.getAndAdd(HASH_INCREMENT);
    }

    # ThreadLocalMap#set
    private void set(ThreadLocal<?> key, Object value) {
    	int i = key.threadLocalHashCode & (len-1);
    }
```

# set：Hash冲突

ThreadLocalMap 没有使用类似 HashMap 的结构（链表/红黑树），所以解决 Hash 冲突的方法也与 HashMap 不同。

当 ThreadLocalMap 发现 Hash 冲突时，就会线性向后查找，一直找到 `Entry` 为 `null` 的槽位才会停止查找，将当前元素放入此槽位中。在这个过程中还可能遇到 Entry 不为 null 但 key 为 null的情况（即 Entry 过期），所以在这个过程中还会进行**探测式清理**（expungeStaleEntry）或者**启发式清理**（cleanSomeSlots）

```java
    # ThreadLocalMap#set
     private void set(ThreadLocal<?> key, Object value) {
            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);
            
            // 理想情况没有 Hash 冲突时，e == null, 不会进入 for 循环。 nextIndex 线性向后查找
            for (Entry e = tab[i]; e != null; e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();
                
                // key相等，相当于更新
                if (k == key) {
                    e.value = value;
                    return;
                }
                
                // i为过期槽位下标，并且可能为该 key 的目标下标，进行替换清理
                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }

            tab[i] = new Entry(key, value);
            int sz = ++size;
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
    }

    // staleSlot为过期槽位下标，并且可能为该 key 的目标下标
    private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                                   int staleSlot) {
        Entry[] tab = table;
        int len = tab.length;
        Entry e;

        // slotToExpunge的值含义为：表示开始探测式清理过期数据的开始下标，初始化为 stateSlot
        int slotToExpunge = staleSlot;
        // 向前迭代，尝试找出更前面的过期槽位，但是遇到 null 会中止
        for (int i = prevIndex(staleSlot, len); (e = tab[i]) != null; i = prevIndex(i, len))
            if (e.get() == null)
                slotToExpunge = i;

        // 向后查找    疑问： tab[i] == null 终止，有没有可能出现 k == key 的值在 null 后面？
        for (int i = nextIndex(staleSlot, len); (e = tab[i]) != null; i = nextIndex(i, len)) {
            ThreadLocal<?> k = e.get();

            // key 相等，更新 Entry 的值并交换 staleSlot 元素的位置，然后开始进行过期Entry的清理工作
            if (k == key) {
                e.value = value;

                tab[i] = tab[staleSlot];
                tab[staleSlot] = e;

                // 说明 一开始的向前迭代并未找到过期的 Entry（上一个for），接着向后查找过程中也未发现过期数据（当前for）
                // 则 修改开始探测式清理过期数据的下标为当前循环的 index
                if (slotToExpunge == staleSlot)
                    slotToExpunge = i;
                cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);  //启发式清理（cleanSomeSlots） 探测式清理（expungeStaleEntry）
                return;
            }

            // 当前为过期数据 && 一开始的向前迭代并未找到过期的 Entry（上一个for）
            // 则 修改开始探测式清理过期数据的下标为当前循环的 index
            if (k == null && slotToExpunge == staleSlot)
                slotToExpunge = i;
        }

        // 没有找到 key， 说明这里是一个添加的逻辑， 直接放到 state slot
        tab[staleSlot].value = null;
        tab[staleSlot] = new Entry(key, value);

        // 如果除了staleSlot以外，还发现了其他过期的slot数据，开启清理数据的逻辑
        if (slotToExpunge != staleSlot)
            cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
    }
```

> 1. 遍历当前`key`值对应的桶中`Entry`数据为空，这说明散列数组这里没有数据冲突，跳出`for`循环，直接`set`数据到对应的桶中
> 2. 如果`key`值对应的桶中`Entry`数据不为空
>    2.1 如果`k = key`，说明当前`set`操作是一个替换操作，做替换逻辑，直接返回
>    2.2 如果`key = null`，说明当前桶位置的`Entry`是过期数据，执行`replaceStaleEntry()`方法(核心方法)，然后返回
> 3. `for`循环执行完毕，继续往下执行说明向后迭代的过程中遇到了`entry`为`null`的情况
>    3.1 在`Entry`为`null`的桶中创建一个新的`Entry`对象
>    3.2 执行`++size`操作
> 4. 调用`cleanSomeSlots()`做一次启发式清理工作，清理散列数组中`Entry`的`key`过期的数据
>    4.1 如果清理工作完成后，未清理到任何数据，且`size`超过了阈值(数组长度的 2/3)，进行`rehash()`操作
>    4.2 `rehash()`中会先进行一轮探测式清理，清理过期`key`，清理完成后如果**size >= threshold - threshold / 4**，就会执行真正的扩容逻辑

# 探测式清理（expungeStaleEntry）

> 遍历散列数组，从开始位置向后探测清理过期数据，将过期数据的`Entry`设置为`null`，沿途中碰到未过期的数据则将此数据`rehash`后重新在`table`数组中定位，如果定位的位置已经有了数据，则会将未过期的数据放到最靠近此位置的`Entry=null`的桶中，使`rehash`后的`Entry`数据距离正确的桶的位置更近一些。
>
> 在往后迭代的过程中碰到空的槽位，终止探测。
>
> 经过一轮探测式清理后，`key`过期的数据会被清理掉，没过期的数据经过`rehash`重定位后所处的桶位置理论上更接近`i= key.hashCode & (tab.len - 1)`的位置。这种优化会提高整个散列表查询性能。

```java
    // param:  已知具有空键的槽的索引
	// return: staleSlot 之后的下一个空槽的索引（所有在 staleSlot 和这个槽之间的都将被检查是否被删除）
	private int expungeStaleEntry(int staleSlot) {
                Entry[] tab = table;
                int len = tab.length;

                // 删除过期条目               expunge entry at staleSlot
                tab[staleSlot].value = null;
                tab[staleSlot] = null;
                size--;

                // 重新散列，直到遇到null      Rehash until we encounter null
                Entry e;
                int i;
                // 向后探测，删除过期数据，未过期的数据进行 Rehash 使 Entry 数据距离正确的桶的位置更近一些。
                for (i = nextIndex(staleSlot, len); (e = tab[i]) != null; i = nextIndex(i, len)) {
                    ThreadLocal<?> k = e.get();
                    // 删除过期数据
                    if (k == null) {
                        e.value = null;
                        tab[i] = null;
                        size--;
                    } else {
                        //Rehash
                        int h = k.threadLocalHashCode & (len - 1);
                        if (h != i) {
                            tab[i] = null;

                            // Unlike Knuth 6.4 Algorithm R, we must scan until
                            // null because multiple entries could have been stale.
                            while (tab[h] != null)
                                h = nextIndex(h, len);
                            tab[h] = e;
                        }
                    }
                }
                return i;
     }
```

# 启发式清理（cleanSomeSlots）

![ThreadLocal cleanSomeSlots](../img/0b6ad344656486cec5617226f07f9bd.png)

```java
    // class: ThreadLocalMap
	private boolean cleanSomeSlots(int i, int n) {
        // return: 是否进行了清理/是否发现了过期数据
        boolean removed = false;
        Entry[] tab = table;
        int len = tab.length;
        do {
            // i线性向后
            i = nextIndex(i, len);
            Entry e = tab[i];
            // 发现过期数据了才进行清理
            if (e != null && e.get() == null) {
                n = len;
                removed = true;
                i = expungeStaleEntry(i);
            }
        } while ( (n >>>= 1) != 0); // i只向后log2n次
        return removed;
    }
```



# 扩容

> 在`ThreadLocalMap.set()`方法的最后，如果执行完启发式清理工作后，未清理到任何数据，且当前散列数组中`Entry`的数量已经达到了列表的扩容阈值`(len*2/3)`，就开始执行`rehash()`逻辑：
>
> rehash 会进行探测式清理工作，从`table`的起始位置往后清理，上面有分析清理的详细流程。清理完成之后，`table`中可能有一些`key`为`null`的`Entry`数据被清理掉，所以此时通过判断`size >= threshold - threshold / 4` 也就是`size >= threshold * 3/4` 来决定是否扩容。

```java
    // class: ThreadLocalMap
	private void set(ThreadLocal<?> key, Object value) {
        ...
        if (!cleanSomeSlots(i, sz) && sz >= threshold)  // 2/3
            rehash();
    }

	// class: ThreadLocalMap
    private void rehash() {
        expungeStaleEntries();

        if (size >= threshold - threshold / 4)   // 3/4
            resize();
    }
```

> resize():  扩容后的`tab`的大小为`oldLen * 2`，然后遍历老的散列表，重新计算`hash`位置，然后放到新的`tab`数组中，如果出现`hash`冲突则往后寻找最近的`entry`为`null`的槽位，遍历完成之后，`oldTab`中所有的`entry`数据都已经放入到新的`tab`中了。重新计算`tab`下次扩容的**阈值 ** 

resize的代码比较简单就不放出来了，看上面的描述就可以了。

# get

上面的 Hash 冲突以及 探测式清理基本把 set 的流程都讲完了，下面是 get 的流程，比较简单。

> 通过查找`key`值计算出散列表中`slot`位置，然后该`slot`位置中的`Entry.key`和查找的`key`一致，则直接返回。
>
>  如果`slot`位置中的`Entry.key`和要查找的`key`不一致，则向后迭代，如果遇到`Entry[i] != null && Entry[i].key == null`，则触发一次探测式数据回收操作expungeStaleEntry(i)，然后再向后继续迭代

```java
    // class: ThreadLocal
	public T get() {
        ...
		ThreadLocalMap.Entry e = map.getEntry(this);
        ...
    }

    // class: ThreadLocalMap
    private Entry getEntry(ThreadLocal<?> key) {
        int i = key.threadLocalHashCode & (table.length - 1);
        Entry e = table[i];
        // 直接命中，返回
        if (e != null && e.get() == key)
            return e;
        else
            return getEntryAfterMiss(key, i, e); // 向后迭代
    }

    // class: ThreadLocalMap
    private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
        Entry[] tab = table;
        int len = tab.length;
        // 向后迭代
        while (e != null) {
            ThreadLocal<?> k = e.get();
            if (k == key)
                return e;
            if (k == null)
                expungeStaleEntry(i); // 探测式清理
            else
                i = nextIndex(i, len);
            e = tab[i];
        }
        return null;
    }

```

# InheritableThreadLocal

> `ThreadLocal`的时候，在异步场景下是无法给子线程共享父线程中创建的线程副本数据的。
>
> 为了解决这个问题，JDK 中还有一个`InheritableThreadLocal`类

实现原理是子线程是通过在父线程中通过调用 new Thread() 方法来创建子线程，而在 Thread 的构造方法中判断了如果`inheritThreadLocals`， 则拷贝父线程的 InheritableThreadLocal 到子线程中。

```java
	// 没有用线程池的情况下， 一般 new Thread(() -> doSomeThing())
    // 实际上调用到  this(group, target, name, stackSize, null, true); 
    // 即 	private Thread(...... boolean inheritThreadLocals)
   
    private Thread(...... boolean inheritThreadLocals) {
        if (inheritThreadLocals && parent.inheritableThreadLocals != null)
            this.inheritableThreadLocals =
            ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
    }

```

> 阿里巴巴 TransmittableThreadLocal 组件

# traceId

未确认实际实现方式

> 这里我们使用 `org.slf4j.MDC` 来实现此功能，内部就是通过 `ThreadLocal` 来实现的，具体实现如下：
>
> 当前端发送请求到**服务 A**时，**服务 A**会生成一个类似`UUID`的`traceId`字符串，将此字符串放入当前线程的`ThreadLocal`中，在调用**服务 B**的时候，将`traceId`写入到请求的`Header`中，**服务 B**在接收请求时会先判断请求的`Header`中是否有`traceId`，如果存在则写入自己线程的`ThreadLocal`中。
