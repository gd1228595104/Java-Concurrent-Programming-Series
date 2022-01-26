# ReentrantReadWriteLock

`ReentrantReadWriteLock`是JUC包提供的一个读写锁，在这个类中，维护了一个读锁和一个写锁。
```java
/** Inner class providing readlock */
private final ReentrantReadWriteLock.ReadLock readerLock;
/** Inner class providing writelock */
private final ReentrantReadWriteLock.WriteLock writerLock;
```
这两个锁有什么功能呢？
被读锁锁住的资源，在没有写锁的情况下，可以被多个线程的读锁占有，也就是说读锁是一个共享锁，而写锁与`ReentrantLock`一样，是一个独占锁，被其占有的资源，其他锁不能继续抢占。下面我们可以看一下示例。
```java
public class ReadWriteLockDemo {
    private ReadWriteLock readWriteLock = new ReentrantReadWriteLock();

    private Lock readLock = readWriteLock.readLock();
    private Lock writeLock = readWriteLock.writeLock();

    private List<Object> datas = new ArrayList<>();

    public void add(Object data){
        writeLock.lock();
        try {
            datas.add(data);
        }finally {
            writeLock.unlock();
        }
    }

    public Object get(int index){
        readLock.lock();
        try {
            if (datas.size() == 0 || index < 0 || index > datas.size() - 1){
                return null;
            }
            return datas.get(index);
        }finally {
            readLock.unlock();
        }
    }
}
```
示例中提供了`add(Object data)`方法和`get(int index)`方法，分别用来添加值和获取值，我们都知道`ArrayList`不是线程安全的，所以分别在两个方法里使用了`writeLock`和`readLock`两个锁，使用读写锁与使用重入锁，效果上有什么不同呢？
如果大家看过之前介绍`ReentrantLock`的文章，便可以想到，如果`add`方法和`get`方法使用重入锁，那么两个方法同一时刻只能有一个方法执行，并且同一时刻多线程下执行`get`方法，也会阻塞，这种方式极大影响程序的性能。而`ReentrantReadWriteLock`能允许多个线程同时调用`get`方法，但是只要有任何一个线程在写，其他线程不管访问哪个方法都必须阻塞，这种方式在读多写少的场景中能够极大提升程序的性能。

读写锁具有以下三种特性：

- 读/读 不互斥：允许多个线程同时执行读方法，并且不会被阻塞；
- 读/写 互斥：如果有一个线程要访问写方法，此时有其他线程正在执行读方法，那么写线程需要阻塞；如果一个线程要访问读方法，此时有其他线程正在执行写方法，那么读线程需要阻塞；
- 写/写互斥：如果有多个线程同时执行写方法，则必须按照互斥规则进行同步。

### ReentrantReadWriteLock源码分析
`ReentrantReadWriteLock`的类关系如下图所示，可以看到，它是基于`AbstractQueueSynchronizer`来实现独占锁功能的，这里根据读写锁的特性可推测出，它可能使用了AQS的共享锁和排他锁功能。
![](media/16420883054742/16427808401210.jpg)

下面我们看看`ReentrantReadWriteLock`是如何初始化的。
```java
 public class ReentrantReadWriteLock
        implements ReadWriteLock, java.io.Serializable {
    private static final long serialVersionUID = -6992448646407690164L;
    /** Inner class providing readlock */
    private final ReentrantReadWriteLock.ReadLock readerLock;
    /** Inner class providing writelock */
    private final ReentrantReadWriteLock.WriteLock writerLock;
    /** Performs all synchronization mechanics */
    final Sync sync;

    /**
     * Creates a new {@code ReentrantReadWriteLock} with
     * default (nonfair) ordering properties.
     */
    public ReentrantReadWriteLock() {
        this(false);
    }

    /**
     * Creates a new {@code ReentrantReadWriteLock} with
     * the given fairness policy.
     *
     * @param fair {@code true} if this lock should use a fair ordering policy
     */
    public ReentrantReadWriteLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
        readerLock = new ReadLock(this);
        writerLock = new WriteLock(this);
    }
}
```
可以看到，在`ReentrantReadWriteLock`里定义了一个读锁，一个写锁，还有一个同步器，跟`ReentrantLock`一样，会在构造方法里初始化一个`FairSync`或者`NonfairSync`，`ReadLock`，`WriteLock`。接下来我们先看下写锁是如何加锁和释放锁的。

### WriteLock的锁竞争原理
#### WriteLock.lock()
首先，我们看下`WriteLock.lock()`方法。
```java
public void lock() {
    sync.acquire(1);
}

public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```
在写锁的`lock()`方法里，与重入锁一样，会调用AQS的`acuqire(1)`方法，同样会调用`tryAcquire()`方法去争抢锁。
```java
protected final boolean tryAcquire(int acquires) {
    /*
     * Walkthrough:
     * 1. If read count nonzero or write count nonzero
     *    and owner is a different thread, fail.
     * 2. If count would saturate, fail. (This can only
     *    happen if count is already nonzero.)
     * 3. Otherwise, this thread is eligible for lock if
     *    it is either a reentrant acquire or
     *    queue policy allows it. If so, update state
     *    and set owner.
     */
    Thread current = Thread.currentThread();
    // 获取state变量
    int c = getState();
    // 获取独占锁的线程数
    int w = exclusiveCount(c);
    if (c != 0) { // 如果此时已经有线程获取到锁的情况
        // 如果w==0，说明有线程获取到读锁
        if (w == 0 || current != getExclusiveOwnerThread())
            // 如果此时有线程抢到读锁 或者 抢到独占锁的不是当前线程，则抢占失败
            return false;
        if (w + exclusiveCount(acquires) > MAX_COUNT) 
            // 如果抢占锁的线程数大于最大值，抛出异常
            throw new Error("Maximum lock count exceeded");
        // Reentrant acquire
        // 如果不是读锁，抢占到锁的也是当前线程，则增加重入数量
        setState(c + acquires);
        return true;
    }
    // 如果当前没有线程抢占到锁的情况，则尝试使用cas方式设置state变量，
    // 如果设置成功，则设置独占的线程，否则直接返回false，表示抢占失败
    if (writerShouldBlock() ||
        !compareAndSetState(c, c + acquires))
        return false;
    setExclusiveOwnerThread(current);
    return true;
}

// ===============分割线===========================

static final int SHARED_SHIFT   = 16;
static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;
static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }

```
与`ReentrantLock`一样，本质上都是通过CAS方式设置AQS的`state`互斥变量来标识是否抢占到锁资源，但实现逻辑上还是有些不同，总体步骤如下：
- 先拿到AQS的互斥变量`state`，
- 通过`exclusiveCount()`方法获取到独占锁的线程数量；
- 判断AQS的互斥变量`state`，如果不为0，则表示此时已经有线程获取到锁了（可能是读锁或者是写锁）；如果是其他线程抢到锁，则返回false，否则会增加线程的重入次数
- 如果`state`等于0，表明现在没有线程抢占到锁，则通过CAS方法设置互斥变量，成功的情况下设置独占的线程信息。

在源码中，可能大家无法理解的是`int w = exclusiveCount(c)`，在该方法中会将AQS的互斥变量与`EXCLUSIVE_MASK`做与运算，而`EXCLUSIVE_MASK`是通过1通过二进制左移动16位后减1得到的值，为什么可以通过这样的运算可以得到独占锁的线程数呢？
其实`ReentrantReadWriteLock`的读写锁是通过`state`的高低位来进行存储的，高16位存储读锁状态，低16位存储写状态。
```java
            高16位                  低16位
state : 00000000 00000001   |  0000000 000000001
        表示有一个线程获得读锁     表示有一个线程获得写锁
```
大家可参考上面的例子，如果有线程抢占到读锁，`state`变量会加上`2^16`,如果有线程抢占到写锁，则`state`变量会加上`1`。所以，无论线程是抢占到读锁还是写锁，`state`变量都会大于0，所以在`tryAcquire()`方法中判断`state`不等于0的情况下，说明有线程获取到了锁，但是不确定是否是读锁或者写锁，所以需要通过`exclusiveCount()`方法拿到独占锁的线程数量。在`exclusiveCount()`中的`EXCLUSIVE_MASK`变量二进制值是`1111111111111111`,所以计算写锁的线程数只要`EXCLUSIVE_MASK`与`state`的低16位做与运算即可，如果大于0，则表示此时有线程抢占到写锁。
> 2^16 = 65536 = 10000000000000000;
> 2^16 - 1 = 65535 = 1111111111111111;

#### WriteLock.unlock() 
`WriteLock`是释放锁也是通过`sync.release(1)`实现的。
```java
public final boolean release(int arg) {
    // 释放锁成功，会重新唤醒队列中还在阻塞的线程
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}

protected final boolean tryRelease(int releases) {
    // 如果不是持有锁的线程释放锁，抛出异常
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    
    // state - 1 
    int nextc = getState() - releases;
    // 判断独占的线程是否为0
    boolean free = exclusiveCount(nextc) == 0;
    if (free) // 如果是，将ExclusiveOwnerThread变量设置为null
        setExclusiveOwnerThread(null);
    // 重新设置state
    setState(nextc);
    return free;
}
```
`release()`方法是由AQS实现的，与`ReentrantLock`的流程一致，我们可以直接看`tryRelease()`方法，该方法是在`ReentrantReadWriteLock`中实现的。

- 先判断是否为持有锁的线程释放锁，如果不是，则抛出异常；
- 拿到state互斥变量并减少锁的次数，因为写锁的重入次数保存在低位，所以直接使用十进制计算接口；
- 通过exclusiveCount()方法计算写锁重入的次数，如果为0，则说明锁已经释放成功；
- 重新设置state互斥变量，并返回锁释放成功的标识。

> WriteLock锁竞争失败后进入阻塞的逻辑是在AQS中实现的，与ReentrantLock是一样的，这里不再重复分析。

### ReadLock的锁竞争原理

`ReadLock`允许多个线程同时获得锁，它是通过AQS实现的。同样，先看看`ReadLock`的`lock`方法。
```java
public void lock() {
    sync.acquireShared(1);
}

public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}
```
`ReadLock`会调用AQS的`acquireShared()`方法，然后又会调用`tryAcquireShared()`方法，其代码逻辑如下。
```java
protected final int tryAcquireShared(int unused) {
    /*
     * Walkthrough:
     * 1. If write lock held by another thread, fail.
     * 2. Otherwise, this thread is eligible for
     *    lock wrt state, so ask if it should block
     *    because of queue policy. If not, try
     *    to grant by CASing state and updating count.
     *    Note that step does not check for reentrant
     *    acquires, which is postponed to full version
     *    to avoid having to check hold count in
     *    the more typical non-reentrant case.
     * 3. If step 2 fails either because thread
     *    apparently not eligible or CAS fails or count
     *    saturated, chain to version with full retry loop.
     */
    Thread current = Thread.currentThread();
    // 拿到互斥变量state
    int c = getState();
    // 如果当前有线程获取到读锁 并且 不是本线程获取到锁的情况下，返回-1，表示阻塞
    if (exclusiveCount(c) != 0 &&
        getExclusiveOwnerThread() != current)
        return -1;
        
    // 获得读锁的线程数
    int r = sharedCount(c);
    // 如果不必阻塞读线程 并且 读线程数量小于最大值 并且 CAS设置state成功
    if (!readerShouldBlock() &&
        r < MAX_COUNT &&
        compareAndSetState(c, c + SHARED_UNIT)) {
        if (r == 0) { // 如果读线程数为0，表示第一次获取读锁
            firstReader = current; // 保存第一次获得读锁的线程
            firstReaderHoldCount = 1;
        } else if (firstReader == current) { // 读锁重入
            firstReaderHoldCount++;
        } else {
            // 通过ThreadLocal保存每个线程读锁的重入次数
            HoldCounter rh = cachedHoldCounter;
            if (rh == null || rh.tid != getThreadId(current))
                cachedHoldCounter = rh = readHolds.get();
            else if (rh.count == 0)
                readHolds.set(rh);
            rh.count++;
        }
        return 1;
    }
    // 同一轮争抢不到读锁的线程，会进入这个方法，尝试获取共享锁
    return fullTryAcquireShared(current);
}
```
`tryAcquireShared()`方法如果返回-1，表示需要等待其他线程释放写锁，此时需要进入阻塞状态。该方法流程如下：
- 先判断当前是否有其他线程获取写锁，如果有，返回-1，表示当前抢占读锁的线程需要进入阻塞，等待写线程释放锁；
- 然后通过sharedCount()拿到读线程的数量,此方法会拿到state的高16位的值（c >>> SHARED_SHIFT）；
- 在读线程不比阻塞并且读线程数量小于最大值（65535）的情况下，通过CAS设置互斥变量state，注意，这里修改state不是跟写锁一样简单的加上1，而是2^16，因为读锁是通过state高16位保存的。
- 如果抢占到读锁，则根据不同的条件进行处理：
    - r == 0: 表示第一次获得读锁；
    - firstReader == current: 表示第一次获得读锁为当前线程，继续记录重入次数；
    - 采用ThreadLocal保存每个线程获得读锁的次数。
- 如果同一轮争抢不到读锁，则进入fullTryAcquireShared()方法继续尝试获取读锁。

在`tryAcquireShared()`方法中，使用到`readerShouldBlock()`方法来的判断是否需要阻塞读线程，它在`NonfairSync`和`FairSync`有不同的实现，对于公平锁来说，会调用`hasQueuedPredecessors()`方法判断同步队列中是否正在等待的线程，如果没有，才能尝试去抢占锁，非公平锁的实现则会调用`apparentlyFirstQueuedIsExclusive()`方法。
```java
final boolean apparentlyFirstQueuedIsExclusive() {
    Node h, s;
    return (h = head) != null &&
        (s = h.next)  != null &&
        !s.isShared()         &&
        s.thread != null;
}
```
这个方法的目的是避免写锁无限等待的问题。
- 如果同步队列中head节点的下一个节点是独占节点，那么该方法会返回true，表示当前抢占读锁的线程需要排队；
- 如果同步队列中head节点的下一个节点是共享节点，那么该方法会返回false，表示允许当前抢占读锁的线程通过CAS修改互斥锁的状态。

下面继续看`fullTryAcquireShared()`方法
```java
final int fullTryAcquireShared(Thread current) {
    /*
     * This code is in part redundant with that in
     * tryAcquireShared but is simpler overall by not
     * complicating tryAcquireShared with interactions between
     * retries and lazily reading hold counts.
     */
    HoldCounter rh = null;
    for (;;) {
        // 拿到互斥变量state
        int c = getState();
        if (exclusiveCount(c) != 0) { // 当前有线程获取到写锁的情况
            if (getExclusiveOwnerThread() != current) 
               // 如果获取写锁的不是当前线程，返回-1，进入阻塞状态
                return -1;
            // else we hold the exclusive lock; blocking here
            // would cause deadlock.
        } else if (readerShouldBlock()) { 
        // 抢占读锁需要等待，表明当前有已经有线程抢占到读锁
            if (firstReader == current) {
            } else { 
                // 抢占到锁的不是当前线程，需要进入阻塞
                if (rh == null) {
                    rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current)) {
                        rh = readHolds.get();
                        if (rh.count == 0)
                            readHolds.remove();
                    }
                }
                if (rh.count == 0)
                    return -1;
            }
        }
        if (sharedCount(c) == MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        if (compareAndSetState(c, c + SHARED_UNIT)) {
            if (sharedCount(c) == 0) {
                firstReader = current;
                firstReaderHoldCount = 1;
            } else if (firstReader == current) {
                firstReaderHoldCount++;
            } else {
                if (rh == null)
                    rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                    rh = readHolds.get();
                else if (rh.count == 0)
                    readHolds.set(rh);
                rh.count++;
                cachedHoldCounter = rh; // cache for release
            }
            return 1;
        }
    }
}
```
`fullTryAcquireShared()`方法与`tryAcquireShared()`方法类似，只是增加了自旋来保证抢占到读锁,在该方法中，有以下两种情况需要加入到同步队列等待。
- 当前有其他线程获得了写锁，并且当前线程不是重入；
- readerShouldBlock()方法返回true并且不是重入。

最后看到`doAcquireShared()`方法
```java
private void doAcquireShared(int arg) {
    // 向同步队列中添加一个SHARED节点
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            // 找到新添加节点的前置节点
            final Node p = node.predecessor();
            if (p == head) {
                // 如果该SHARED节点为头节点，并且抢到读锁的情况下
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    // 唤醒同步队列中所有的SHARED节点
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
            // 不是头节点或者抢不到锁的情况下，剔除队列中的无效节点并挂起该线程
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```
`doAcquireShared()`方法是在AQS中，针对同步队列节点进行操作，它会将新增的节点添加到不同队列中，如果当前线程成功抢到读锁的情况下，会把所有`SHARED`节点都唤醒，也就是允许多个线程抢占到资源，这符合读写锁的特性。

### 总结
本文从`ReentrantReadWriteLock`的基本使用入手，分析了其原理，本质上是通过AQS提供的排他锁和共享锁功能来实现读写锁，通过互斥变量`state`的高低位来区分读写锁，总体实现与`ReentrantLock`类似。

> 对于ReentrantReadWriteLock的相关知识就讲到这里。读完记得 **赞** 一个，如发现文章有错误知识点，可以点击 **阅读原文** 给笔者留言修正。