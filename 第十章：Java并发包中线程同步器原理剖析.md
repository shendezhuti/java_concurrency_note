# 第十章：Java并发包中线程同步器原理剖析

## 目录

- [案例介绍](#1)
- [CountDownLatch和join方法的区别](#2)
- [实现原理探究](#3)
  - [void await()方法](#4)
  - [boolean await(long timeout, TimeUnit unit](#5)
  - [void countDown()](#6)
  - [long getCount()](#7)
  - [小结](#8)
- [回环屏障CyclicBarrier原理探究](#9)
  - [案例介绍](#10)
  - [源码分析](#11)
  - [int await()](#12)
  - [ boolean await(long timeout, TimeUnit unit)](#13)
  - [int dowait(boolean timed, long nanos)](#14)
  - [小结](#15)
- [信号量Semaphore原理探究](#16)
  - [类图结构](#17)
  - [void acquire()](#18)
  - [void acquire(int permits)](#19)
  - [ void acquireUninterruptibly()](#20)
  - [void acquireUninterruptibly(int permits)](#21)
  - [void release()](#22)
  - [void release(int permits)](#23)
- [总结](#24)

<h2 id='1'>案例介绍</h2>
在日常开发中经常会遇到需要在主线程中开启多个线程去并行执行任务，并且主线程需要等待所有子线程执行完毕后再进行汇总的场所。在CountDownLatch出现之前一般都使用线程的join()方法来实现这一点，但是join方法不够灵活，不能满足不同场景的需要，所以JDK开发组提供了CountDownLatch这个类，我们前面介绍的例子使用CountDownLatch会更加的优雅。

```java
public class JoinCountDownLatch {
    private static volatile CountDownLatch countDownLatch = new CountDownLatch(2);
    public static void main(String[]args)throws InterruptedException{
        Thread threadone = new Thread(new Runnable() {
            @Override
            public void run() {
                    try{
                        Thread.sleep(1000);
                    }catch(InterruptedException e){
                        e.printStackTrace();
                    }finally {
                        countDownLatch.countDown();
                    }
                    System.out.println("child threadone over");
            }
        });

        Thread threadtwo = new Thread(new Runnable() {
            @Override
            public void run() {
                try{
                    Thread.sleep(1000);
                }catch(InterruptedException e){
                    e.printStackTrace();
                }finally {
                    countDownLatch.countDown();
                }
                System.out.println("child threadone over");
            }
        });

        threadone.start();
        threadtwo.start();

        System.out.println("wait for all thread over");

        countDownLatch.await();
        System.out.println("all child thread over");

    }
}
```

上述代码中，创建一个CountDownLatch实例，因为有两个子线程所以构造函数的传参为2。主线程调用了countDownLatch.await()方法后会被阻塞。子线程执行完毕后调用countDownLatch.countDown()方法后让countDownLatch内部的计数器减1，所有子线程执行完毕后调用countDown()方法后计数器会变为0，这时候主线程的await()方法才会返回。

其实上面的代码不够优雅，在项目时间中一般都避免直接操作线程，而是使用ExecutorService线程池来管理。使用ExecutorService时传递的参数是Runnable或者Callble对象，这时候你没有办法直接调用这些线程的join()方法，这就需要选择使用CountDownLatch。

```java
public class JoinCountDownLatch2 {

    private static CountDownLatch countDownLatch = new CountDownLatch(2);
    public static void main(String []args) throws InterruptedException{
        ExecutorService executorService= Executors.newFixedThreadPool(2);
        executorService.submit(new Runnable() {
            @Override
            public void run() {
                try{
                    Thread.sleep(1000);
                }catch(InterruptedException e){
                    e.printStackTrace();
                }finally {
                    countDownLatch.countDown();
                }
                System.out.println("child threadOne over!");
            }
        });

        executorService.submit(new Runnable() {
            @Override
            public void run() {
                try{
                    Thread.sleep(1000);
                }catch(InterruptedException e){
                    e.printStackTrace();
                }finally {
                    countDownLatch.countDown();
                }
                System.out.println("child threadtwo over!");
            }
        });
        System.out.println("wait all child thread over");
        countDownLatch.await();
        System.out.println("all child thread over");
        executorService.shutdown();
    }
}

```

<h2 id='2'>CountDownLatch和join方法的区别</h2>
一个区别是，调用子线程的join()方法后，该线程会一直被阻塞到子线程运行完毕，而CountDownLatch则使用计数器来允许子线程运行完毕，或者在运行中递减计数，也就是CountDownLatch可以在子线程运行的任何时候让await方法返回而不一定等到运行结束。另外使用线程池来管理线程时一般都是直接添加Runable到线程池，这时候就没办法再调用线程的join方法了，就是说countDownLatch相比join方法让我们对线程同步有更灵活的控制。

<h2 id='3'>实现原理探究</h2>
从CountDownLatch的名字可以猜测内部有计数器，并且计数器是递减的。下面会讨论JDK开发组在何时初始化计数器，在何时递减计数器，当计数器变为0时做了什么操作，多个线程是如何通过计时器值实现同步的。为了一览CountDownLatch的内部结构，先看类图结构

![image-20191113185256449](/Users/hzx/Library/Application Support/typora-user-images/image-20191113185256449.png)

CountDownLatch使用AQS实现的，把计数器的值赋给了AQS状态变量state，也就是这里使用AQS的状态值来表示计数器值。

```java
public CountDownLatch(int count){
	if(count < 0) throw new IllegalArgumentException("count <0");
  this.sync = new Sync(count);
  
  sync(int count){
    setState(count);
  }
}
```

下面来看CountDownLatch中几个重要的方法，看看它们是如何调用AQS来实现功能的。

<h3 id='4'>void await()方法</h3>
线程调用await()方法后被阻塞，直到下面两个情况才会返回：

1.所有线程都调用了CountDownLatch对象的countDown方法后（计数器值为0）

2.其他线程调用了当前线程的interrupt()方法中断了当前线程抛出异常

下面看await()方法内部如何调用AQS方法

```java
public void await()throws InterruptedException{
	sync.acquireShareInterruptibly(1);
}

public final void acquireSharedInterruptibly(int arg) throws InterruptedException{
  	//如果线程被中断则抛出异常
  if(Thread.interrupted()) throw new InterrutedException();
  //查看当前计数器值是否为0，为0则直接返回，否则进入AQS的队列等待
  if(tryAcquireShared(arg)<0)doAcquireSharedInterruptibly(arg);
  
  //sync类实现的AQS的接口,传入的参数没有被用到，调用方法仅仅是为了检查当前状态值是不是为0，并没有调用CAS让当前的状态值减1
  protected int tryAcquireShared(int acquires){
    return (getState()==0)? 1:-1);
  }
}
```

<h3 id='5'>boolean await(long timeout, TimeUnit unit)</h3>
比起上面的await方法，多了一种返回的情况：当线程调用了CountDownLatch对象的countDown方法后，如果设置的timeout时间到了，会因为超时而返回false；

```java
public boolean await(long timeout, TimeUnit unit) throws InterruptedException {
  return sync.tryAcquireSharedNanos(1,unit.toNanos(timeout))
}
```

<h3 id='6'>void countDown()</h3>
线程调用该方法后，计数器的值递减，递减后如果计数器的值为0则唤醒所有因调用await方法而被阻塞的线程，否则什么都不做。下面看下countDown()方法是如何调用AQS方法的。

```java
//CountDownLatch的countDown()方法
public void countDown(){
  //委托sync调用AQS的方法
  sync.releaseShared(1);
}

//AQS的方法
public final boolean releaseShared(int arg){
  //调用sync实现的tryRealeaseShared
  if(tryRealeaseShared(arg)){
    //AQS的释放资源方法
    doRealseaseShared();
    return true;
  }
  return false;
}

//sync方法
protected boolean tryRealeaseShared(int releases){
  //循环进行CAS，直到当前线程成功完成CAS使得计数器值（状态值state）减1并更新到state 
  for(;;){
		int c = getState();
    //如果当前状态值为0则直接返回(1)
    if(c==0)return false;
    
    //使用CAS让计数器值减1 (2)
    int nextc = c-1;
    if(compareAndSetState(c,nextc)){
      return nextc == 0;
    }
  }
}
```

代码(1)判断如果当前状态值为0，则直接返回false，从而countDown()方法直接返回；否则执行代码(2)使用CAS将计数器值减1，CAS失败则循环重试，否则如果当前计数器值为0返回true，返回true说明是最后一个线程调用的countdown方法，那么该线程除了让计数器值减1外，还需要唤醒因调用CountDownLatch的await方法而被阻塞的线程，具体是调用AQS的doReleaseShared方法来激活阻塞的线程。这里代码(1)不是多余的，代码(1)防止当计数器变为0后，其他线程调用了countDown方法，否则状态值可能为负数

<h3 id='7'>long getCount()</h3>
获取当前计数器的值，也就是AQS的值，一般在测试时使用

```java
public long getCount(){
	return sync.getCount();
}

int getCount(){
	return getState();
}
```

<h3 id='8'>小结</h3>
本节首先介绍了 CountDownLatch 的使用，相比使用 join方法来实现线程间同步，前者更具有灵活性和方便性 。另外还介绍了 CountDownLatch 的原理， CountDownLatch是使用 AQS 实现的。使用 AQS 的状态变量来存放计数器 的值 。首先在 初始化CountDownLatch 时设置状态值(计数器值)，当多个线程调用 countdown 方法时实际是原子性递减 AQS 的状态值。当线程调用 await方法后当前线程会被放入 AQS 的阻塞队列等待计数器为 0再返回。其他线程调用 countdown方法让计数器值递减1，当计数器值变为0时， 当前线程还要调用 AQS 的doReleaseShared方法来激活由于调用 await()方法而被阻塞的线程。

<h2 id='9'>回环屏障CyclicBarrier原理探究</h2>
上节介绍的CountDownLatch在解决多个线程同步方面相对于调用线程的join方法已经有不小的优化，但是CountDownLatch的计数器是一次性的，也就是等到计数器值变为0后，在调用CountDownLatch的await和countdown方法都会理科返回，这就起不到线程同步的效果了。所以为了满足计数器可以重置的需要， JDK 开发组提供了 CyclicBarrier 类 ， 并 且 CyclicBarrier 类的功能 不限于 CountDownLatch 的功能 。从字面意思理解， CyclicBarrier 是回环屏障的意思 ，它可以让一组线程全部达到一个状态后再全部同时执行 。 这里之所以叫作回环 是因为当所有等待线程执行完毕，并重置 CyclicBarrier 的状态后它可以被重用。之所以叫作屏障是因为线程调用 await方法后就会被阻塞，这个阻塞点就称为屏障点，等所有线程都调用了 await方法后，线程们就会冲破屏障，继续向下运行。 

<h3 id='10'>
  案例介绍
</h3>

```java
public class CycleBarrierTest1 {
    //创建一个CyclicBarrier实例，添加一个所有子线程全部到达屏障后执行的任务
    private static CyclicBarrier cyclicBarrier = new CyclicBarrier(2, new Runnable() {
        @Override
        public void run() {
            System.out.println(Thread.currentThread()+"task1 merge result");
        }
    });

    public static void main(String []args){
        //创建一个线程个数固定为2的线程池
        ExecutorService executorService = Executors.newFixedThreadPool(2);
        //将线程A添加到线程池
        executorService.submit(new Runnable() {
            @Override
            public void run() {
                try {
                    System.out.println(Thread.currentThread() + "task1-1");
                    System.out.println(Thread.currentThread() + "enter in barrier");
                    cyclicBarrier.await();
                    System.out.println(Thread.currentThread() + "enter out barrier");
                }catch (Exception e){
                    e.printStackTrace();
                }
            }
        });
        //将线程B添加到线程池
        executorService.submit(new Runnable() {
            @Override
            public void run() {
                try {
                    System.out.println(Thread.currentThread() + "task1-2");
                    System.out.println(Thread.currentThread() + "enter in barrier");
                    cyclicBarrier.await();
                    System.out.println(Thread.currentThread() + "enter out barrier");
                }catch (Exception e){
                    e.printStackTrace();
                }
            }
        });
        executorService.shutdown();

    }
}
```

输出结果为

```java
Thread[pool-1-thread-1,5,main]task1-1
Thread[pool-1-thread-1,5,main]enter in barrier
Thread[pool-1-thread-2,5,main]task1-2
Thread[pool-1-thread-2,5,main]enter in barrier
Thread[pool-1-thread-2,5,main]task1 merge result
Thread[pool-1-thread-2,5,main]enter out barrier
Thread[pool-1-thread-1,5,main]enter out barrier
```

上述代码创建了一个CyclicBarrier对象，其第一个参数为计数器初始值，第二个参数Runable是当计数器值为0时需要执行的任务。在main函数里面首先创建了一个大小为 2 的线程池，然后添加两个子任务到线程池， 每个子任务在执行完 自己的逻辑后会调 用 await 方法 。一开始计数器值为 2， 当第一个线程调用 await 方法时，计数器值会递减为1。 由于此时计数器值不为0，所以当前线程就到了屏障点而被阻塞。 然后第二个线程调用 await 时，会进入屏障，计数器值也会递减，现在计数器值为 0， 这时就会去执行 CyclicBanier 构造函数中的任务，执行完毕后退出屏障点，并且唤醒被阻塞的第二个线程， 这时候第一个线程也会退出屏障点继续向下运行 。 

上面的例子说明了多个线程之间是相互等待的，假如计数器值为 N，那么随后调用 await 方法的 N-1 个线程都会因为到达屏障点而被阻塞 ，当第 N 个线程调用 await 后，计数器值为 0 了，这时候第 N个线程才会发出通知唤醒前面的 N-1个线程。 也就是当全部线程都到达屏障点时才能一块继续向下执行 。 对于这个例子来说，使用 CountDownLatch 也可以得到类似的输出结果。下面再举个例子来说明 CyclicBaiTier 的可复用性。

假设一个任务由阶段1、阶段2、阶段3组成，每个线程要串行地执行阶段1、阶段2和阶段3，当多个线程执行该任务时，必须要保证所有线程的阶段1全部完成后才能进入阶段2执行，当所有线程的阶段2全部完成后才能进入阶段3执行。下面使用CycliBarrier来完成这个需求。  

```java
public class CycleBarrierTest2 {
    private static CyclicBarrier cyclicBarrier= new CyclicBarrier(2);

    public static void main(String[]args)throws  InterruptedException{
        ExecutorService executorService = Executors.newFixedThreadPool(2);
        executorService.submit(new Runnable() {
            @Override
            public void run() {
                try {
                    System.out.println(Thread.currentThread() + "step1");
                    cyclicBarrier.await();
                    System.out.println(Thread.currentThread() + "step2");
                    cyclicBarrier.await();
                    System.out.println(Thread.currentThread()+"step3");
                }catch (Exception e){
                    e.printStackTrace();
                }
            }
        });

        executorService.submit(new Runnable() {
            @Override
            public void run() {
                try {
                    System.out.println(Thread.currentThread() + "step1");
                    cyclicBarrier.await();
                    System.out.println(Thread.currentThread() + "step2");
                    cyclicBarrier.await();
                    System.out.println(Thread.currentThread()+"step3");
                }catch (Exception e){
                    e.printStackTrace();
                }
            }
        });
        executorService.shutdown();
    }
}

```

在如上代码中，每个子线程在执行完阶段1后都调用了 await 方法，到达屏障点后才会一块往下执行，这就保证了所有线程都完成了阶段1后才会开始执行阶段 2。然后在阶段 2 后面调用了 await 方法，这保证了所有线程都完成了阶段 2 后，才能开始阶段 3 的执行 。 这个功能使用单个CountDownLatch 是无法完成的 。

<h3 id='11'>实现原理探究</h3>
![img](https://github.com/afkbrb/java-concurrency-note/raw/master/images/14.png)

由类图可知，CyclicBarrier基于独占锁实现，本质的底层还是基于AQS。parites用来记录线程个数，这里表示多少线程调用await后，所有线程才会冲破屏障继续往下运行。而count一开始等于parites，每当有线程调用await方法就递减1，当count为0时就表示所有线程都到了屏障点。由于CyclicBarrier是可以被复用的，当count的值变为0后，会将paties的值赋予给count，进行复用。

lock用于保证更新计数器count的原子性。lock的条件变量trip用于支持线程间使用await和signalAll进行通信。

以下是CyclicBarrier的构造函数：

```java
public CyclicBarrier(int parties) {
    this(parties, null);
}

// barrierAction为达到屏障点（parties个线程调用了await方法）时执行的任务
public CyclicBarrier(int parties, Runnable barrierAction) {
    if (parties <= 0) throw new IllegalArgumentException();
    this.parties = parties;
    this.count = parties;
    this.barrierCommand = barrierAction;
}
```

Generation的定义如下：

```java
private static class Generation {
    // 记录当前屏障是否可以被打破
    boolean broken = false;
}
```

<h2 id='11'>源码分析</h2>
<h3 id='12'>int await()</h3>
当前线程调用该方法时会阻塞，直到满足以下条件之一才会返回：

- parties个线程调用了await方法，也就是到达屏障点
- 其他线程调用了当前线程的interrupt方法
- Generation对象的broken标志被设置为true，抛出BrokenBarrierExecption

```java
public int await() throws InterruptedException, BrokenBarrierException {
    try {
        // false表示不设置超时时间，此时后面参数无意义
        // dowait稍后具体分析
        return dowait(false, 0L);
    } catch (TimeoutException toe) {
        throw new Error(toe); // cannot happen
    }
}
```

<h3 id='13'>
  boolean await(long timeout, TimeUnit unit)方法
</h3>

相比于await()，等待超时会返回false。

```java
public int await(long timeout, TimeUnit unit)
    throws InterruptedException,
            BrokenBarrierException,
            TimeoutException {
           // 设置了超时时间
           // dowait稍后分析     
    return dowait(true, unit.toNanos(timeout));
}
```

<h3 id='14'>
int dowait(boolean timed, long nanos)</h3>
```java
private int dowait(boolean timed, long nanos)
    throws InterruptedException, BrokenBarrierException,
            TimeoutException {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        final Generation g = generation;
        // 屏障已被打破则抛出异常
        if (g.broken)
            throw new BrokenBarrierException();

        // 线程中断则抛出异常
        if (Thread.interrupted()) {
            // 打破屏障
            // 会做三件事
            // 1. 设置generation的broken为true
            // 2. 重置count为parites
            // 3. 调用signalAll激活所有等待线程
            breakBarrier();
            throw new InterruptedException();
        }
			//(1)index==0说明所有线程都到达了屏障点，此时执行初始化时传递的任务
        int index = --count;
        if (index == 0) {
            boolean ranAction = false;
            try {
                final Runnable command = barrierCommand;
                if (command != null)
                    //(2)执行每一次到达屏障点所需要执行的任务
                    command.run();
                ranAction = true;
                //(3)激活其他因调用await方法而被阻塞的线程，重置状态，进入下一次屏障
                nextGeneration();
                return 0;
            } finally {
                if (!ranAction)
                    breakBarrier();
            }
        }

        // (4)如果index不为0
        // loop until tripped, broken, interrupted, or timed out
        for (;;) {
            try {
              //(5)没有设置赶超时间
                if (!timed)
                    trip.await();
              //(6)设置了赶超时间
                else if (nanos > 0L)
                    nanos = trip.awaitNanos(nanos);
            } catch (InterruptedException ie) {
                // 执行此处时，有可能其他线程已经调用了nextGeneration方法
                // 此时应该使当前线程正常执行下去
                // 否则打破屏障
                if (g == generation && ! g.broken) {
                    breakBarrier();
                    throw ie;
                } else {
                    // We're about to finish waiting even if we had not
                    // been interrupted, so this interrupt is deemed to
                    // "belong" to subsequent execution.
                    Thread.currentThread().interrupt();
                }
            }

            if (g.broken)
                throw new BrokenBarrierException();
            // 如果此次屏障已经结束，则正常返回
            if (g != generation)
                return index;
            // 如果是因为超时，则打破屏障并抛出异常
            if (timed && nanos <= 0L) {
                breakBarrier();
                throw new TimeoutException();
            }
        }
    } finally {
        lock.unlock();
    }
}

// 打破屏障
private void breakBarrier() {
    // 设置打破标志
    generation.broken = true;
    // 重置count
    count = parties;
    // 唤醒所有等待的线程
    trip.signalAll();
}

private void nextGeneration() {
    // (7)唤醒当前屏障下所有被阻塞的线程
    trip.signalAll();
    // (8)重置状态，进入下一次屏障
    count = parties;
    generation = new Generation();
}
```

以上的dowait方法主干代码。当一个线程调用了dowait方法后，首先会获取独占锁lock，如果创建CyclicBarrier时传递的参数为10，那么后面9个调用线程会被阻塞，然后当前获取到锁的线程会对计数器count进行递减操作，递减后count=index=9，因为index!=0所以当前线程会执行代码(4)，如果当前线程调用的是无参数的await()方法，则timed=false，所以当前线程会被放入条件变量trip的条件阻塞队列，当前线程会被挂起并释放获取的lock锁。如果调用的是有参数的await方法则timed=true，然后线程也会被放入条件变量的条件队列并释放锁资源，不同的是当前线程会在指定时间超时后自动被激活

当第一个获取锁的线程由于被阻塞释放锁后，被阻塞的9个线程中会有一个竞争到lock锁，然后执行与第一个线程同样的操作，知道最后一个线程获取到lock锁，此时已经有9个线程被放入了条件变量trip的条件队列里面，最后count=index等于0，所以执行代码(2)，如果创建CyclicBarrier时传递了任务，则再其他线程被唤醒前先执行任务，任务执行完毕后再执行代码(3)，唤醒其他9个线程，并充值CyclicBarrier，然后这10个线程就可以继续向下运行了。

<h3 id='15'>小结</h3>
本节通过案例说明CyclicBarrier与CountDownLatch的不同在于，前者可以复用，并且前者特备适合分配任务有序执行的场景，然后分析了CycleBarrier，其通过独占锁ReentrantLock实现计数器原子性更新，并使用条件变量队列来实现线程同步。

<h2 id='16'>信号量Semaphore原理探究</h2>
Semaphore信号量也是Java中的一个同步器，与CountDownLatch和CyclicBarrier不同的是，他的内部计数器是递增的，并且在一开始初始化Semaphore可以指定一个初始值，但是并不需要知道需要同步的线程个数，而是在需要同步的地方调用acquire方法时指定需要同步的线程个数。

```java
public class SemaphoreTest {
    //创建Semaphore实例
    private static Semaphore semaphore = new Semaphore(0);

    public static void main(String[]arsg) throws InterruptedException{
        ExecutorService executorService = Executors.newFixedThreadPool(2);
      	//将线程A添加到线程池
        executorService.submit(new Runnable() {
            @Override
            public void run() {
                    try{
                        System.out.println(Thread.currentThread()+"over");
                        semaphore.release();
                    }catch (Exception e){
                        e.printStackTrace();
                    }
            }
        });
      	//将线程B添加到线程池
        executorService.submit(new Runnable() {
            @Override
            public void run() {
                try{
                    System.out.println(Thread.currentThread()+"over");
                    semaphore.release();
                }catch (Exception e){
                    e.printStackTrace();
                }
            }
        });
      //等待子线程执行完毕，返回
        semaphore.acquire(2);
        System.out.println("all child thread over");
      //关闭线程池
        executorService.shutdown();
    }
}

```
如上代码首先创建了一个信号量实例，构造函数的入参为0，说明当前信号量计数器的值为 0。然后 main 函数向线程池添加两个线程任务，在每个线程内部调用信号量的release 方法，这相当于让计数器值递增1。最后在 main 线程里面调用信号量的 acquire 方法，传参为2说明调用 acquire方法的线程会一直阻塞， 直到信号量的计数变为 2才会返回

<h2 id='17'>类图结构</h2>
![img](https://github.com/afkbrb/java-concurrency-note/raw/master/images/16.png)

由图可知，Semaphore还是使用AQS实现的，并且可以选取公平性策略（默认为非公平性的）。

<h3 id='18'>void acquire()</h3>
表示当前线程希望获取一个信号量资源，如果当前信号量大于0，则当前信号量的计数减1，然后该方法直接返回。否则如果当前信号量等于0，则被阻塞。

```java
public void acquire() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}

public final void acquireSharedInterruptibly(int arg)
        throws InterruptedException {
    // 可以被中断
    if (Thread.interrupted())
        throw new InterruptedException();
    // 调用Sync子类方法尝试获取，这里根据构造函数决定公平策略
    if (tryAcquireShared(arg) < 0)
        // 将当前线程放入阻塞队列，然后再次尝试
        // 如果失败则挂起当前线程
        doAcquireSharedInterruptibly(arg);
}
```

tryAcquireShared由Sync的子类实现以根据公平性采取相应的行为。

以下是非公平策略NofairSync的实现:

```java
protected int tryAcquireShared(int acquires) {
    return nonfairTryAcquireShared(acquires);
}

final int nonfairTryAcquireShared(int acquires) {
    for (;;) {
        int available = getState();
        int remaining = available - acquires;
        // 如果剩余信号量小于0直接返回
        // 否则如果更新信号量成功则返回
        if (remaining < 0 ||
            compareAndSetState(available, remaining))
            return remaining;
    }
}
```

下面是公平性的实现：

```java
protected int tryAcquireShared(int acquires) {
    for (;;) {
        // 关键在于先判断AQS队列中是否已经有元素要获取信号量
        if (hasQueuedPredecessors())
            return -1;
        int available = getState();
        int remaining = available - acquires;
        if (remaining < 0 ||
            compareAndSetState(available, remaining))
            return remaining;
    }
}
```

hasQueuedPredecessors方法（可参看第六章）用于判断当前线程的前驱节点是否也在等待获取该资源，如果是则自己放弃获取的权限，然后当前线程会被放入AQS中，否则尝试去获取。

<h3 id='19'>void acquire(int permits)</h3>

可获取多个信号量。

```java
public void acquire(int permits) throws InterruptedException {
    if (permits < 0) throw new IllegalArgumentException();
    sync.acquireSharedInterruptibly(permits);
}
```

<h3 id='20'>void acquireUninterruptibly()
</h3>

不对中断进行响应。

```java
public void acquireUninterruptibly() {
    sync.acquireShared(1);
}
```

<h3 id='21'>void acquireUninterruptibly(int permits)
</h3>

不对中断进行相应并且可获取多个信号量。

```java
public void acquireUninterruptibly(int permits) {
    if (permits < 0) throw new IllegalArgumentException();
    sync.acquireShared(permits);
}
```

<h3 id='22'>void release()</h3>

使信号量加1，如果当前有线程因为调用acquire方法被阻塞而被放入AQS中的话，会根据公平性策略选择一个信号量个数能被满足的线程进行激活。

```java
public void release() {
    sync.releaseShared(1);
}

public final boolean releaseShared(int arg) {
    // 尝试释放资源（增加信号量）
    if (tryReleaseShared(arg)) {
        // 释放资源成功则根据公平性策略唤醒AQS中阻塞的线程
        doReleaseShared();
        return true;
    }
    return false;
}

protected final boolean tryReleaseShared(int releases) {
    for (;;) {
        int current = getState();
        int next = current + releases;
        if (next < current) // overflow
            throw new Error("Maximum permit count exceeded");
        if (compareAndSetState(current, next))
            return true;
    }
}
```

<h3 id='22'>void release(int permits)</h3>