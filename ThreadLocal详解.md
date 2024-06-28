`Java`的**四种引用类型**：

- **强引用**：我们常常 new 出来的对象就是强引用类型，只要强引用存在，垃圾回收器将永远不会回收被引用的对象，哪怕内存不足的时候
- **软引用**：使用 SoftReference 修饰的对象被称为软引用，软引用指向的对象在内存要溢出的时候被回收
- **弱引用**：使用 WeakReference 修饰的对象被称为弱引用，只要发生垃圾回收，若这个对象只被弱引用指向，那么就会被回收
- **虚引用**：虚引用是最弱的引用，在 Java 中使用 PhantomReference 进行定义。虚引用中唯一的作用就是用队列接收对象即将死亡的通知

![](https://raw.githubusercontent.com/Aaron-DBJ/ImageRepo/img/20240102164220.png)

### 定义

ThreadLocal是一个在多线程中为每一个线程创建单独的变量副本的类; 当使用ThreadLocal来维护变量时, ThreadLocal会为每个线程创建单独的变量副本, 避免因多线程操作共享变量而导致的数据不一致的情况。它是一种实现线程安全的方式。

### 线程隔离的实现

`Thread`类有一个类型为`ThreadLocal.ThreadLocalMap`的实例变量`threadLocals`，也就是说每个线程有一个自己的`ThreadLocalMap`。

`ThreadLocalMap`有自己的独立实现，可以简单地将它的`key`视作`ThreadLocal`，`value`为代码中放入的值（实际上`key`并不是`ThreadLocal`本身，而是它的一个**弱引用**）。

每个线程在往`ThreadLocal`里放值的时候，都会往自己的`ThreadLocalMap`里存，读也是以`ThreadLocal`作为引用，在自己的`map`里找对应的`key`，从而实现了**线程隔离**。

`ThreadLocalMap`有点类似`HashMap`的结构，只是`HashMap`是由**数组+链表**实现的，而`ThreadLocalMap`中并没有**链表**结构。

### ThreadLocalMap Hash 算法

既然是`Map`结构，那么`ThreadLocalMap`当然也要实现自己的`hash`算法来解决散列表数组冲突问题。

```
int i = key.threadLocalHashCode & (len-1);
```

`ThreadLocalMap`中`hash`算法很简单，这里`i`就是当前 key 在散列表中对应的数组下标位置。

这里最关键的就是`threadLocalHashCode`值的计算，`ThreadLocal`中有一个属性为`HASH_INCREMENT = 0x61c88647`

```java
    private final int threadLocalHashCode = nextHashCode();

    /**
     * The next hash code to be given out. Updated atomically. Starts at
     * zero.
     */
    private static AtomicInteger nextHashCode =
        new AtomicInteger();

    /**
     * The difference between successively generated hash codes - turns
     * implicit sequential thread-local IDs into near-optimally spread
     * multiplicative hash values for power-of-two-sized tables.
     */
    private static final int HASH_INCREMENT = 0x61c88647;

    /**
     * Returns the next hash code.
     */
    private static int nextHashCode() {
        return nextHashCode.getAndAdd(HASH_INCREMENT);
    }
    /**
         * Construct a new map initially containing (firstKey, firstValue).
         * ThreadLocalMaps are constructed lazily, so we only create
         * one when we have at least one entry to put in it.
         */
        ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
            table = new Entry[INITIAL_CAPACITY];
            int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
            table[i] = new Entry(firstKey, firstValue);
            size = 1;
            setThreshold(INITIAL_CAPACITY);
        }
```

每当创建一个`ThreadLocal`对象，这个`ThreadLocal.nextHashCode` 这个值就会增长 `0x61c88647` 。

这个值很特殊，它是**斐波那契数** 也叫 **黄金分割数**。`hash`增量为 这个数字，带来的好处就是 `hash` **分布非常均匀**。

### ThreadLocalMap Hash冲突

虽然`ThreadLocalMap`中使用了**黄金分割数**来作为`hash`计算因子，大大减少了`Hash`冲突的概率，但是仍然会存在冲突。`HashMap`中解决冲突的方法是在数组上构造一个**链表**结构，冲突的数据挂载到链表上，如果链表长度超过一定数量则会转化成**红黑树**。而 `ThreadLocalMap` 中并没有链表结构，所以这里不能使用 `HashMap` 解决冲突的方式了

`ThreadLocalMap` 中哈希冲突的**解决方式是开放地址法-线性探测，遇到冲突后向后查找，直到遇到为Entry为null的槽位，然后将数据储存。** 具体场景分为：

- entry为null：将当前元素直接存放在该位置

- entry不为null，且key相同：更新当前位置的数据value值

- 找到entry为null的槽位前，找到了key为null的槽位：此时就会执行`replaceStaleEntry()`方法，该方法含义是**替换过期数据的逻辑**，以当前`staleSlot`(指当前数据在数组中的下标)开始向前迭代查找，找其他过期的数据，然后更新过期数据起始扫描下标`slotToExpunge`。`for`循环迭代，直到碰到`Entry`为`null`结束。
  
  接着开始以`staleSlot`位置向后迭代，**如果找到了相同 key 值的 Entry 数据**：找到后更新`Entry`的值并交换`staleSlot`元素的位置(`staleSlot`位置为过期元素)，更新`Entry`数据，然后开始进行过期`Entry`的清理工作。

### ThreadLocalMap过期元素清理

ThreadLocal为避免内存泄露，在存放元素（`set`）、获取元素（`getEntry`）和移除元素时都会清理过期数据（key为null的数组）。

#### 探测式清理 - `expungeStaleEntry`

```java
 private int expungeStaleEntry(int staleSlot) {
            Entry[] tab = table;
            int len = tab.length;

            // expunge entry at staleSlot
            tab[staleSlot].value = null;
            tab[staleSlot] = null;
            size--;

            // Rehash until we encounter null
            Entry e;
            int i;
            for (i = nextIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = nextIndex(i, len)) {
                ThreadLocal<?> k = e.get();
                if (k == null) {
                    e.value = null;
                    tab[i] = null;
                    size--;
                } else {
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

1. 首先清除当前槽位上的`Entry`数据，将其`value`字段和本身都设置为null，数组长度减一。

2. 开始循环，从当前位置的后一个槽位出发，直到遇到第一个为null的槽位。如果遇到`key`为null的槽位，逻辑和操作1一样。如果遇到`key`不为null的槽位，重新计算当前槽位数据在数组中的索引值。（**为什么要重新计算位置：因为Hash冲突的存在，元素存放的槽位可能不是它通过Hash计算出的位置。所以当有过期元素被移除时，重新计算下位置，让其尽可能靠近原始位置，提升查找性能**）如果重新计算的位置和当前位置的索引不一样，说明当前元素是因为Hash冲突被移到当前位置的，所以需要重新计算本来的槽位位置存放，如果还有冲突则向后遍历寻找一个空槽位存放。

#### 启发式清理 - `cleanSomeSlots`

```java
private boolean cleanSomeSlots(int i, int n) {
            boolean removed = false;
            Entry[] tab = table;
            int len = tab.length;
            do {
                i = nextIndex(i, len);
                Entry e = tab[i];
                if (e != null && e.get() == null) {
                    n = len;
                    removed = true;
                    i = expungeStaleEntry(i);
                }
            } while ( (n >>>= 1) != 0);
            return removed;
        }
```

这种方式会尝试log2N次，如果遇到`key`为null的槽位，则会使用探测式清理方式。

### ThreadLocal内存泄露

`ThreadLocalMap`内部维护了一个`Entry[] table`来存储键值对的映射关系，内存泄漏和`Entry`类有非常大的关系，下面是`Entry`的源码：

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

`Entry`将`ThreadLocal`作为Key，值作为value保存，它继承自`WeakReference`，注意构造函数里的第一行代码`super(k)`，这意味着ThreadLocal对象是一个「弱引用」

在实际使用中，ThreadLocal通常存在如下的引用关系：

![](https://raw.githubusercontent.com/Aaron-DBJ/ImageRepo/img/20240204160116.png)

图中虚线表示弱引用，GC后会断开

可以看到，key为弱引用虽然可以在GC后解除引用连接，但是如果Thread长期存活的话，依旧会存在这条链路：Thread Ref → Current Thread → ThreadLocalMap → Entry → Value，value由于是强引用不会被回收，就会导致内存泄露。

ThreadLocal也考虑打了这个问题，在执行 ThreadLocal 的 `set`、`remove`、`rehash` 等方法时，它都会扫描 key 为 null 的 Entry，如果发现某个 Entry 的 key 为 null，则代表它所对应的 value 也没有作用了，所以它就会把对应的 value 置为 null，这样，value 对象就可以被正常回收了。

但是假设 ThreadLocal 已经不被使用了，那么实际上 `set`、`remove`、`rehash` 方法也不会被调用，与此同时，如果这个线程又一直存活、不终止的话，那么刚才的那个调用链就一直存在，也就导致了 value 的内存泄漏。

**推荐做法**：在ThreadLocal使用完成后，手动调用`remove`方法将value置为null，方便后续回收。

### InheritableThreadLocal

我们使用`ThreadLocal`的时候，在异步场景下是无法给子线程共享父线程中创建的线程副本数据的。为了解决这个问题，JDK 中还有一个`InheritableThreadLocal`类

实现原理是子线程是通过在父线程中通过调用`new Thread()`方法来创建子线程，`Thread#init`方法在`Thread`的构造方法中被调用。在`init`方法中拷贝父线程数据到子线程中
