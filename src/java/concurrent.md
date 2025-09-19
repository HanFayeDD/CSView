---
title: 并发
author: xmy
---

9.19

进程、线程的基础知识请查阅操作系统部分，这里只介绍Java中的线程哦~

### Java线程模型
在Java中，启动一个main函数就代表启动了一个**JVM进程**，**main函数所在的线程是这个进程的主线程**，该JVM进程中的所有线程共享该JVM的**堆和方法区**，同时JVM在每个线程创建时为其分配各自的**PC、虚拟机栈和本地方法栈**，每个线程有一个Thread对象维护其上下文。

![image-20210903102622040](https://pic.imgdb.cn/item/63f8b734f144a010079b07cb.jpg)

Java中线程有六个状态

| 状态          | 含义                                                         |
| ------------- | ------------------------------------------------------------ |
| NEW           | 线程对象刚被创建时的状态                                     |
| RUNNABLE      | 包含了操作系统中的运行和就绪状态，调用了start方法后就从NEW进入该状态 |
| BLOCKED       | 运行过程中需要调用的对象正在被其它线程占用时，进入该对象EntrySet里，同时转为BLOCKED状态 |
| WAITING       | 等待状态，进入该状态后需要等待被唤醒                         |
| TIMED_WAITING | 和等待状态一样，但若没人唤醒，超过时间参数自动唤醒           |
| TERMINATED    | 线程执行完毕，即run方法返回了                                |

![image-20210903120621985](https://i.loli.net/2021/09/12/MYFgLm8Q5nDfrCi.png)

哪个线程调用了**Object.wait()/线程对象.join()**，哪个线程就进入WAITING状态，同时该线程进入Object的Monitor的WaitSet里

哪个线程调用了**Object.notify()**，就把这个Object的Monitor的WaitSet里随机一个线程唤醒

哪个线程调用了**Object.notifyAll()**，就把Object的Monitor的WaitSet里所有线程唤醒

如果上面的方法带时间参数，就不是进入WAITING而是TIMED_WAITING状态

### 锁

并发控制中的锁一般有两种，**悲观锁**和**乐观锁**

> [!NOTE]
>
> | **特性**     | **乐观锁**                                                   | **悲观锁**                                      |
> | ------------ | ------------------------------------------------------------ | ----------------------------------------------- |
> | **基本假设** | 假设冲突很少发生，因此直接执行操作。只是在提交修改的时候去验证对应的资源是否被其它线程修改了。 | 假设冲突频繁发生，因此先加锁再操作。            |
> | **实现方式** | 基于 自旋重试+CAS（Compare-And-Swap）。                      | 基于锁机制（如 `synchronized` 或显式锁）。      |
> | **开销**     | 重试成本较高，但不涉及线程阻塞和上下文切换。                 | 线程阻塞会导致上下文切换，开销较大。            |
> | **适用场景** | 多读。竞争较少、冲突概率低的场景。                           | 多写。竞争较多、冲突概率高的场景。              |
> | **常见实现** | `Atomic` 类、`StampedLock` 乐观读等。                        | `synchronized`、`ReentrantLock`、`Semaphore` 等 |
>
> > #### CAS实现乐观锁：
> >
> > - V当前值
> > - E期待值
> > - N新的值
> >
> > 当且仅当 V 的值等于 E 时，CAS 通过原子方式用新值 N 来更新 V 的值。如果不等，说明已经有其它线程更新了 V，则当前线程放弃更新。
> >
> > 当高并发写多的场景，会多次失败与重试，可能会造成CPU飙升

> [!NOTE]
>
> - 悲观和乐观：是否需要先加锁再操作
> - 独占和共享：是否允许多个线程对同一资源访问
> - 公平和非公平：排队队列不为空时是否需要进入排队队列才能获取到锁
> - 可重入锁：递归锁。一个线程可以获取自己已经拥有的锁。`synchronized`、`ReentrantLock`都是。否则会死锁

#### synchronized

synchronized修饰的**方法或代码块**同一时间只能被一个线程执行

一般有三种使用方法：

1. 修饰实例方法：调用某对象的该方法前获取该对象实例的锁

2. 修饰静态方法：调用某对象的该方法前获取该类的锁。

   两个线程分别执行同一个对象synchronized修饰的实例方法和静态方法时不会发生互斥，因为锁的资源不同，一个锁了对象实例，一个锁了类。

3. 锁对象，修饰代码块：synchronized(对象的引用)锁的是对象实例，synchronized(类.class)锁的是类

尽量不要使用 synchronized(String a)  因为 JVM 中，字符串常量池具有缓存功能！

synchronized不能修饰构造方法，也没必要修饰，构造方法本身就是线程安全的

**底层原理**

尝试获取对象的monitor，monitor已被其他线程占用时，获取失败，该线程进入EntrySet。占有monitor时调用wait()进入WaitSet。调用notify()时从WaitSet里随机选一个线程唤醒，调用notifyAll时唤醒WaitSet里所有线程

#### AQS

> [!NOTE]
>
> #### AQS AbstractQueuedSynchronizer
>
> 构建锁和同步器的抽象类
>
> - 组成
>   - 一个共享状态变量
>   - 一个FIFO队列、双向链表
> - 支持公平锁和非公平锁
>
> - 模式
>
>   - 独占模式：ReentrantLock一个线程独占资源。都是唤醒等待队列中的第一个线程。
>
>     - 公平锁（不能抢占）：如果队列中有线程等待，当前线程必须排队。如果队列中没有线程等待，当前线程可以直接获取锁。性能较差、不会饥饿
>
>     - 非公平锁（可以抢占）：如果获取成功，则当前线程获取锁，无需进入待队列（即插队）。
>
>       只有当锁已经被占用时，当前线程才会进入等待队列。性能较优、会饥饿
>
>   - 共享模式：Semaphore多个线程可以共享资源。释放锁时唤醒多个线程（多个线程尝试共享资源）

AQS全称是AbstractQueuedSynchronizer，它是Java中用来构建锁和同步器的基础框架，可以用于实现诸如ReentrantLock、Semaphore、CountDownLatch等多种同步工具。

![image-20210906170322803](https://i.loli.net/2021/09/12/hCq9xuJva2zidsN.png)

AQS主要依赖于一个双向链表和一个volatile类型的整数state来实现同步控制。该整数state用来表示同步状态，一般情况下，state=0表示没有线程占用同步资源，state>0表示有线程占用同步资源，state<0表示同步资源已经被争用了多次，比如ReentrantLock可以允许一个线程多次获得锁，每次state值减一。

AQS的主要方法有下面几个：

- acquire()：该方法用来获取同步状态，如果同步状态被占用，则线程将被加入等待队列中。

- acquireInterruptibly()：与acquire()类似，但是该方法允许中断操作。

- tryAcquire()：该方法用来尝试获取同步状态，如果成功则返回true，否则返回false。

- release()：该方法用来释放同步状态，并唤醒等待队列中的线程。

- acquireShared()：该方法用来获取共享式同步状态，如果同步状态被占用，则线程将被加入等待队列中。

- releaseShared()：该方法用来释放共享式同步状态，并唤醒等待队列中的线程。

AQS实现同步的关键在于，它提供了一个基于FIFO队列的等待队列，通过将等待线程加入等待队列中，然后在释放同步状态的时候，从等待队列中唤醒等待线程，从而实现了同步机制。

AQS的实现主要有两种方式：独占式（Exclusive）和共享式（Shared）。独占式是指只有一个线程可以占用同步资源，比如ReentrantLock，而共享式是指多个线程可以同时占用同步资源，比如CountDownLatch。在AQS中，**这两种方式的实现是基本相同的，区别在于获取和释放同步状态的方式不同**。

以上是AQS的基本实现方式，它是Java中构建锁和同步器的核心框架，为各种同步工具的实现提供了强大的基础支持。
### 并发容器

Java中为保证并发下的线程安全，设计了一些并发容器，常见的有CopyOnWriteArrayList和ConcurrentHashMap
#### CopyOnWriteArrayList
CopyOnWriteArrayList的实现原理主要分为两个方面，一是利用可重入锁实现线程安全，二是通过复制数组实现读写分离。

> [!NOTE]
>
> - 写：获取锁，复制，修改，赋值引用，释放锁（**写时复制**）
> - 读：无需加锁。读写分离
>
> 读多写少、对数据实时一致性要求不高的场景。并发性较好，但是不能保证数据的实时一致性。

在CopyOnWriteArrayList中，每次写操作都会先获取可重入锁，然后将当前数组复制一份，进行修改后再将新的数组赋值给原来的引用。在修改完成后释放锁。由于读操作不会对原数组进行修改，所以读操作可以直接对原来的数组进行读取，无需加锁。这样就实现了**读写分离**的效果，可以在不影响正在进行的读操作的情况下进行写操作。

在CopyOnWriteArrayList的实现中，由于每次写操作都需要复制整个数组，所以写操作的性能比较低。但是在读多写少的场景中，CopyOnWriteArrayList的并发性能比较好，因为读操作不会加锁，可以同时进行。

需要注意的是，虽然CopyOnWriteArrayList是线程安全的，但是它并不能保证数据的实时一致性。由于写操作的结果只会对新的数组产生影响，所以在多线程环境中，读取到的数据可能不是最新的。因此，CopyOnWriteArrayList适用于读多写少且对实时性要求不高的场景。

#### ConcurrentHashMap

ConcurrentHashMap是Java中的一个线程安全的哈希表实现，它可以在多线程环境下并发地进行读写操作，而不需要像传统的HashTable那样在读写时加锁。

ConcurrentHashMap的实现原理主要基于分段锁和CAS操作。它将整个哈希表分成了多个Segment（段），每个Segment都类似于一个小的HashMap，它拥有自己的数组和一个独立的锁。在ConcurrentHashMap中，读操作不需要锁，可以直接对Segment进行读取，而写操作则只需要锁定对应的Segment，而不是整个哈希表，这样可以大大提高并发性能。

在ConcurrentHashMap的实现中，写操作需要进行CAS操作，以保证线程安全。在写操作时，如果当前Segment已经被其他线程锁定了，那么当前线程就会进行自旋等待，直到其他线程释该段锁。

ConcurrentHashMap的另一个特点是它的**迭代器是弱一致**的，也就是说，在遍历时可以看到不一致的数据。这是因为在多线程环境下，ConcurrentHashMap中的数据是动态的，而迭代器的创建时间和遍历时间可能存在一定的时间差，这样就可能会出现一些数据的更新在迭代器创建后进行，但在遍历时还没有反映出来的情况。虽然这样会影响到数据的可见性，但是可以提高并发性能，因为不需要在迭代器上加锁。

需要注意的是，虽然ConcurrentHashMap是线程安全的，但是它并不能保证数据的实时一致性。在多线程环境下，由于线程的执行顺序和时间差异，可能会出现数据的不一致性。因此，**ConcurrentHashMap适用于读多写少、对数据一致性要求不高的场景**，对于对数据实时一致性要求较高的场景，可能需要采用其他的数据结构或者加锁来保证数据的一致性。

### 阻塞队列

BlockingQueue是Java中一个线程安全的队列接口，它扩展了Queue接口，提供了阻塞操作，可以在队列为空或者队列已满时自动阻塞等待。BlockingQueue在多线程编程中广泛应用，特别是在生产者消费者模式中。

BlockingQueue提供了一系列阻塞方法，包括：

- put(E e): 向队列尾部添加元素，如果队列已满，会一直阻塞直到有空闲的空间。
- offer(E e, long timeout, TimeUnit unit): 向队列尾部添加元素，在指定的时间内等待空闲的空间，如果队列还是满的，返回false。
- take(): 获取并删除队列头部的元素，如果队列为空，会一直阻塞直到有元素可用。
- poll(long timeout, TimeUnit unit): 获取并删除队列头部的元素，在指定的时间内等待元素，如果还是空的，返回null。

BlockingQueue实现了多线程间的同步，它可以保证线程之间的有序性，而不需要显式地加锁。BlockingQueue还提供了可扩展和不可扩展的两种实现方式。在可扩展的实现中，如果队列满了，可以自动扩展队列大小；在不可扩展的实现中，队列的大小是固定的，如果队列已满，新的元素无法添加。

Java中提供了多种BlockingQueue的实现类，如ArrayBlockingQueue、LinkedBlockingQueue、PriorityBlockingQueue等。其中，ArrayBlockingQueue和LinkedBlockingQueue是最常用的实现类。它们的实现方式不同，ArrayBlockingQueue是基于数组实现的，而LinkedBlockingQueue是基于链表实现的，因此它们在性能和可扩展性方面有所差异，应该根据实际需求来选择。

### ThreadLocal

Thread类有两个变量：threadLocals和inheritableThreadLocals

这两个变量默认为null，只有当该线程调用了ThreadLocal类的get/set方法时才会创建他们，而调用ThreadLocal的get/set实际上是调用ThreadLocalMap的get/set

ThreadLocalMap可理解成给ThreadLocal定制化的HashMap

最终的变量放在了线程的ThreadLocalMap中，而不是ThreadLocal中，ThreadLocal只是对其进行封装，向其传递变量值

**用一个场景分析ThreadLocal的get/set流程：**

首先在所有线程外部创建一个共享的ThreadLocal对象，记为TL1。在一个线程中调用TL1.get()时，首先获取到当前线程对象，记为t，然后判断t.threadLocals是否为null，如果为null，就在t中创建一个新的ThreadLocalMap对象赋值给t.threadLocals，并将<TL1, null>插入其中，最后get方法返回null；如果不为null，则尝试获取threadLocals中TL1所在的键值对，如果该键值对为null，则向threadLocals中通过set方法插入<TL1, null>，最后返回null，如果键值对不为null，则返回键值对中的值。

调用set方法时，流程和get基本一致，只是从读变成了写

这样就可以实现不同线程访问同一个ThreadLocal能拿到各自向其中存放的值

![image-20210904182447978](https://i.loli.net/2021/09/12/MNrw3TFOAv9hGXP.png)

**ThreadLocal内存泄漏问题：**

ThreadLocalMap中的key为ThreadLocal的弱引用，value为强引用

如果ThreadLocal没有被外部强引用，垃圾回收时key会被清理掉，但value不会

这时key=null，而value不为null，如果不做处理，value将永远不会被GC掉，就有可能发生内存泄漏

ThreadLocalMap的实现中考虑了这个问题，在调用get/set/remove时会清理掉key为null的entry

在编程时如果意识到当前编写的run方法里不再会使用ThreadLocal了，最好手动调用remove

**为什么用ThreadLocal不用线程成员变量**

如果用成员变量，那么成员变量必须在Thread里，不能在Runnable里，因为一个Runnable对象可以被多个Thread执行。而如果在Thread中添加成员变量，就要加强Thread和Runnable的耦合，将Thread作为Runnable的成员变量，并在Runnable中调用具体的Thread变量，如果执行Runnable的Thread可能有很多子类，不同子类有不同的成员变量，则要在run方法中进行复杂处理，扩展性较低，不利于维护。而ThreadLocal就是将成员变量统一为一个Map放到线程里。

### 线程池

> [!NOTE]
>
> #### **线程池的好处**
>
> 降低**资源消耗**。通过**复用**已经创建的线程减少线程创建和销毁的开销
>
> 提高**响应速度**。任务到达时**不需要等线程创建**就能直接处理
>
> 提高线程**可管理**型。数量得到控制、线程池能对其中的线程统一分配、监控和调优
>
> #### 参数
>
> **核心线程数 (`corePoolSize`)**：
>
> - 线程池会始终保留的线程数量。
> - 即使线程处于空闲状态，也**不会销毁**。
>
> **最大线程数 (`maximumPoolSize`)**：
>
> - 当任务队列已满时，线程池会创建额外的线程，直到达到最大线程数。
>
> **存活时间 (`keepAliveTime`)**：
>
> - **超过核心线程数的线程（非核心线程）在空闲时间达到** `keepAliveTime` 后会被销毁。
>
> **任务队列 (`BlockingQueue`)**：
>
> - 用于保存等待执行的任务。
>
> **拒绝策略**
>
> - 线程池炸了的拒绝策略
>
> #### 执行过程：
>
> 提交任务到线程池（通过 `execute` 或 `submit` 方法）。
>
> 线程池检查当前运行线程数：
>
> - 如果运行线程数小于核心线程数，则创建一个新线程执行任务。
> - 如果运行线程数达到核心线程数，则将任务加入任务队列。
>
> 如果任务队列已满且线程数小于最大线程数，则创建新线程执行任务（也可以服用非核心线程）。
>
> 如果任务队列已满且线程数达到最大线程数，则执行拒绝策略。
>
> > 两个size？？
> >
> > - 核心线程：低负载情况下快速响应
> > - 最大线程：高负载突发场景。高负载过去后，这些线程可以被销毁

> [!NOTE]
>
> 使用`execute()`时，未捕获异常导致线程终止，线程池创建新线程替代（用户自己处理yi'chang）；使用`submit()`时，异常被封装在`Future`中，线程继续复用。





**线程池系统模型和任务执行流程**

![image-20210905104343506](https://i.loli.net/2021/09/12/kDbxTKnwZFPXVY6.png)

ThreadPoolExecutor.execute()执行流程：

这个流程设计的原则是尽可能避免全局锁mainLock的获取，进而优化其性能

流程中只有1和3这两个步骤中创建新线程时需要获取全局锁，而在线程池实际工作中，当ThreadPoolExecutor预热好以后（预热指逐步加满corePool的过程），基本上大多数任务都是在步骤2中进行处理，不需要获取全局锁，因此性能得到了提升。

通过上面的流程，我们可以发现线程执行任务的方式：

1. 当execute()过程中创建了新的线程时，该线程直接执行当前任务
2. 该新线程执行完第一个任务后，就反复从workQueue中获取任务执行

**线程池的创建方式**

**方式一：通过Executors，按照其设计好的三个线程池模板进行创建**

1. FixedThreadPool：返回一个固定线程数量的线程池，其中线程数量从创建后就不再改变，执行任务时如果有空闲线程就立即执行，如果没有就将任务加到队列中，有线程空闲后从队列头获取任务执行
2. SingleThreadExecutor：只有一个线程的FixedThreadPool
3. CachedThreadPool：没有队列，动态调整线程数量，有空闲线程就复用，没有就创建，新创建的执行完第一个任务后留在线程池中等待被复用

《阿里巴巴Java开发手册》中不允许使用Executors创建线程池，这是因为两点，一是Executors提供的模板不够灵活，多数情况需要程序员自定义线程池，二是模板容易造成内存溢出（1和2队列过载、3线程池过载）。

**方式二：通过ThreadPoolExecutor构造方法创建**

ThreadPoolExecutor一共有4个构造器，其中最核心的就是下面给出的，另外3个只是在其基础上为一些参数设定了默认值

```java
    /**
     * 用给定的初始参数创建一个新的ThreadPoolExecutor。
     */
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```

| 参数            | 类型                     | 意义                                                         |
| --------------- | ------------------------ | ------------------------------------------------------------ |
| corePoolSize    | int                      | 核心线程池容量                                               |
| maximumPoolSize | int                      | 线程池总容量                                                 |
| keepAliveTime   | long                     | 核心外线程最大空闲时间，空闲超过这个时间就会被销毁           |
| unit            | TimeUnit                 | keepAliveTime的时间单位                                      |
| workQueue       | BlockingQueue/<Runnable/>  | 任务队列                                                     |
| threadFactory   | ThreadFactory            | 产生线程的工厂                                               |
| handler         | RejectedExecutionHandler | 拒绝执行处理器，当线程池饱和时，按照该参数对应的饱和策略对新任务进行拒绝处理 |

JUC下提供的RejectedExecutionHandler的实现类即饱和策略有4种：

1. AbortPolicy：抛出 RejectedExecutionException 来拒绝新任务的处理。
2. CallerRunsPolicy：让调用execute()提交任务的线程执行任务。这种方式的弊端是该线程执行任务时，可能线程池中已经有很多任务已经执行完了，这就阻挡了后续任务的提交，降低了性能。
3. DiscardPolicy：啥也不做，直接过
4. DiscardOldPolicy：丢弃workQueue队列头的任务（即最早未处理的任务），然后重新调用execute()提交当前任务

不指定的话默认是AbortPolicy



> [!NOTE]
>
> #### volatile关键字
>
> 每次使用它都到主存中进行读取，禁用CPU缓存。只保证可见性。多线程下+1操作可能交替执行。
>
> synchronized能保证可见性与原子性
>
> #### 可见性与原子性
>
> - 可见性：当一个线程修改了共享变量的值，其他线程能够立即看到这个修改的结果
> - 原子性：一组操作是不可中断的。要么执行成功、要么不执行。

