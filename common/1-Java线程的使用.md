# 初探Java多线程

> 如果你还不知道进程和线程是什么，请先自行百度，本文不再赘述其概念。
### Java线程的使用
在Java中，实现线程的方式有多种，继承Thread类，实现Runnable接口，使用Callable/Future获取多线程返回值，ExecutorService线程池等等，由此可见，在Java中使用多线程是比较简单的。
* 实现Runnable接口
  ```java
  public class RunnableExample implements Runnable{
    @Override
    public void run() {
        System.out.println("This is a java runnable thread");
    }

    public static void main(String[] args) {
        new Thread(new RunnableExample()).start();
    }
}
  ```
在Runnable接口中有一个run()方法，当调用线程的start()启动线程后，JVM会回调接口的run()方法执行里面的逻辑。

执行结果：
  ```txt
  This is a java runnable thread

Process finished with exit code 0
  ```
* 继承Thread类
  ```java
  public class ThreadExample extends Thread {

    @Override
    public void run() {
        System.out.println("This is ThreadExample ");
    }

    public static void main(String[] args) {
        new ThreadExample().start();
    }
}
  ```
Thread类实现了Runnable接口，在继承Thread类需要重写run()方法，和Runnable接口一样，在线程启动时会回调run()方法，我们可以看下Thread类run()方法源码
```java

 /* What will be run. */
    private Runnable target;

    @Override
    public void run() {
        if (target != null) {
            target.run();
        }
    }
```
可以看到，如果没有重写run()方法，它会默认调用target的run()方法，target是一个Runnable接口，在实现Runnable接口方式中所实现的run()方法便是在这里调用的。而继承Thread类重写run()方法，则是将原来的源码逻辑替换成自己的业务代码逻辑。

* Callable/Future

Callable/Future的方式与前面两种有些不同，在实现Callable接口的同时，还需要一个FutureTask来接收线程的返回值
  ```java
  public class CallableExample implements Callable<Integer> {
    @Override
    public Integer call() throws Exception {
        return 1;
    }

    public static void main(String[] args) throws ExecutionException, InterruptedException {
        FutureTask task = new FutureTask(new CallableExample());
        new Thread(task).start();
        Integer result = (Integer)task.get();
        System.out.println("callable result: "+result);
    }
}
  ```
执行结果：
  ```txt
  callable result: 1

Process finished with exit code 0
  ```
Callable是一个简单的Java接口，里面只有一个call方法，我们需要在call方法开发自己的业务代码，那这个方法是怎么通过线程调用，然后又获取到返回值的呢？想必你也可以想到，答案就藏在FutureTask中。
  ```java
  public class FutureTask<V> implements RunnableFuture<V> {
      // 保存结果
      private Object outcome; // non-volatile, protected by state reads/writes
      
      // Callable接口，run方法中会调用其call方法
      private Callable<V> callable;
  
      public FutureTask(Callable<V> callable) {
        if (callable == null)
            throw new NullPointerException();
        this.callable = callable;
        this.state = NEW;       // ensure visibility of callable
      }
     
      public void run() {
        ...
        try {
            // 拿到传进来的Callable接口
            Callable<V> c = callable;
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    // 执行Callable接口的逻辑获取返回值
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {
                    ...
                }
                if (ran)
                    // 保存返回值
                    set(result);
            }
        } finally {
          ...
        }
      }
    
      protected void set(V v) {
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
            outcome = v;
            UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
            finishCompletion();
        }
      }
  }
  ```
可以看出，FutureTask实现啦RunnableFuture接口，而RunnableFuture继承了Runnable接口，所以，我们可以直接跳到run()方法中，可以看到，FutureTask会先拿到Callable，然后调用它的call()方法，这里便拿到call()方法的执行结果，然后调用set方法保存这个返回值，也就是将结果值保存在outcome这个全局变量中。
接下来我们只要调用get方法就可以拿到这个返回值了，需要注意的是，当线程未执行完就去调用get方法获取返回值时，主线程会进入阻塞状态，等待call方法执行完成。

### 多线程原理
> 当我们通过Thread的start()方法启动线程时，底层到底做了什么工作呢？

我们可以先来看下start()方法的源码
 ```java
   public synchronized void start() {
        /**
         * This method is not invoked for the main method thread or "system"
         * group threads created/set up by the VM. Any new functionality added
         * to this method in the future may have to also be added to the VM.
         *
         * A zero status value corresponds to state "NEW".
         */
        if (threadStatus != 0)
            throw new IllegalThreadStateException();

        /* Notify the group that this thread is about to be started
         * so that it can be added to the group's list of threads
         * and the group's unstarted count can be decremented. */
        group.add(this);

        boolean started = false;
        try {
            start0();
            started = true;
        } finally {
            try {
                if (!started) {
                    group.threadStartFailed(this);
                }
            } catch (Throwable ignore) {
                /* do nothing. If start0 threw a Throwable then
                  it will be passed up the call stack */
            }
        }
    }

    private native void start0();
 ```
可以看到，在start()方法中，会调用native方法start0()，在此方法中，会先在JVM层面创建一个线程，JVM是跨平台的，它会根据当前系统类型，调用相关的指令来创建并启动线程。
调用start()方法并不会立即启动一个线程，而是要等到操作系统层面的CPU调度算法，把当前线程分配给某个CPU来执行。当线程分配完并执行后，会回调Runnbale中的run()方法执行相关的代码逻辑。

### 总结
今天主要讲了线程的基本使用，这里对相关知识点总结一下，方便大家记忆

1、Java线程主要有以下创建方式
* 实现Runnable接口
* 继承Thread类
* 使用Callable/Future（可获取返回值）
> 线程池的部分会在后面进行讲解

2、Callable/Future获取返回值的方式是通过FutureTask线程执行call方法，获取call方法的返回值后保存在FutureTask的全局变量outcome中，然后调用get方法获取outcome，当线程未执行完成时，get方法会进入阻塞；

3、真正启动一个线程需要调用start()方法，而不是直接调用run()方法，run()方法只是简单执行其里面的代码逻辑，无法真正创建一个线程；

4、调用start()方法不能立即启动线程，启动过程具有随机性，需要等CPU调度算法分配资源；


> 好了，今天的学习就到这里。记得读完 **赞** 一个，最新文章可以点击 **阅读全文**






