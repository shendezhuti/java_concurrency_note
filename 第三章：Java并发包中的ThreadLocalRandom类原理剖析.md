# 第三章 Java并发包中ThreadLocalRandom类原理剖析

## 目录

- [Random及其局限性](#1)

  - [示例](#2)
  - [分析](#3)

- [ThreadLocalRandom](#4)

  - [示例](#5)
  - [原理](#6)
  - [Unsafe机制](#7)
  - [ThreadLocalRandom current()方法](#8)
  - [int nextInt(int bound)](#9)

  ThreadLocalRandom类是JDK7在JUC包下新增的随机数生成器，它弥补了Random类在多线程下的缺陷。本章讲解为何要在JUC下新增该类，以及该类的实现原理。

<h2 id='1'>Random及其局限性</h2>

一般情况我们可以用java.util.Random来实现生成随机数（java.lang.Math中的随机数生成也是使用Random实例生成随机数）

<h3 id='2'>示例</h3>

```java
public static void main(String[]args){
	Random random=new Random();
  for(int i=0;i<10;i++){
    System.out.println(random.nextInt(10));
  }
}
```

<h3 id='3'>分析</h3>

下面以nextInt(int bound)方法为例来分析Random的源码

```java
public int nextInt(int bound) {
    //边界检测
    if (bound <= 0)
        throw new IllegalArgumentException(BadBound);

    //获取下一随机数
    int r = next(31);

    //(*)此处以特定算法根据r计算出最终结果
    ...

    return r;
}

protected int next(int bits) {
    long oldseed, nextseed;
    AtomicLong seed = this.seed;
    //CAS操作更新seed
    do {
        oldseed = seed.get();
        //根据老的种子计算新的种子
        nextseed = (oldseed * multiplier + addend) & mask;
    } while (!seed.compareAndSet(oldseed, nextseed));
    return (int)(nextseed >>> (48 - bits));
}
```

由此可见，新的随机数的生成需要两个步骤：

- 首先根据老的种子生成新的种子

- 然后根据新的种子来计算随机数

  

  在单线程下调用nextInt都是根据老的种子计算新的种子，可以保证随机性。

  但是在多线程的情况下，每个线程都可能拿同一个老种子去执行计算新种子，**如果**next方法因此返回相同的值的话，由于(*)处的算法是固定的，这会导致不同线程生成相同的随机数，这并非我们想要的。所以next方法使用CAS操作保证每次只有一个线程可以更新老的种子，失败的线程则重新获取，这样就解决了上述问题。

  但是这样做仍然有缺点：当多个线程同时计算随机数来计算新的种子时，多个线程会竞争同一个原子变量的更新操作，由于原子变量的更新是CAS操作，同时只有一个线程会成功，所以会造成大量线程进行自旋重试，这回降低并发的性能，所以ThreadLocalRandom应运而生。

<h2 id='2'>ThreadLocalRandom</h2>

<h3 id='5'>示例</h3>

```java
public class RandomTest{
  //(10)获取一个随机数生成器
	ThreadRandom random = ThreadLocalRandom.current();
  
  for(int i=0;i<10;i++){
    System.out.println(random.nextInt(5));
  }
}
```

<h3 id='6'>原理</h3>

Random的缺点在于多个线程会使用同一个原子性种子变量，从而导致对原子变量更新的竞争；而ThreadLocalRandom保证每个线程都维护一个种子变量，每个线程根据老的种子计算新的种子，并使用新种子更新老的种子，再根据新种子计算随机数，就不会存在竞争问题，这会提高并发的性能。

下面看看ThreadLocalRandom的主要代码实现逻辑

<h3 id='7'>Unsafe机制</h3>

```java
private static final sun.misc.Unsafe UNSAFE;
private static final long SEED;
private static final long PROBE;
private static final long SECONDARY;
static{
  try{
    //获取unsafe实例
    UNSAFE=sun.misc.Unsafe.getUnsage();
    Class<?>tk=Thread.class;
    //获取Thread类里面threadLocalRandomSeed变量在Thread实例里面的偏移量
    SEED=UNSAFE.objectFieldOffset(tk.getDeclaredField("threadLocalRandomSeed"));
    //获取Thread类里面threadLocalRandomProbe变量在Thread实例里面的偏移量
    PROBE= UNSAFE.objectFieldOffset(tk.getDeclaredField("threadLocalRandomSeed"));
    //获取Thread类里面threadLocalRandomSecondarySeed变量在Thread实例里面的偏移量，这个值在后面的讲解LongAdder时会用到
    SECONDARY = UNSAFE.objectFiledOffset(tk.getDeclaredField("threadlocalRandomSecondarySeed"));
  }catch(Exception e){
    throw new Error(e);
  }
}
```

<h3 id='8'>2.ThreadlocalRandom current()方法</h3>

该方法获取ThreadLocalRandom实例，并初始化调用线程中的threadLocalRandomSeed和threadLocalRandomProbe变量

```java
static final ThreadLocalRandom instance = new ThreadLocalRandom();

public static ThreadLocalRandom current() {
    //检测是否初始化过
    //PROBE为Thread类中threadLocalRandomProb偏移
    if (UNSAFE.getInt(Thread.currentThread(), PROBE) == 0)
        localInit();
    return instance;
}

static final void localInit() {
    int p = probeGenerator.addAndGet(PROBE_INCREMENT);
    int probe = (p == 0) ? 1 : p; // skip 0
    long seed = mix64(seeder.getAndAdd(SEEDER_INCREMENT));
    Thread t = Thread.currentThread();
    //SEED为Thread类中threadLocalRandomSeed内存偏移
    UNSAFE.putLong(t, SEED, seed);
    UNSAFE.putInt(t, PROBE, probe);
}
```

如果线程第一次调用current方法，就需要调用LocalInit()方法进行初始化设置threadLocalRandomProb和threadLocalRandomSeed变量

<h3 id='9'>3.int nextInt(int bound) </h3>

```java
public int nextInt(int bound) {
    if (bound <= 0)
        throw new IllegalArgumentException(BadBound);
    //根据当前Thread中的threadLocalRandomSeed变量生成新种子    
    int r = mix32(nextSeed());
    int m = bound - 1;
    if ((bound & m) == 0) // power of two
        r &= m;
    else { // reject over-represented candidates
        for (int u = r >>> 1;
                u + m - (r = u % bound) < 0;
                u = mix32(nextSeed()) >>> 1)
            ;
    }
    return r;
}

final long nextSeed() {
    Thread t; long r;
    //生成并存入新种子
    UNSAFE.putLong(t = Thread.currentThread(), SEED,
                    r = UNSAFE.getLong(t, SEED) + GAMMA);
    return r;
```

在上述代码中，首先使用r=UNSAFE.getLong（t,SEED)获取当前线程中threadLocalRandomSeed变量的值，然后在种子的基础上累加GAMMA值作为新种子，而后使用UNSAFE的putLong方法把新种子放入当前线程的threadLocalRandomSeed变量中。