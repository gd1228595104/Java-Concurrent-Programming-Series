# 从可见性问题理解volatile的作用

在并发编程中，不同的线程可能会对同一个变量进行操作，如果在没有任何措施的干预下，对这个变量的操作结果会有很多不确定性。这就是线程的安全性问题，它主要体现在以下三个方面：

- 可见性
- 有序性
- 原子性

### 可见性
什么是可见性？

一个线程对一个共享变量进行修改，其他线程无法及时获取该共享变量修改后的值，这就是可见性问题。我们先看看例子

```java
public class VolatileQuestionDemo {

    private static boolean stop = false;

    public static void main(String[] args) throws InterruptedException {
        new Thread(() ->{
            while (!stop){

            }
        }).start();
        Thread.sleep(1000);
        stop = true;

    }
}
```
这个例子中，虽然在`main`方法中将全局变量设置为`true`，但线程却一直无法结束。这个问题只有在`HotSpot`使用`Server Compiler（C2编译器）`时才会出现。
> HotSpot虚拟机中内置了两个即时编译器，Client Compiler（C1 编译器）和Server Compiler（C2 编译器），C2 编译器面向服务器端，它会对我们的代码进行一定程度的优化，比如：消除无用代码（Dead Code Elimination），循环展开（Loop Unrolling），循环表达式外提（Loop Expression Hoisting），消除公共子表达式（Common Subexpression Elimination）等等

在上面这个例子中，`while(!stop)`会被优化成下面的样子,因此，无法程序无法正常结束。
```java
if(!stop){
  while(true){}
}
```
我们可以通过修改JVM参数`-Djava.compiler=NONE`来关闭JIT优化，那么程序便可以正常结束了，但是这种做法代价太大了。

Java中提供了`volatile`关键字来解决可见性问题，在上面的例子中，如果全局变量`stop`用`volatile`修饰，那么这个程序是可以正常停止的
```java
    private static volatile boolean stop = false;
```
由此可见，`volatile`关键字可以禁止编译器优化，在多线程的情况下可以保证共享变量的可见性。

### 剖析可见性问题的本质

造成可见性问题的因素有多种，比如：CPU高速缓存，CPU指令重排序等，接下来笔者将从源头开始，给大家剖析这个问题的本质。

#### CPU高速缓存

CPU，计算机最核心的资源，它在做运算时，无可避免地需要从内存中读取数据或者指令，但是CPU的运算速度时远远高于读写内存的I/O速度，这种CPU与内存I/O的速度瓶颈也被称为冯诺依曼瓶颈。为了降低内存I/O会性能的影响，便出现了高速缓存，人们在CPU中设计了高速缓存，用来存储与内存交互的数据。我们以主流的X86架构为例，CPU高速缓存被分为了L1、L2、L3三级，它们的访问速度依次递减，容量依次增多。L1和L2都是CPU核内缓存，属于CPU私有的，L3则是跨CPU核心共享的缓存。当CPU读取数据时，会先从L1缓存开始读取，如果没有命中数据，则继续从L2，L3读取，L3没有命中数据的情况下才会访问内存；而加载数据的顺序与读取则是相反的，数据会先加载到L3，再到L2，最后才会到L1缓存。

![](https://files.mdnice.com/user/21786/c7d53442-0987-42b0-b834-1e8292eef2b4.png)

虽然高速缓存提高了CPU的运算性能，但是，因为L1和L2是CPU私有的，如果两个CPU加载了同一份数据，并且对其进行了修改，便会出现缓存一致性问题。

#### 缓存一致性协议
为了解决缓存一致性问题，人们便在CPU层面引入了总线索和缓存锁机制。
> 总线：是计算机各种功能部件之间传送信息的公共通信干线，可以划分为数据总线、地址总线和控制总线，分别用来传输数据、数据地址和控制信号。简单的理解就是CPU与内存、输入/输出设备进行数据交互时，必须经过总线来传输。

总线锁就是在总线上声明一个Lock#信号，表明只有当前的CPU可以访问共享内存，其它处理器的请求会被阻塞，这样便只有一个CPU可以访问共享内存，从而解决了缓存不一致的问题，但是这样极大影响了CPU的利用率，多核CPU便失去了意义。缓存锁指的是采用缓存一致性协议来保证多核CPU的缓存一致性。

缓存一致性协议有多种，比如MESI，MSI，MOSI等等，比较常见的便是MESI协议，MESI表示缓存行的四种状态：

- M（Modify）：表示共享数据被当前CPU缓存，并且已经被修改了，缓存中的数据与主内存中的数据不一致。
- E（Exclusive)：表示共享数据只被当前CPU缓存，但是没有被修改。
- S（Shared）：表示数据被多个CPU缓存，并且缓存中的数据都与主内存中的数据一致。
- I（Invalid）：表示缓存数据已经失效了。

下面看下几个状态的示例图

![](https://files.mdnice.com/user/21786/bb3d5dfd-0b14-47f9-a62e-da35fdd741c9.png)

当只有一个CPU缓存了`i`时，那么它的状态便是E

![](https://files.mdnice.com/user/21786/6810053c-ebc9-4668-a285-eee095082b17.png)
此时，CPU_2也读取了`i`值，那么CPU_1和CPU_2同时缓存了`i`值，且`i`的值没有改变，那么它的状态会被设置为`S`

![](https://files.mdnice.com/user/21786/d9c39359-b73a-4845-8007-f47fc6db734c.png)


如果CPU_1继续对`i`值进行修改，CPU_1的状态会被设置为`M`，然后CPU_1会通过总线发送信号，表明它对`i`值有改动，CPU_2监听到这个信号时，会把自己内存中`i`值状态设置为`I`，需要重新读取最新数据。

至此，我们可以知道，在CPU层面，会通过锁机制（总线锁/缓存锁）来解决缓存一致性问题。下面我们看一下`volatile`关键字是怎么解决可见性问题的，这里笔者使用 hsdis 工具获取JIT生成的汇编指令，看看`volatile`带来的变化.
首先设置JVM参数
```txt
-server -Xcomp -XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly -XX:CompileCommand=compileonly,*VolatileQuestionDemo.*
```
执行上面的例子可以得到汇编指令，可以看到在修改`stop`变量时，在前面会有`lock`字样，这里表明是基于总线锁/缓存锁的方式来保证缓存一致性，从而保证结果的可见性。

```txt
  0x000000010d841777: lock addl $0x0,(%rsp)     ;*putstatic stop
                                                ; - org.example.safethread.safequestion.VolatileQuestionDemo::main@20 (line 17)
```
### 指令重排序

上面提到，指令重排序也是导致可见性问题的原因之一。那什么是指令重排序呢？

指令重排序是指编译器或者CPU为了优化程序执行性能而对指令进行重新排序。根据`as if serial`语义，在单线程的情况下，所有的程序指令都是可以因为优化而被重排序的，且重排序之后的运行结果和程序代码本身的预期执行结果是一样的。为了保证语义的正确 ，编译器和处理器不会对存在依赖关系的操作进行指令重排序，即使如此，在多线程的情况下，重排序还是会带来可见性问题。

前面讲到缓存一致性协议虽然理论上可以保证可见性，但为了避免不必要的阻塞，在每个CPU中又增加了一个`Store Buffers`，你可以把`Store Buffers`理解为一个队列，CPU可以把指令直接放在队列中（比如`I`指令），还未得到指令结果便往后执行，从而减少缓存同步导致的CPU性能损耗，这也是指令重排的根本原因。不过这种异步的思想，虽然提升了性能，却无法保证数据一致性。而且，不论CPU本身如何优化，它并不清楚指令什么时候需要优化，什么时候不需要，因而出现了内存屏障指令，让程序开发者自己去确定程序指令是否需要优化，这样便把数据一致性问题抛给了开发者。

### 内存屏障

大多数处理器会提供以下内存屏障的指令
- SFENCE（写屏障）：在该指令前的写操作必须在该指令后的写操作前完成
- LFENCE（读屏障）：在该指令前的读操作必须在该指令后的读操作前完成
- MFENCE（读写屏障）：在该指令前的读写操作必须在该指令后的读写操作前完成

> 本文偏重于Java应用层面，这里对底层原理不过多赘述，有兴趣的同学自行百度相关资料

这便是CPU层面通过屏障来控制读写指令的顺序性，但是每种CPU对指令的实现方法也不同，Java作为一个跨平台语言，必须要针对不同的操作系统和硬件提供统一的线程安全保障，JMM（Java Memory Mode）应运而生。

### JMM

JMM是一种规范，定义了线程与主内存之间的关系，它不像JVM的一样是真实存在的，它只是描述了线程对一个共享变量的写操作何时对另外一个线程可见。

- 每个线程都有一个用来存储数据的工作内存，工作内存保存了主内存中的变量副本，线程对变量的操作是在工作内存中进行的。
- 每个线程的工作内存都是相互隔离的，数据的变更需要通过主内存完成。
- 所有变量都存储在主内存中。

有没有JMM的定义线程与主内存的关系和CPU的架构有点类似。这种规范可以用来屏蔽掉各种硬件和操作系统的内存访问差异，从而实现让Java程序在各种平台下都能达到一致的内存访问效果。
JMM定义了以下8种操作来完成一个变量从主内存拷贝到工作内存或者从工作内存同步回主内存之类的实现细节：

- lock（锁定）：作用于主内存的变量，它把一个变量标识为一个线程的独占状态
- unlock（解锁）：作用于主内存的变量，它把一个处于锁定状态的变量释放出来，释放之后的变量才能够被其他线程锁定
- read（读取）：作用于主内存的变量，它把一个变量的值从主内存传输到线程的工作内存中，
- load（载入）：作用于工作内存的变量，它把read操作从主内存中中得到的变量值放入到工作内存的变量副本中
- use（使用）：作用于工作内存的变量，它把工作内存中一个变量的值传递给执行引擎
- assign（赋值）：作用于工作内存的变量，它把一个从执行引擎接收到的值赋给工作内存的变量
- store（存储）：作用于工作内存的变量，它把工作内存中一个变量的值传送到主内存中。
- write（写入）：作用于主内存的变量，它把store操作从工作内存中得到的变量的值放入到主内存的变量中

除此之外，JMM规定了在执行上述8种基本操作时必须满足如下规则：

- 不允许read和load、store和write操作之一单独出现，即不允许一个变量从主内存读取后但工作内存不接受，或者从工作内存发起回写了单主内存不接受的情况发生
- 不允许一个线程丢弃它最近的assign操作，即变量变化后必须把该变化同步回主内存
- 不允许一个线程无原因地（没有发生过任何assign操作）把数据从线程的工作内存同步回主内存中
- 一个新的变量只能在主内存中“诞生”，不允许在工作内存中直接使用一个未初始化的变量（没有执行过load或者assign）
- 一个变量在同一个时刻只允许一个线程对其进行lock操作，但lock操作可以被同一个线程重复执行多次，多次执行lock后，只有执行相同次数的unlock操作，变量才能被解锁
- 如果对一个变量执行lock操作，那将会清空工作内存中此变量的值，在执行引擎使用这个变量之前，需要重新执行load或assign操作初始化变量的值
- 如果一个变量事先没有被lock操作锁定，那就不允许对它执行unlock操作，也不允许去unlock一个被其他线程锁住的变量
- 对一个变量执行unlock操作之前，必须先把该变量同步回主内存中（执行store、write操作）。


在JMM规范的基础上，JVM同样提供来内存屏障来解决可见性问题。

- LoadLoad Barriers：确保在该指令前的读操作必须在该指令后的读操作前完成
- StoreStore Barriers: 确保该指令前的写操作数据对于在该指令后的写操作时可见的，也就是说指令前的写操作必须刷新到内存中
- LoadStore Barriers：确保该指令前的读操作必须在该指令后的存储指令被刷新之前，加载到内存中的数据
- StoreLoad Barriers：确保该指令之前的写操作结果对于指令后的读操作是可见的，也就是说指令前的写操作必须在读操作之前将数据刷新到内存中。

在最开始的例子中，我们通过`volatile`解决了由于编译器的指令重排序导致的可见性问题，这意味着该关键字底层用到了内存屏障。至此，相信同学对可见性，有序性和`volatile`也有了一个整体的了解。

### Happens-Before

在JMM中，还定义了`happens-before`模型来描述两个操作指令的顺序关系，如果操作A`happens-before`于操作B，操作A产生的影响能被操作B观察到。下面介绍一下常见的`happens-before`规则：

- 程序顺序规则（Program Order Rule）：在一个线程内，按照程序代码顺序，书写在前面的操作要先执行于后面的操作，这里更准确应该是说控制流顺序而不是程序代码顺序，因为根据as if serial语义，代码指令是允许被重排序的
- 监听器锁规则（Monitor Lock Rule）：一个unlock操作先行发生于后面对同一个锁的lock操作，即一个线程要获得锁，必须等另外一个占着锁的线程将锁unlock
- volatile变量规则（Volatile Variable Rule）：对一个volatile变量的写操作先行发生与后面对这个变量的读操作，
- 线程启动规则（Thread Start Rule）：Thread对象的start()方法先行发生于此线程的每一个动作
- 线程终止规则（Thread Termination Rule）： 线程中的所有操作都先行发生于对此线程的终止检测，我们可以用Thread.join()方法来等待线程执行完成，然后再执行main线程的余下代码。
- 线程中断规则（Thread Interruption Rule）：对于线程的interrupt()方法的调用先行发生于被中断线程的代码检测到中断事件的发生
- 对象终结规则（Finalizer Rule）：一个对象的初始化先行发生于它的finalize()方法的开始
- 传递性（Transitivity）：如果操作A先行发生于操作B，操作B先行发生于操作C，那么操作A先行发生于操作C

> [JavaTM 内存模型与线程规范](http://ifeve.com/wp-content/uploads/2014/03/JSR133%E4%B8%AD%E6%96%87%E7%89%88.pdf)

### 总结

在本文章中，笔者主要从CPU层面，简单阐述可见性问题和有序性问题的根本，因为CPU无法识别代码指令是否需要被优化，所以提供了内存屏障指令，因为不同类型CPU，操作系统的指令实现不同，于是引出了JMM（java内存模型），最后简单介绍了Java中定义的指令执行规则`Happens-Before`规则。

> 好了，对于原子性问题等到下篇文章再继续阐述了。读完记得 赞 一个，如发现文章有错误知识点，可以点击 阅读原文 给笔者留言修正。


