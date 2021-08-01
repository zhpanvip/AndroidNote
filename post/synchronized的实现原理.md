
## synchronized使用

synchronized可以修饰方法或者同步代码块，主要确保多个线程在同一时刻只能有一个线程处于该方法或者同步代码块中，它保证了线程对变量访问的可见性和排他性。

1.修饰代码块，对指定的对象加锁
```
public void add() {
    synchronized (this) {
        i++;
    }
}
```

2.修饰实例方法，对当前实例对象this加锁

```
public synchronized void add(){
       i++;
}
```

3.修饰静态方法，对该类的Class对象加锁

```
public static synchronized void add(){
       i++;
}
```

要注意synchronized关键字是一个对象锁，无论怎么使用，它一定是对某个对象加锁。

## Java对象头与monitor对象

Java中对象由三部分构成，分别为对象头、实例变量、填充字节。
- (1) 实例数据：存放类的属性数据信息，包括父类的属性信息，这部分内存按4字节对齐。
- (2) 填充数据：由于虚拟机要求对象起始地址必须是8字节的整数倍。填充数据不是必须存在的，仅仅是为了字节对齐。
- (3) 对象头：它是实现synchronized的锁对象的基础，这点我们重点讨论它。

synchronized锁是存储在Java对象头中的，JVM中采用两个字节来存储对象头，以Hotspot为例，对象头主要包括三部分：Mark Word（标记字段）、Klass Pointer（类型指针）以及数组数组长度（只有数组对象有）。其中Mark Word用于存放对象自身的运行时数据。

Mark Word主要用于存储对象自身的运行时数据，如哈希码、GC分代年龄、锁状态、线程持有的锁、偏向线程ID、偏向时间戳等。Mark Word同时也记录了对象和锁有关的信息。当这个对象被synchronized关键字当成同步锁时，围绕这个锁的一系列操作都和Mark Word有关。Mark Word在不同的锁状态下存储的内容不同，在32位JVM中的存储内容如下图，其中无锁和偏向锁的标记状态都是01，只是在前面的1bit区分了这是无锁状态还是偏向锁状态。
![](https://img-blog.csdnimg.cn/20191022155910999.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTI3MjM2NzM=,size_16,color_FFFFFF,t_70)

可以看到，当对象状态为偏向锁（biasable）时，mark word存储的是偏向的线程ID；当状态为轻量级锁（lightweight locked）时，mark word存储的是指向线程栈中Lock Record的指针；当状态为重量级锁（inflated）时，为指向堆中的monitor对象的指针。


## monitor对象

 由上述内容可知，重量级锁synchronized的标识位是10，其中指针指的是monitor对象(也称为管程或监视器锁)的起始地址。每个对象实例都会有一个 monitor与之关联。monitor既可以与对象一起创建、销毁；也可以在线程试图获取对象锁时自动生成。但当一个 monitor 被某个线程持有后，它便处于锁定状态。

  在Java虚拟机(HotSpot)中，monitor是由ObjectMonitor实现的，其主要数据结构如下（位于HotSpot虚拟机源码ObjectMonitor.hpp文件中，使用C++实现）

```
ObjectMonitor() {
    _header       = NULL;
    _count        = 0; //记录个数
    _waiters      = 0,
    _recursions   = 0;
    _object       = NULL;
    _owner        = NULL;
    _WaitSet      = NULL; //处于wait状态的线程，会被加入到_WaitSet
    _WaitSetLock  = 0 ;
    _Responsible  = NULL ;
    _succ         = NULL ;
    _cxq          = NULL ;
    FreeNext      = NULL ;
    _EntryList    = NULL ; //处于获取锁失败的线程，会被加入到该列表
    _SpinFreq     = 0 ;
    _SpinClock    = 0 ;
    OwnerIsThread = 0 ;
  }
```

## synchronized实现原理

### 1.同步代码块
synchronized底层是通过monitorenter和moniterexit指令实现的，monitorenter指令是在编译后插入到同步代码块的开始位置，而monitorexit是插入到方法结束和异常处。
通过javap的工具对上述同步代码块进行反汇编
```
public void add() {
    synchronized (this) {
        i++;
    }
}
```
可以看到字节码指令：

```
public class com.zhangpan.text.TestSync {
  public com.zhangpan.text.TestSync();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public void add();
    Code:
       0: aload_0
       1: dup
       2: astore_1
       3: monitorenter    // synchronized关键字的入口
       4: getstatic     #2                  // Field i:I
       7: iconst_1
       8: iadd
       9: putstatic     #2                  // Field i:I
      12: aload_1
      13: monitorexit  // synchronized关键字的出口
      14: goto          22
      17: astore_2
      18: aload_1
      19: monitorexit // synchronized关键字的出口
      20: aload_2
      21: athrow
      22: return
    Exception table:
       from    to  target type
           4    14    17   any
          17    20    17   any
}

```
当代码执行到monitorenter 指令时，将会尝试获取该对象对应的Monitor的所有权，即尝试获得该对象的锁。当该对象的 monitor 的进入计数器为 0，那线程可以成功取得 monitor，并将计数器值设置为 1，取锁成功。如果当前线程已经拥有该对象monitor的持有权，那它可以重入这个 monitor ，计数器的值也会加 1。与之对应的执行monitorexit指令时，锁的计数器会减1。倘若其他线程已经拥有monitor 的所有权，那么当前线程获取锁失败将被阻塞并进入到_WaitSet 中，直到等待的锁被释放为止。也就是说，当所有相应的monitorexit指令都被执行，计数器的值减为0，执行线程将释放 monitor(锁)，其他线程才有机会持有 monitor 。

需要注意的是，字节码中有两个monitorexit指令，因为编译器需要确保方法中调用过的每条monitorenter指令都有执行对应的monitorexit 指令，而无论这个方法是正常结束还是异常结束。为了保证在方法异常时，monitorenter和monitorexit指令也能正常配对执行，编译器会自动产生一个可以处理所有异常的异常处理器，它的目的就是用来执行异常的monitorexit指令。而字节码中多出的monitorexit指令，就是异常结束时用来释放monitor的指令。

上述过程可以总结如下：
- (1) 如果monitor的进入数为0，则该线程进入monitor，然后将进入数设置为1，该线程即为monitor的所有者。
- (2) 如果线程已经占有该monitor，只是重新进入，则进入monitor的进入数加1.
- (3) 如果其他线程已经占用了monitor，则该线程进入阻塞状态，直到monitor的进入数为0，再重新尝试获取monitor的所有权。

### 2.同步方法的实现
```
public synchronized void add(){
       i++;
}
```

通过javap -v 获取上面代码字节码的附件信息，得到如下结果：

```
 public synchronized void add();
    descriptor: ()V
    flags: (0x0021) ACC_PUBLIC, ACC_SYNCHRONIZED
    Code:
      stack=3, locals=1, args_size=1
         0: aload_0
         1: dup
         2: getfield      #2                  // Field i:I
         5: iconst_1
         6: iadd
         7: putfield      #2                  // Field i:I
        10: return
      LineNumberTable:
        line 5: 0
        line 6: 10

```

从字节码中可以看出，synchronized修饰的方法并没有monitorenter指令和monitorexit指令，取得代之的确实是ACC_SYNCHRONIZED标识，该标识指明了该方法是一个同步方法，JVM通过该ACC_SYNCHRONIZED访问标志来辨别一个方法是否声明为同步方法，从而执行相应的同步调用。这便是synchronized锁在同步代码块和同步方法上实现的基本原理。

## Java虚拟机对synchronized的优化

需要注意的是，在Java早期版本中，synchronized属于重量级锁，效率低下。这是因为在实现上，JVM会阻塞未获取到锁的线程，直到锁被释放的时候才唤醒这些线程。阻塞和唤醒操作是依赖操作系统来完成的，所以需要从用户态切换到内核态，开销很大。并且monitor调用的是操作系统底层的互斥量(mutex)，本身也有用户态和内核态的切换。Java 6之后，为了减少获得锁和释放锁所带来的性能消耗，引入了轻量级锁和偏向锁，接下来我们将简单介绍一下Java官方在JVM层面对synchronized锁的优化。
  
  ### 1.Java6之后synchronized引入的多种锁机制

  锁的状态总共有四种：无锁状态、偏向锁、轻量级锁和重量级锁。随着锁的竞争，锁可以从偏向锁升级到轻量级锁，再升级到重量级锁，但是锁的升级是单向的，也就是说只能从低到高升级，不会出现锁的降级。前面已经详细分析过重量级锁，下面将介绍偏向锁和轻量级锁以及JVM的其他优化手段。

- 1.偏向锁
  偏向锁是Java 6之后加入的新锁，它是一种针对加锁操作的优化手段.经过研究发现，在大多数情况下，锁不仅不存在多线程竞争，而且总是被同一线程多次获得，因此为了减少同一线程获取锁(会涉及到一些CAS操作,耗时)的代价而引入偏向锁。偏向锁的核心思想是，如果一个线程获得了锁，那么锁就进入偏向模式，此时Mark Word 的结构也变为偏向锁结构，当这个线程再次请求锁时，无需再做任何同步操作，即可获取锁的过程，这样就省去了大量有关锁申请的操作，从而也就提升了程序的性能。所以，对于没有锁竞争的场合，偏向锁有很好的优化效果，毕竟极有可能连续多次是同一个线程申请相同的锁。但是对于锁竞争比较激烈的场合，偏向锁就失效了，因为这样场合极有可能每次申请锁的线程都是不相同的，因此这种场合下不应该使用偏向锁，否则会得不偿失，需要注意的是，偏向锁失败后，并不会立即膨胀为重量级锁，而是先升级为轻量级锁。

- 2.轻量级锁
  如果偏向锁失败，虚拟机并不会立即升级为重量级锁，它还会尝试使用一种称为轻量级锁的优化手段(Java1.6之后加入的)，此时Mark Word 的结构也变为轻量级锁的结构。轻量级锁能够提升程序性能的依据是“对绝大部分的锁，在整个同步周期内都不存在竞争”，注意这是经验数据。需要了解的是，轻量级锁所适应的场景是线程交替执行同步块的场合，如果存在同一时间访问同一锁的场合，就会导致轻量级锁膨胀为重量级锁。轻量级锁在实际没有锁竞争的情况下，将申请互斥量这步也省掉。

- 3.自旋锁
  轻量级锁失败后，虚拟机为了避免线程真实地在操作系统层面挂起，还会进行一项称为自旋锁的优化手段。这是基于在大多数情况下，线程持有锁的时间都不会太长，如果直接挂起操作系统层面的线程可能会得不偿失，毕竟操作系统实现线程之间的切换时需要从用户态转换到核心态，这个状态之间的转换需要相对比较长的时间，时间成本相对较高，因此自旋锁会假设在不久将来，当前的线程可以获得锁，因此虚拟机会让当前想要获取锁的线程做几个空循环(这也是称为自旋的原因)，一般不会太久，可能是50个循环或100循环，在经过若干次循环后，如果得到锁，就顺利进入临界区。如果还不能获得锁，那就会将线程在操作系统层面挂起，这就是自旋锁的优化方式，这种方式确实也是可以提升效率的。最后没办法也就只能升级为重量级锁了。
  自旋会跑一些无用的CPU指令，所以会浪费处理器时间，如果锁被其他线程占用的时间短的话确实是合适的。但是如果长的话就不如直接使用阻塞。那么JVM怎么知道锁被占用的时间到底是长还是短呢？因为JVM不知道锁被占用的时间长短，所以使用的是自适应自旋。就是线程空循环的次数时会动态调整的。可以看出，自旋会导致不公平锁，不一定等待时间最长的线程会最先获取锁。

- 4.锁消除
  消除锁是虚拟机另外一种锁的优化，这种优化更彻底，Java虚拟机在JIT编译时(可以简单理解为当某段代码即将第一次被执行时进行编译，又称即时编译)，通过对运行上下文的扫描，去除不可能存在共享资源竞争的锁，通过这种方式消除没有必要的锁，可以节省毫无意义的请求锁时间。例如，StringBuffer的append是一个同步方法，但是在add方法中的StringBuffer属于一个局部变量，并且不会被其他线程所使用，因此StringBuffer不可能存在共享资源竞争的情景，JVM会自动将其锁消除。


### 2.synchronized关键字锁升级过程

- (1）当没有被当成锁时，这就是一个普通的对象，Mark Word记录对象的HashCode，锁标志位是01，是否偏向锁那一位是0;
- (2）当对象被当做同步锁并有一个线程A抢到了锁时，锁标志位还是01，但是否偏向锁那一位改成1，前23bit记录抢到锁的线程id，表示进入偏向锁状态;
- (3) 当线程A再次试图来获得锁时，JVM发现同步锁对象的标志位是01，是否偏向锁是1，也就是偏向状态，Mark Word中记录的线程id就是线程A自己的id，表示线程A已经获得了这个偏向锁，可以执行同步中的代码;
- (4) 当线程B试图获得这个锁时，JVM发现同步锁处于偏向状态，但是Mark Word中的线程id记录的不是B，那么线程B会先用CAS操作试图获得锁，这里的获得锁操作是有可能成功的，因为线程A一般不会自动释放偏向锁。如果抢锁成功，就把Mark Word里的线程id改为线程B的id，代表线程B获得了这个偏向锁，可以执行同步代码。如果抢锁失败，则继续执行步骤5;
- (5) 偏向锁状态抢锁失败，代表当前锁有一定的竞争，偏向锁将升级为轻量级锁。JVM会在当前线程的线程栈中开辟一块单独的空间，里面保存指向对象锁Mark Word的指针，同时在对象锁Mark Word中保存指向这片空间的指针。上述两个保存操作都是CAS操作，如果保存成功，代表线程抢到了同步锁，就把Mark Word中的锁标志位改成00，可以执行同步代码。如果保存失败，表示抢锁失败，竞争太激烈，继续执行步骤6;
- (6) 轻量级锁抢锁失败，JVM会使用自旋锁，自旋锁不是一个锁状态，只是代表不断的重试，尝试抢锁。从JDK1.7开始，自旋锁默认启用，自旋次数由JVM决定。如果抢锁成功则执行同步代码，如果失败则继续执行步骤7;
- (7) 自旋锁重试之后如果抢锁依然失败，同步锁会升级至重量级锁，锁标志位改为10。在这个状态下，未抢到锁的线程都会被阻塞。

https://juejin.cn/post/6844903726545633287

[深入理解Java中synchronized关键字的实现原理](https://blog.csdn.net/u012723673/article/details/102681942)

[死磕synchronized底层实现](https://juejin.cn/post/6844904196676780040)