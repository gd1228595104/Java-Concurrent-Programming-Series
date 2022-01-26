# synchronized的使用与原理

在上篇并发编程系列文章中，笔者从线程安全性问题的三个方面入手，简单剖析了可见性和有序性问题的本质，今天继续将从剩下的原子性问题开始，一起去了解`synchronized`关键字的使用。

### 原子性问题
什么是原子性？

我们知道数据库的事务特性，同一事务下，包含多个对数据库的事务操作（`insert`,`update`,`delete`），这些操作，要么全部成功，要么全部失败，不会存在部分成功的情况，这就是数据库事务的原子性。线程中的原子性也是如此，多个指令在被CPU执行的过程中，要么全部成功，要么全部失败。

那原子性会带来什么问题呢？

我们知道，CPU在执行指令的时候是具有不确定性的，可能在一个线程中，一组指令执行到一半，CPU通过时间片切换，将资源让给另一个线程去执行，此时第一个线程的指令就被中断，可能执行出来的结果会不符合预期。比如Java中常见自增操作`i++`，这个简单指令，却不是一个原子指令，它是由三个指令组成的，`取值`，`+1`，`赋值`，下面我们可以看看这个指令被转成字节码后样子。
```java
private static int count;

public static void main(String[] args){
  count++;
  System.out.println(count);
}
```
笔者将上面这几句简单的指令转成字节码文件
```
 0: getstatic     #2                  // Field count:I
 3: iconst_1
 4: iadd
 5: putstatic     #2                  // Field count:I
 8: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
11: getstatic     #2                  // Field count:I
14: invokevirtual #4                  // Method java/io/PrintStream.println:(I)V
17: return
```
可以看到`count++`被分成以下几个指令
- `getstatic`：将变量count从内存加载到CPU中
- `iconst_1`：将变量压入栈中
- `iadd`：在CPU寄存器中执行+1操作
- `putstatic`： 将计算结果保存到内存中

在CPU执行其中任意一个指令的时候都有可能存在中断的情况，这个时候就会出现原子性问题。我们可以看看下面例子
```java
public class AtomicQuestionDemo {
    public static void main(String[] args) {
        List<Future> list = new ArrayList<>();
        for (int i = 0; i < 1000; i++) {
            FutureTask task = new FutureTask(() -> {
                Thread.sleep(1);
                return incr();
            });
            list.add(task);
            new Thread(task).start();
        }

        list.stream().forEach(f -> {
            try {
                f.get();
            } catch (InterruptedException e) {
                e.printStackTrace();
            } catch (ExecutionException e) {
                e.printStackTrace();
            }
        });
        System.out.println(count);
    }

    private static int count = 0;

    private static int incr() {
        return count++;
    }
}
```
开启1000个线程去累加全局变量`count`，最后得到的结果会小于等于1000，这就是非原子性指令在多线程的情况下可能会出现的问题，大家可以先看一下多线程下该指令可能执行的顺序：

- 线程1执行`getstatic`指令，将变量加载到`线程1`内存中，然后被中断
- 线程2执行`getstatic`指令，将变量加载到`线程2`的内存中，然后执行`add`操作
- 线程1重新继续执行`add`指令，对工作内存中的变量执行`add`操作

在`线程1`被中断时，此时已经将变量加载到自己的工作内存中了，`线程1`被中断后，`线程2`再加载到变量到自己的工作内存中，此时`线程2`与`线程1`加载的变量值都是相同，所以各自执行完`add`操作后的值是一样的，最终得到的结果就是`count`变量会比预期的小。

在多线程的情况下，如何保证非原子的指令在执行过程中不因为上下文切换而被中断呢？Java中`synchronized`便提供了这样一个功能。我们可以在`incr()`方法上加上`synchronized`关键字，此时重新执行上面的例子，得到结果就是1000.
```java
  private synchronized static int incr() {
      return count++;
  }
```

### synchronized使用方法

`synchronized`是一个同步锁，它将原本并行执行的代码串行化，从而保证线程安全性，但是，这种串行化执行明显会损失性能。

`synchronized`的使用方法比较简单，主要使用方式有两种，一种是修饰方法，另一种是修饰代码块

```java
public class SynchronizedDemo {

    private int count = 0;

    /**
     * 第一种方式，修饰方法
     */
    private synchronized void incr1() {
        count++;
    }

    /**
     * 第二种方式，修饰代码块
     */
    private void incr2() {
        synchronized (this) {
            count++;
        }
    }
}
```
- 第一种方式表示对`incr1()`方法加锁，当有多个线程同时访问该对象中此方法时，只有一个线程能够执行成功； 

- 第二种方式表示对代码块进行加锁，它与第一种方式类似，但是加锁的粒度更小，只有执行到被修饰的代码块时才会竞争锁资源，从而保证只有一个线程能够执行成功。

### synchronized作用域

`synchronized`的作用范围也有两种，一种是对象级别，一种是类级别。
#### 1）、对象锁
只有在多个线程在使用同一个对象实例的同步方法时，才会产生互斥，这就是对象锁，上面例子`synchronized`的使用方式的就是属于对象锁。下面给大家看一个例子。

```java
public class SynchronizedDemo {

    private static int count = 0;

    /**
     * 修饰方法
     */
    private synchronized void incr1() {
        count++;
    }

    public static void main(String[] args) {
        List<Future> list = new ArrayList<>();
        for (int i = 0; i < 1000; i++) {
            FutureTask task = new FutureTask(() -> {
                Thread.sleep(1);
                new SynchronizedDemo().incr1();
                return 1;
            });
            list.add(task);
            new Thread(task).start();
        }
        
        list.stream().forEach(f -> {
            try {
                f.get();
            } catch (InterruptedException e) {
                e.printStackTrace();
            } catch (ExecutionException e) {
                e.printStackTrace();
            }
        });
        System.out.println(count);
    }
}
```
运行结果
```txt
995
```
从运行结果可以看到，即使`incr1()`方法已经加上了`synchronized`关键字，运行结果还是无法保证正确的，而例子中`synchronized`是一个对象级别的同步锁，意味着同一时刻只能有一个线程抢占到共享资源 **(SynchronizedDemo实例对象)**,而示例中每个线程的都会自己单独创建一个实例，因此无法达到`排他`的特性,所以程序最终的运行结果无法得到保证。知道了运行结果不正确的原因，我们对例子进行修改。

```java
public static void main(String[] args) {
        SynchronizedDemo sd = new SynchronizedDemo();
        List<Future> list = new ArrayList<>();
        for (int i = 0; i < 1000; i++) {
            FutureTask task = new FutureTask(() -> {
                Thread.sleep(1);
                sd.incr1();
                return 1;
            });
            list.add(task);
            new Thread(task).start();
        }

        list.stream().forEach(f -> {
            try {
                f.get();
            } catch (InterruptedException e) {
                e.printStackTrace();
            } catch (ExecutionException e) {
                e.printStackTrace();
            }
        });
        System.out.println(count);
    }
```
在修改后的例子中，每个线程都会使用同一个对象实例的`incr1()`方法，因每个线程需要争抢的共享变量都是同一个`SynchronizedDemo`对象实例，所以此时可以保证排他性，从而保证了结果的正确性。

#### 2) 类锁
类锁是全局锁，与对象锁不同的是，类锁支持跨对象实例，可以在多个线程调用不同对象实例的同步方法时保证排他性。具体方式有如下
- 修饰静态方法；
```java
    private static synchronized void incr1() {count++;}
```
- 修饰代码块时，锁对象是类
```java
private void incr1() {
    synchronized (SynchronizedDemo.class) {
        count++;
    }
}
```
将对象锁第一个例子中的`incr1()`方法改成上面这两种方式的任意一种，都能够实现保证运行结果的正确性。为什么类锁的粒度会如此之大呢？
Class是在JVM启动过程中加载的，每个.class文件被装载后会产生一个Class对象，Class对象在JVM进程中是全局唯一的。而通过`static`修饰的成员对象及方法的生命周期都是属于类级别，因此，可以做到跨对象的效果。

`synchronized`在使用时需要修饰一个共享资源对象来实现互斥性，那线程之间是如何识别共享资源已经被抢占的呢？实际上，在对象头中，有一个锁标记位，用来记录锁状态。

### 对象头

Java中对象存储结构可以分为三个部分：对象头(Header)、实例数据(Instance Data)、对其填充(Padding)。
对象头又由三个部分组成：
- Mark Word：记录对象和锁相关信息
- Klass Pointer：指针，指明对象具体属于哪个类
- Length：数组长度，只有构建对象数组时才会有该属性

#### Mark Word

Mark Word存储对象自身的运行时数据，比如HashCode，GC分代年龄，锁标记状态，线程持有的，偏向线程ID等，具体结构如下（以32位系统为例）
![](media/16396686883992/16398167837375.jpg)
从图中可以看到，锁的状态有五种：无锁（01），偏向锁（01），轻量级锁（00），重量级锁（10），GC标记（11），其中还额外用1bit来区分无锁（0）和偏向锁（1）。
这些锁有什么含义呢？在JDK1.6之前，`synchronized`只提供了重量级锁，而重量级锁的本质就是阻塞，也就是没有获得锁的线程会被挂起进行阻塞，接着获得锁的线程会唤醒阻塞的线程让其继续抢占锁，直到抢占成功，这种方式需要从用户态切换到内核态执行，这种切换带来的性能开销是非常大的，因此，在JDK1.6之后，官方对`synchronized`进行了优化，引入了`偏向锁`和`轻量级锁`，让线程在不阻塞的情况下也能达到线程安全的目的。
`synchronized`在加锁的过程中，会按照 `无锁 -> 偏向锁（偏向锁有启动的情况下） -> 轻量级锁 -> 重量级锁`这样的顺序逐步对锁进行升级。

### 偏向锁
偏向锁指的是如果一个线程获得了锁（偏向锁），那么当该线程后续继续访问相同的同步方法或者相同锁时，只需要判断对象头的线程ID和该线程是否相等，如果相等，则不再需要去抢占锁，直接访问即可。下面我们可以通过一个例子看看对象头中偏向锁标记。
```java
public class BaseLockDemo {

    public static void main(String[] args) throws InterruptedException {
        BaseLockDemo baseLockDemo = new BaseLockDemo();
        System.out.println("=====before lock=======");
        System.out.println(ClassLayout.parseInstance(baseLockDemo).toPrintable());
        System.out.println("=====after lock");
        synchronized (baseLockDemo){
            System.out.println(ClassLayout.parseInstance(baseLockDemo).toPrintable());
        }
    }
}
```
运行结果
```txt
=====before lock=======
# WARNING: Unable to attach Serviceability Agent. You can try again with escalated privileges. Two options: a) use -Djol.tryWithSudo=true to try with sudo; b) echo 0 | sudo tee /proc/sys/kernel/yama/ptrace_scope
org.example.synchronize.BaseLockDemo object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           05 c1 00 f8 (00000101 11000001 00000000 11111000) (-134168315)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

=====after lock========
org.example.synchronize.BaseLockDemo object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           c0 89 ff 09 (11000000 10001001 11111111 00001001) (167741888)
      4     4        (object header)                           03 00 00 00 (00000011 00000000 00000000 00000000) (3)
      8     4        (object header)                           05 c1 00 f8 (00000101 11000001 00000000 11111000) (-134168315)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```
可以看到，在加锁之前，对象头第一个字节`00000001`最后三位`001`，表示此时是无锁状态，加锁之后，对象头第一个字节会变成`11000000`,最后三位变成`000`，表示轻量级锁。按照上面的说法，从无锁状态开始，之后应该会是偏向锁，为什么此时标记位显示的却是轻量级锁呢？
这是因为JVM启动时，偏向锁是延迟开启的，之所以要延迟开启，是因为JVM在启动时会有很多线程运行，可能会存在线程竞争的情况，此时开启偏向锁的意义不大。偏向锁的延迟启动时间由启动参数`-XX:BiasedLockingStartupDelay`控制，默认是4秒。此时将参数设置为0(`-XX:BiasedLockingStartupDelay=0`)，也就是不延迟开启偏向锁，再次运行代码看看结果。
```
=====before lock=======
# WARNING: Unable to attach Serviceability Agent. You can try again with escalated privileges. Two options: a) use -Djol.tryWithSudo=true to try with sudo; b) echo 0 | sudo tee /proc/sys/kernel/yama/ptrace_scope
org.example.synchronize.BaseLockDemo object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 00 00 00 (00000101 00000000 00000000 00000000) (5)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           05 c1 00 f8 (00000101 11000001 00000000 11111000) (-134168315)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

=====after lock========
org.example.synchronize.BaseLockDemo object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 90 80 36 (00000101 10010000 10000000 00110110) (914395141)
      4     4        (object header)                           81 7f 00 00 (10000001 01111111 00000000 00000000) (32641)
      8     4        (object header)                           05 c1 00 f8 (00000101 11000001 00000000 11111000) (-134168315)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```
此时可以看到加锁后，对象头第一个字节最后三位都会变成`101`，表示偏向锁，符合预期。
细心的朋友应该也观察到，加锁之前的对象头第一个字节最后三位也是`101`，为什么会出现这种情况呢？笔者查阅资料找到一种分析，加锁之前并没有存储线程ID，加锁之后才会有一个线程ID（914395141），因此在获得偏向锁之前，`101`应该是表示当前是一种可偏向的预先状态，而不是当前已经处于状态。

### 轻量级锁
轻量级锁也称自旋锁，就是没有抢占到锁的线程，进行一定次数的CAS（Compare And Swap）操作去争抢锁，如果在重试的过程中抢占到锁，则该线程不需要被阻塞。
因为轻量级锁是通过自旋的方式实现，如果不断自旋重试，CPU会一直处于运行状态，而且只做没有意义的循环，这对CPU资源是极大的浪费；如果持有锁的线程占有锁的时间比较短，其他线程经过少量的重试操作就能获取到锁，那么这种自旋等待相比阻塞与唤醒线程而言，所带来的性能提升也是比较明显的。因此，在JDK1.6中，默认的自旋次数是10次，我们可以通过`-XX:PreBlockSpin`来设置自旋的次数。
JDK1.6中还对自旋锁进行了优化，引进了自适应自旋锁，如果一个线程通过自旋的方式获取到锁，那么下一次，针对同一锁对象，JVM会将线程的自旋时间相对延长；如果通过自旋的方式很难获取到锁，那么JVM会相对减少自旋的次数。

### 重量级锁
如果轻量级锁在经过一定次数的自旋之后，还获取不到锁，此时过多的循环操作会过多浪费CPU资源，`synchronized`便会将锁升级为重量级锁，挂起线程，使其进入阻塞状态，释放CPU资源，此时对象头的的锁标记会变成`10`。
下面例子是在分别在线程获取锁之前，获取到锁后，main线程争抢到锁三个地方打印对象头信息。
```java
public class BlockLockDemo {

    public static void main(String[] args) throws InterruptedException {
        Object lock = new Object();
        System.out.println("===before lock====");
        System.out.println(ClassLayout.parseInstance(lock).toPrintable());
        new Thread(() -> {
            synchronized (lock) {
                System.out.println("=====thread get lock======");
                System.out.println(ClassLayout.parseInstance(lock).toPrintable());
                try {
                    Thread.sleep(5000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }).start();
        Thread.sleep(1000);
        synchronized (lock) {
            System.out.println("======main get lock======");
            System.out.println(ClassLayout.parseInstance(lock).toPrintable());
        }
    }
}
```
运行结果
```
===before lock====
# WARNING: Unable to attach Serviceability Agent. You can try again with escalated privileges. Two options: a) use -Djol.tryWithSudo=true to try with sudo; b) echo 0 | sudo tee /proc/sys/kernel/yama/ptrace_scope
java.lang.Object object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           e5 01 00 f8 (11100101 00000001 00000000 11111000) (-134217243)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

=====thread get lock======
java.lang.Object object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           e0 c9 5d 0a (11100000 11001001 01011101 00001010) (173918688)
      4     4        (object header)                           03 00 00 00 (00000011 00000000 00000000 00000000) (3)
      8     4        (object header)                           e5 01 00 f8 (11100101 00000001 00000000 11111000) (-134217243)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

======main get lock======
java.lang.Object object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           4a b1 82 64 (01001010 10110001 10000010 01100100) (1686286666)
      4     4        (object header)                           c5 7f 00 00 (11000101 01111111 00000000 00000000) (32709)
      8     4        (object header)                           e5 01 00 f8 (11100101 00000001 00000000 11111000) (-134217243)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```
可以看到，在线程未获取到锁之前，对象头第一个字节最后三位是`001`，表示无锁；获取到锁之后,最后两位变成了`00`，表示轻量级锁；main线程争抢锁之后，变成了`10`，表示重量级锁，可以看到，`synchronized`底层会有一个锁升级机制，JVM根据线程的竞争情况逐步升级锁，在保证安全性的情况下，尽量提升了同步锁的性能，这个过程不需要开发者干预，且是不可逆的。

### 死锁
虽然`synchronized`可以解决线程安全问题，但是，如果使用不当，会造成死锁，导致请求一直被阻塞无法返回。
什么是死锁呢？两个或两个以上的线程由于争夺共享资源而造成的相互等待的对象，在这个过程中，两个线程都无法正常向下执行，这种线程也称为`死锁线程`。下面看看死锁的例子。
```java
public class DeadLockDemo {

    @Data
    @AllArgsConstructor
    static class Person {
        private String name;
        private Long money;
    }


    public static void transferMoney(Person from, Person to, long money) {
        if (from.money <= 0 || to.money <= 0) {
            System.out.println("Any one do not have enough money");
            System.out.println(from.name + " has " + from.money);
            System.out.println(to.name + " has " + to.money);
            return;
        }
        synchronized (from) {
            from.money -= money;
            synchronized (to) {
                to.money += money;
                System.out.println("Transferring money... from: " + from.name + " to:" + to.name + " money:" + money);
            }
        }
    }

    public static void main(String[] args) {
        Person p1 = new Person("p1", 100000L);
        Person p2 = new Person("p2", 100000L);
        new Thread(() -> {
            for (; ; ) {
                transferMoney(p1, p2, 10);
            }
        },"TRANSFER_1").start();
        new Thread(() -> {
            for (; ; ) {
                transferMoney(p2, p1, 5);
            }
        },"TRANSFER_2").start();
    }
}
```
在例子中模拟转帐场景，`p1`和`p2`同时向对方转钱，在转账方法中，会先对出账方进行加锁，然后减去对应金额，然后对入账方加锁，加上对应的金额，两个线程分别相同锁对象加锁，但加锁的顺序不一样，这必然会造成死锁问题。我们可以通过jstack查看线程的堆栈信息。
```
"TRANSFER_2" #12 prio=5 os_prio=31 tid=0x00007fe25e818800 nid=0x6003 waiting for monitor entry [0x000000030af69000]
   java.lang.Thread.State: BLOCKED (on object monitor)
        at org.example.synchronize.DeadLockDemo.transferMoney(DeadLockDemo.java:31)
        - waiting to lock <0x000000076ac2f938> (a org.example.synchronize.DeadLockDemo$Person)
        - locked <0x000000076ac2fb18> (a org.example.synchronize.DeadLockDemo$Person)
        at org.example.synchronize.DeadLockDemo.lambda$main$1(DeadLockDemo.java:47)
        at org.example.synchronize.DeadLockDemo$$Lambda$2/2129789493.run(Unknown Source)
        at java.lang.Thread.run(Thread.java:748)

"TRANSFER_1" #11 prio=5 os_prio=31 tid=0x00007fe27f8bd800 nid=0x5d5f waiting for monitor entry [0x000000030ae66000]
   java.lang.Thread.State: BLOCKED (on object monitor)
        at org.example.synchronize.DeadLockDemo.transferMoney(DeadLockDemo.java:31)
        - waiting to lock <0x000000076ac2fb18> (a org.example.synchronize.DeadLockDemo$Person)
        - locked <0x000000076ac2f938> (a org.example.synchronize.DeadLockDemo$Person)
        at org.example.synchronize.DeadLockDemo.lambda$main$0(DeadLockDemo.java:42)
        at org.example.synchronize.DeadLockDemo$$Lambda$1/764977973.run(Unknown Source)
        at java.lang.Thread.run(Thread.java:748)
```
可以看到，两个转账线程同时处于`BLOCKED`状态，无法继续向下执行。

#### 产生死锁的必要条件
导致死锁有四个必要条件
- 互斥条件：共享资源A和B只能被一个线程占用；
- 请求和保持条件：线程1已经抢占到共享资源A，在抢占资源B时，不释放资源A；
- 不可剥夺条件：线程1抢占到的资源A，在未主动释放之前，其他线程不能强行抢占线程1的资源；
- 循环等待条件：线程1等待线程2占有的资源，线程2等待线程1占有的资源。

#### 预防死锁问题方法
导致死锁问题，以上四个条件缺一不可，但相对应的，我们只需要破坏其中任意一个条件，就能避免死锁的产生，因为互斥条件是资源使用的固有特性，所以我们无法改变，但其他三个条件是可以破坏的，
 
 - 破坏“请求和保持条件”：我们可以一次性申请所有的资源，这样便不存在等待问题；
 - 破坏“不可抢占条件”：在已经占有了一部分资源的情况下，如果申请不到其他资源，可以主动释放掉已经占有的资源；
 - 破坏“循环等待条件”：在申请资源的时候，可以先申请资源序号小的，再申请资源序号大的，这种显性化申请资源自然就不存在循环等待了。

 下面将基于上面的例子对代码进行改造，针对三种破坏死锁的方法进行演示。
 
 ##### 1、破坏请求和保持条件
 在原来的基础上，笔者新增了一个`ResourceLock`类，在`transferMoney()`方法中通过该类的`trrlock()`方法统一申请资源，然后统一通过`unlock()`方法释放所有资源，这样便打破了请求和保持条件。
 ```java
 static class ResourceLock {
        private List<Object> list = new ArrayList<>();

        public synchronized boolean trylock(Object lock_1, Object lock_2) {
            if (list.contains(lock_1) || list.contains(lock_2)) {
                return false;
            }
            list.add(lock_1);
            list.add(lock_2);
            return true;
        }

        public synchronized void unlock() {
            list.clear();
        }
    }

    private static ResourceLock lock = new ResourceLock();

    public static void transferMoney(Person from, Person to, long money) {
        if (from.money <= 0 || to.money <= 0) {
            System.out.println("Any one do not have enough money");
            System.out.println(from.name + " has " + from.money);
            System.out.println(to.name + " has " + to.money);
            return;
        }
        while (!lock.trylock(from, to)) {

        }
        try {
            from.money -= money;
            to.money += money;
        } finally {
            lock.unlock();
        }
    }
 ```
 
 ##### 2、破坏不可抢占条件
 破坏不可抢占条件，需要当前线程能够主动释放占有的资源，`synchronized`因为抢占不到资源时会进入阻塞状态，所以无法实现破坏不可抢占条件这一点。但是JUC中的Lock锁可以轻松解决这个问题。
 ```java
   private static Lock lock = new ReentrantLock();

    public static void transferMoney(Person from, Person to, long money) {
        if (from.money <= 0 || to.money <= 0) {
            System.out.println("Any one do not have enough money");
            System.out.println(from.name + " has " + from.money);
            System.out.println(to.name + " has " + to.money);
            return;
        }
        if (lock.tryLock()) {
            try {
                from.money -= money;
                if (lock.tryLock()) {
                    try {
                        to.money += money;
                        System.out.println("Transferring money... from: " + from.name + " to:" + to.name + " money:" + money);
                    } finally {
                        lock.unlock();
                    }
                } else {
                    System.out.println("抢占失败");
                }
            } finally {
                lock.unlock();
            }
        } else {
            System.out.println("抢占失败");
        }
    }
 ```
 ##### 3、破坏循环等待条件
 破坏循环等待，就是按照资源的某种顺序进行编号，然后根据序号获取锁，笔者这里直接用`Person`的名称作为排序。
 ```java
 public static void transferMoney(Person from, Person to, long money) {
        if (from.money <= 0 || to.money <= 0) {
            System.out.println("Any one do not have enough money");
            System.out.println(from.name + " has " + from.money);
            System.out.println(to.name + " has " + to.money);
            return;
        }
        String name1 = from.name;
        String name2 = to.name;
        Object lock1 = null;
        Object lock2 = null;
        if (name1.compareTo(name2) >= 0) {
            lock1 = from;
            lock2 = to;
        } else {
            lock1 = to;
            lock2 = from;
        }
        synchronized (lock1) {
            synchronized (lock2) {
                from.money -= money;
                to.money += money;
                System.out.println("Transferring money... from: " + from.name + " to:" + to.name + " money:" + money);
            }
        }
    }
 ```
 
 ### 总结
 本文主要从原子问题开始，讲解`synchronized`关键字的使用，了解了`synchronized`在对象头中的锁标记，为了在保证安全性的前提下，提升锁的性能，引入了偏向锁和轻量级锁，最后对死锁问题做了简单阐述与解决死锁问题的演示。
 Java中除了`synchronized`可以解决线程安全问题外，在JUC（java.util.concurrent）包下还有各种工具可以解决线程同步与通信问题，比如：`ReentrantLock`,`CountDownLatch`,`ThreadLocal`,`Semaphore`等等,后续会逐一给大家讲解。
 
 > 对于synchronized的相关知识就讲到这里。读完记得 **赞** 一个，如发现文章有错误知识点，可以点击 **阅读原文** 给笔者留言修正。 