# 多线程并发

- synchronized 修饰 static 方法、普通方法、类、方法块区别
- synchronized 底层实现原理
- volatile 的作用和原理
- 说两个线程同步的集合类
- 构造一个出现死锁的情况
- 线程安全的集合和类型
- ThreadLocal

Java 内存模型规定了所有的变量都存储在主内存中，每条线程有自己的工作内存。

volatitle保证可见性，禁止指令重排序，只有一个线程执行写操作，其它线程都是读操作的情况。

**volatitle原理**

采用了`内存屏障`

1. 它确保指令重排序时不会把其后面的指令排到内存屏障之前的位置，也不会把前面的指令排到内存屏障的后面；即在执行到内存屏障这句指令时，在它前面的操作已经全部完成；

2. 它会强制将缓存的修改操作立即写到主内存

3. 写操作会导致其它CPU中的缓存行失效，写之后，其它线程的读操作会从主内存读。

**Synchronized**

使用Synchronized进行同步，其关键就是必须要对对象的监视器monitor进行获取，当线程获取monitor后才能继续往下执行，否则就进入同步队列，线程状态变成BLOCK，同一时刻只有一个线程能够获取到monitor，当监听到monitorexit被调用，队列里就有一个线程出队，获取monitor。

`ReentranLock` 可以通过代码释放锁，可以设置锁超时。

#### **原理**

基于 `Lock` 接口的可重入锁，提供更灵活的同步控制

**可重入锁（Reentrant Lock）** 是一种允许同一线程多次获取同一把锁的同步机制。其核心设计是为了避免线程因重复获取自己已持有的锁而导致死锁，

**偏向锁的实现**

只有一个线程执行同步代码块时能够提高性能

Synchronized并不是一开始就是重量级锁，在并发不严重的时候，比如只有一个线程访问的时候，是偏向锁；当多个线程访问，但不是同时访问，这时候锁升级为轻量级锁；当多个线程同时访问，这时候升级为重量级锁

**LinkBlockingQueue和ConcurrentHashMap**

ConcurrentHashMap 采用分段锁，内部默认有16个桶，get和put操作，首先将key计算hashcode，然后跟16取余，落到16个桶中的一个，然后每个桶中都加了锁（ReentrantLock），桶中是HashMap结构（数组加链表，链表过长转红黑树）。

JAva8抛弃了原有的 Segment 分段锁，而采用了 CAS + synchronized 来保证并发安全性。结构上和 Java8 的 HashMap（数组+链表+红黑树） 基本上一样，不过它要保证线程安全性，所以在源码上确实要复杂一些。1.8 在 1.7 的数据结构上做了大的改动，采用红黑树之后可以保证查询效率（O(logn)），甚至取消了 ReentrantLock 改为了 synchronized，这样可以看出在新版的 JDK 中对 synchronized 优化是很到位的。

CAS是无锁并发控制

**那么ThreadLocal是如何确保只有当前线程可以访问呢**

在给ThreadLocal去set值或者get值的时候都会先获取当前线程，然后基于线程去调用getMap(thread),getMap返回的就是线程thread的成员变量threadLocals

以ThreadLocal对象的**弱引用**作为key，ThreadLocal里“存放”的数据作为value，放在该Map中。

每个线程内部有一个名为threadLocals的成员变量，该变量的类型为ThreadLocal.ThreadLocalMap类型（类似于一个HashMap），其中的key为当前定义的ThreadLocal变量的this引用，value为我们使用set方法设置的值。每个线程的本地变量存放在自己的本地内存变量threadLocals中，如果当前线程一直不消亡，那么这些本地变量就会一直存在（所以可能会导致内存溢出），因此使用完毕需要将其remove掉。

ThreadLocalMap内部实际上是一个Entry数组，THreadLocalMap中的Entry的key使用的是ThreadLocal对象的弱引用，在没有其他地方对ThreadLoca依赖，ThreadLocalMap中的ThreadLocal对象就会被回收掉，但是对应的不会被回收，这个时候Map中就可能存在key为null但是value不为null的项，这需要实际的时候使用完毕及时调用remove方法避免内存泄漏。

每个线程都有自己的 `ThreadLocalMap`，并且这些 `ThreadLocalMap` 仅能由所属线程访问，因此不同线程之间的变量副本是完全隔离的

**乐观锁**

CAS 其实是一种乐观锁，一般有三个值，分别为：赋值对象，原值，新值，在执行的时候，会先判断内存中的值是否和原值相等，相等的话把新值赋值给对象，否则赋值失败，整个过程都是原子性操作，没有线程安全问题。

AtomicInteger内部采用CAS原理

CAS的问题：ABA问题，循环时间长开销大和只能保证一个共享变量的原子操作

**阻塞队列**

最常使用的例子就是**生产者消费者模式**

lock锁+多个条件（condition）的阻塞控制

**线程安全的集合和类型**

CopyOnWriteArrayList

- **读操作无锁**：所有读操作（如 `get`、`iterator`）直接访问底层数组，无需加锁。
- **写操作加锁复制**：所有写操作（如 `add`、`set`、`remove`）会先复制底层数组，在副本上修改数据，再将新数组替换旧数组。这一过程使用 **`ReentrantLock`** 保证线程安全。
- **数据可见性**：底层数组引用通过 `volatile` 修饰，确保修改后的新数组对其他线程立即可见。

ConcurrentHashMap 

LinkBlockingQueue

**ThreadLocal**

创建线程局部变量的类



[面试官：说说多线程并发问题](https://juejin.cn/post/6844903941830869006)

[面试必备：Kotlin 线程同步的 N 种方法](https://juejin.cn/post/6981952428786597902)

[不可不说的Java“锁”事](https://tech.meituan.com/2018/11/15/java-lock.html)

[[Java中的ThreadLocal详解](https://www.cnblogs.com/fsmly/p/11020641.html)

# Jvm管理

- 对象被设为null会被回收吗
- 垃圾回收机制

一个对象的引用执行null，并不会被立即回收的，还需要执行`finalize()`方法

1. 堆(heap) : 他是最大的一块区域，用于存放对象实例和数组，是全局共享的.（也称为逻辑堆，主要用来存放对象实例与数组，对于所有的线程来说他是共享的，对于Heap堆区是动态分配内存的，所以空间大小和生命周期都不是明确的，而GC的主要作用就是自动释放逻辑堆里实例对象所占的内存，而在逻辑堆中还分为新生代与老年代，用来区分对象的存活时间，在新生代中还被细致的分为 Eden SurvivorFrom以及SurvivorTo这三部分.）
2. 栈(stack) : 全称为虚拟机栈，主要存储基本数据类型，以及对象的引用，私有线程（在每一个对象被创建的时候，在堆栈区都有一个对他的引用，在这里我们可以这样理解。）每个栈帧存储了方法的局部变量表、操作数栈、动态连接和方法返回地址等信息
3. 方法区(Method Area) : 在class被加载后的一些信息 如常量，静态常量这些被放在这里，在Hotspot里面我们将它称之为永生代（方法区主要存储（类加载器）ClassLoader加载的类信息，在这里我们可以理解为已经编译好的代码储存区，所以存储包括类的元数据，常量池，字段，静态变量与方法内的局部变量以及编译好的字节码，等等）它用于存储已被虚拟机加载的类信息、常量、静态变量、及时编译器编译后的代码等数据

在《Java虚拟机规范中》，对这个区域规定了两种异常状况：

1. 如果线程请求的栈深度大于虚拟机所允许的深度，将抛出 StackOverflowError
2.  如果Java虚拟机栈可以动态扩展，当扩展时无法申请到足够的内存，就会抛出OutOfMemoryError异常

 执行 a = a++ 操作，原先已经执行了 a++ 操作，这个时候将 a++ 中 a 赋值给 int a ，所以会将栈中的数据赋值到 局部变量表中，所以这个时候局部变量表中的数据就是88了

JVMTI OOM监控

直接内存：它避免了Java堆和Native堆来回交换数据的时

GCRoot：

**1、虚拟机栈中引用的对象**
比如：各个线程被调用的方法中使用到的参数、局部变量等。

**2、本地方法栈内JNI（通常说的本地方法）引用的对象**

**3、方法区中类静态属性引用的对象**
比如：Java类的引用类型静态变量

**4、方法区中常量引用的对象**
比如：字符串常量池（string Table） 里的引用

**5、所有被同步锁synchronized持有的对象**

**6、Java虚拟机内部的引用。**
基本数据类型对应的Class对象，一些常驻的异常对象（如：
NullPointerException、OutOfMemoryError） ，系统类加载器。

**7、反映java虚拟机内部情况的JMXBean、JVMTI中注册的回调、本地代码缓存等**

## 垃圾回收机制



### Minor GC、Major GC、Full GC是什么

1. Minor GC是新生代GC，指的是发生在新生代的垃圾收集动作。由于java对象大都是朝生夕死的，所以Minor GC非常频繁，一般回收速度也比较快。（一般采用复制算法回收垃圾）
2. Major GC是老年代GC，指的是发生在老年代的GC，通常执行Major GC会连着Minor GC一起执行。Major GC的速度要比Minor GC慢的多。（可采用标记清楚法和标记整理法）
3. Full GC是清理整个堆空间，包括年轻代和老年代

直接内存是基于物理内存和Java虚拟机内存的中间内存

[JVM那点事-对象的自救计划（对象被设为null会被回收吗？）](https://www.jianshu.com/p/0618241f9f44)

[JVM内存结构——运行时数据区](https://www.cnblogs.com/zhengbin/p/5617023.html)

[【死磕JVM】一道面试题引发的“栈帧”！！！](https://www.cnblogs.com/mingyueyy/p/14538754.html)

[Java虚拟机（JVM）面试题](https://juejin.cn/post/6844904125696573448?searchId=2023081311435315EC3968E2B93A344149)

![image-20220423111925307](../assets/Java知识点汇总/image-20220423111925307.png)

# 集合

- Java 集合，介绍下ArrayList 和 HashMap 的使用场景，底层实现原理
- ArrayList 与 LinkedList 的区别
- ArrayList、HashMa和LinkedList的默认空间是多少？扩容机制是什么
- Hashmap、hashtable/TreeMap原理

**ArrayList、HashMa和LinkedList的默认空间是多少？扩容机制是什么**

ArrayList 的默认大小是 10 个元素。扩容点规则是，新增的时候发现容量不够用了，就去扩容；扩容大小规则是：扩容后的大小= 原始大小+原始大小/2 + 1。HashMap 的默认大小是16个元素(必须是2的幂)。扩容因子默认0.75，扩容机制.(当前大小 和 当前容量 的比例超过了 扩容因子，就会扩容，扩容后大小为 一倍。例如：初始大小为 16 ，扩容因子 0.75 ，当容量为12的时候，比例已经是0.75 。触发扩容，扩容后的大小为 32.)LinkedList 是一个双向链表，没有初始化大小，也没有扩容的机制，就是一直在前面或者后面新增就好。

## HashMap

- HashMap的底层数据结构？
- HashMap的存取原理？
- Java7和Java8的区别？
- 为啥会线程不安全？
- 有什么线程安全的类代替么?
- 默认初始化大小是多少？为啥是这么多？为啥大小都是2的幂？
- HashMap的扩容方式？负载因子是多少？为什是这么多？
- HashMap的主要参数都有哪些？
- HashMap是怎么处理hash碰撞的？
- hash的计算规则
- HashMap容量为什么是2的n次方？如何做到的
- 头插法 尾插法

HashMap基于hashing原理，我们通过put()和get()方法储存和获取对象。当我们将键值对传递给put()方法时，它调用键对象的hashCode()方法来计算hashcode，让后找到bucket位置来储存值对象。当获取对象时，通过键对象的equals()方法找到正确的键值对，然后返回值对象。HashMap使用链表来解决碰撞问题，当发生碰撞了，对象将会储存在链表的下一个节点中。 HashMap在每个链表节点中储存键值对对象。

**HashMap在put数据时是如何找到要存放的位置的**

HashMap容量取2的n次方，主要与hash寻址有关。在put(key,value)时，putVal()方法中通过i = (n - 1) & hash来计算key的散列地址。其实，i = (n - 1) & hash是一个%操作。也就是说，HashMap是通过%运算来获得key的散列地址的。但是，%运算的速度并没有&的操作速度快。而&操作能代替%运算，必须满足一定的条件，也就是a%b=a&(b-1)仅当b是2的n次方的时候方能成立。这也就是为什么HashMap的容量需要保持在2的n次方了。


hashcode计算时，会做右移16位的操作，根据数组长度和新哈希码值计算数组下标的地方

为什么是n-1，n代表数组的长度，为2的x次幂的数，故（n-1）代表的二进制全为1

p = tab[i = (n - 1) & hash]

头插法  尾插法

**主要是为了安全,防止环化**
因为resize的赋值方式，也就是使用了**单链表的头插入方式，同一位置上新元素总会被放在链表的头部位置**，在旧数组中同一条Entry链上的元素，通过重新计算索引位置后，有可能被放到了新数组的不同位置上。

使用头插会改变链表的上的顺序，但是如果使用尾插，在扩容时会保持链表元素原本的顺序，就不会出现链表成环的问题了

[列举2个线程安全的集合类_Java面试题：Java中的集合及其继承关系](https://blog.csdn.net/weixin_28878185/article/details/113534446)

[《我们一起进大厂》系列-HashMap](https://juejin.cn/post/6844904017269637128)

[吊打面试官》系列-ConcurrentHashMap & Hashtable](https://juejin.cn/post/6844904023003250701)

[HashMap底层实现原理/HashMap与HashTable区别/HashMap与HashSet区别](https://www.jianshu.com/p/45fa4e80b631)

TreeMap的实现是红黑树算法的实现

1. TreeMap中的元素，key是升序的唯一，value是无序，不唯一

put(key,value)：当我们调用该方法时，首先会以根节点为当前节点，通过比较添加的节点与根节点的大小，当该节点大于根节点且该根节点有右子节点，将会将该右字节点继续比较；当该节点小于根节点且该节点有左子节点时，将会将该左字节点继续比较；当该节点和根节点相等时，此时将会覆盖此处的值；依次重复比较，直到找到该位置，即当小于根节点就将该节点挂在该根节点的左节点上，当大于根节点就将该节点挂在该根节点的右子树上
get(key)：通过key值找value值，首先从根节点作为起始位置，和根节点的key值进行比较，如果大于该根节点，说明该key位置在根节点的右侧，将根节点的右节点作为新的根节点，继续比较（这其中就排除了另根的左子树上所有的集合，类似折半查找的思想，提高了查找的效率）；同理，如果小于根节点的key值，说明在该key在根节点的左侧，将根节点的左节点作为新的根节点，继续比较；如果根节点的key和我们的key值相等，则说明此时找到到了该值，将该节点的value值进行返回即可；不断地重复上述（类似递归的思想），直到找到对应key值

ArrayMap

它内部使用两个数组进行工作，其中一个数组记录key hash过后的顺序列表，另外一个数组按key的顺序记录Key-Value值

SparseArray

内部也是采用数组实现，分别将 key 和 value 保存在两个数组中，通过二分查找来进行增删改查操作



[ArrayMap原理解析](https://blog.csdn.net/qq_31481093/article/details/116110304)

# 序列化

- 序列化Serializable和Parcelable的理解和区别
- 如果持久化需要用哪一个

Parcelable的性能比Serializable好，在内存开销方面较小，所以在内存间数据传输时推荐使用Parcelable，如activity间传输数据，而Serializable可将数据持久化方便保存，所以在需要保存或网络传输数据时选择Serializable，因为android不同版本Parcelable可能不同，所以不推荐使用Parcelable进行数据持久化

**选择序列化方法的原则**

在使用内存的时候，Parcelable比Serializable性能高，所以推荐使用Parcelable。
Serializable在序列化的时候会产生大量的临时变量，从而引起频繁的GC。
Parcelable不能使用在要将数据存储在磁盘上的情况，因为Parcelable不能很好的保证数据的持续性在外界有变化的情况下。尽管Serializable效率低点，但此时还是建议使用Serializable 。

[android 序列化Serializable和Parcelable的理解和区别](https://blog.csdn.net/fendouwangzi/article/details/86240269)

# 线程池

- 线程池都什么时候用，怎么创建，构造函数中的参数分别代表什么意思？
- 说下对线程池的理解，以及创建线程池的几个关键参数
- 线程池中核心线程和非核心线程的区别
- 核心线程在什么情况下会被销毁
- 线程池的数据结构是什么

我如果需要同时做很多事情，是不是给每一个事件都开启一个线程呢？那如果我的事件无限多呢？频繁地创建/销毁线程，CPU该吃不消了吧。所以，这时候线程池的概念就来了。

```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue) {
    this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
         Executors.defaultThreadFactory(), defaultHandler);
}
```

1. corePoolSize 线程池核心线程大小

   线程池中会维护一个最小的线程数量，即使这些线程处理空闲状态，他们也不会被销毁，除非设置了allowCoreThreadTimeOut。这里的最小线程数量即是corePoolSize。

2. maximumPoolSize 线程池最大线程数量

   当任务数量超过最大线程数时其它任务可能就会被阻塞。最大线程数=核心线程+非核心线程。非核心线程只有当核心线程不够用且线程池有空余时才会被创建，执行完任务后非核心线程会被销毁。

3. keepAliveTime 空闲线程存活时间

   一个线程如果处于空闲状态，并且当前的线程数量大于corePoolSize，那么在指定时间后，这个空闲线程会被销毁，这里的指定时间由keepAliveTime来设定

4. unit 空闲线程存活时间单位

5. workQueue 工作队列

   线程池中的任务队列，我们提交给线程池的runnable会被存储在这个对象上。

6. threadFactory 线程工厂

   创建一个新线程时使用的工厂，可以用来设定线程名、是否为daemon线程等等

7. handler 拒绝策略

   当工作队列中的任务已到达最大限制，并且线程池中的线程数量也达到最大限制，这时如果有新任务提交进来，该如何处理呢。这里的拒绝策略，就是解决这个问题的

**线程池的分配遵循这样的规则：**

当线程池中的核心线程数量未达到最大线程数时，启动一个核心线程去执行任务；

如果线程池中的核心线程数量达到最大线程数时，那么任务会被插入到任务队列中排队等待执行；

如果在上一步骤中任务队列已满但是线程池中线程数量未达到限定线程总数，那么启动一个非核心线程来处理任务；

如果上一步骤中线程数量达到了限定线程总量，那么线程池则拒绝执行该任务，且ThreadPoolExecutor会调用RejectedtionHandler的rejectedExecution方法来通知调用者。

**线程池的分类**

`CachedThreadPool`: 可缓存的线程池，该线程池中没有核心线程，非核心线程的数量为Integer.max_value，就是无限大，当有需要时创建线程来执行任务，没有需要时回收线程，适用于耗时少，任务量大的情况。

`SecudleThreadPool:`周期性执行任务的线程池，按照某种特定的计划执行线程中的任务，有核心线程，但也有非核心线程，非核心线程的大小也为无限大。适用于执行周期性的任务。

`SingleThreadPool` :只有一条线程来执行任务，适用于有顺序的任务的应用场景。

`FixedThreadPool`:定长的线程池，有核心线程，核心线程的即为最大的线程数量，没有非核心线程

**线程池一般用法**

- shutDown()，关闭线程池，需要执行完已提交的任务；
- shutDownNow()，关闭线程池，并尝试结束已提交的任务；
- allowCoreThreadTimeOut(boolen)，允许核心线程闲置超时回收；
- execute()，提交任务无返回值；
- submit()，提交任务有返回值；

**拒绝策略有几种**

ThreadPoolExecutor.AbortPolicy:丢弃任务并抛出RejectedExecutionException异常。 
ThreadPoolExecutor.DiscardPolicy：也是丢弃任务，但是不抛出异常。 
ThreadPoolExecutor.DiscardOldestPolicy：丢弃队列最前面的任务，然后重新尝试执行任务（重复此过程）
ThreadPoolExecutor.CallerRunsPolicy：由调用线程处理该任务 



1）当提交一个新任务到线程池时，线程池判断corePoolSize线程池是否都在执行任务，如果有空闲线程，则从核心线程池中取一个线程来执行任务，直到当前线程数等于corePoolSize；

2）如果当前线程数为corePoolSize，继续提交的任务被保存到阻塞队列中，等待被执行；

3）如果阻塞队列满了，那就创建新的线程执行当前任务，直到线程池中的线程数达到maxPoolSize，这时再有任务来，由饱和策略来处理提交的任务

无任务执行时，线程池其实是利用阻塞队列的take方法挂起，从而维持核心线程的存活

核心线程通常情况下不会被销毁，只有在以下几种情况下会被销毁： 1. 当线程池调用shutdown()或shutdownNow()方法时，核心线程会被销毁。 2. 当线程池设置allowCoreThreadTimeOut(boolean value)方法的参数为true，并且设置了线程空闲时间，那么空闲时间超过设置的时间，核心线程也会被销毁。 3. 当线程所在的进程结束时，核心线程也会被销毁。 4. 手动调用线程的interrupt方法，也可以销毁线程。但这种方式并不推荐，因为可能会导致线程不安全问题。 5. 核心线程在执行任务时出现未捕获的异常，也可能导致线程被销毁。

CPU密集型：核心线程数 = CPU核数 + 1
IO密集型：核心线程数 = CPU核数 * 2

注：IO密集型（某大厂实践经验）
    核心线程数 = CPU核数 / （1-阻塞系数）   例如阻塞系数 0.8，CPU核数为4
    则核心线程数为20

- 线程池初始化时，**核心线程不会立即创建**（按需创建，直到达到 `corePoolSize`）。
- **空闲状态**：即使核心线程处于空闲状态（无任务执行），默认情况下也不会被销毁，会永久阻塞等待新任务。
- **典型场景**：
  假设线程池参数为 `corePoolSize=5`，所有核心线程创建后长期存活，即使线程池中没有任务。
- 通过 `allowCoreThreadTimeOut(true)`，核心线程空闲超时后销毁，最终线程池可能完全无线程。



[Android 线程和线程池一篇就够了](https://juejin.cn/post/6844903480193187854)

[java线程池如何配置核心线程数](https://www.cnblogs.com/xy-ouyang/p/14718195.html)

[基础篇：高并发一瞥，线程和线程池的总结](https://juejin.cn/post/6854573219341402119#heading-2)

[java线程池](https://blog.csdn.net/sjzwangxufeng/article/details/129415572)

# 类加载

其中系统类加载器包括3种，分别是

Bootstrap ClassLoader(启动加载器)、 

Extensions ClassLoade(扩展加载器)r和 

App ClassLoader(应用加载器)。

类加载机制，加载过程 ？

加载，验证，准备，解析，初始化，使用和卸载。其中验证，准备，解析3个部分统称为连接。

## 双亲委托模式

所谓双亲委托模式就是首先判断该Class是否已经加载，如果没有则不是自身去查找而是委托给父加载器进行查找，这样依次的进行递归，直到委托到最顶层的Bootstrap ClassLoader，如果Bootstrap ClassLoader找到了该Class，就会直接返回，如果没找到，则继续依次向下查找，如果还没找到则最后会交由自身去查找。

采取双亲委托模式主要有两点好处：

1. 避免重复加载，如果已经加载过一次Class，就不需要再次加载，而是先从缓存中直接读取。
2. 更加安全，如果不使用双亲委托模式，就可以自定义一个String类来替代系统的String类，这显然会造成安全隐患，采用双亲委托模式会使得系统的String类在Java虚拟机启动时就被加载，也就无法自定义String类来替代系统的String类，除非我们修改
   类加载器搜索类的默认算法。还有一点，只有两个类名一致并且被同一个类加载器加载的类，Java虚拟机才会认为它们是同一个类，想要骗过Java虚拟机显然不会那么容易

[Android解析ClassLoader（一）Java中的ClassLoader](http://liuwangshu.cn/application/classloader/1-java-classloader-.html)

[深入探讨 Java 类加载器](https://blog.csdn.net/gongxiao1993/article/details/81351988)

[类加载机制](https://lrh1993.gitbooks.io/android_interview_guide/content/java/virtual-machine/classloader.html)

# 基础知识：

## 注解

- Source和Class、Runtime注解的区别

这个表示注解的保留方式，具体有一下三种类型:

- RetentionPolicy.SOURCE(源码级注解): 只保留在源码中，不保留在class中，同时也不加载到虚拟机中。在编译阶段丢弃。这些注解在编译结束之后就不再有任何意义，所以它们不会写入字节码。
  源码级注解主要有两个作用:
  (1).作为源代码的补充，给开发者看的，比如@Override注解
  (2).给注解处理器(APT)用的，用于生成源代码。

- RetentionPolicy.CLASS(字节码注解): 保留在源码中，同时也保留到class中，但是不加载到虚拟机中。在类加载的时候丢弃。在字节码文件的处理中有用，比如字节码插桩。注解，默认使用这种方式。
  字节码级注解主要的作用:
  用于字节码修改，字节码插桩。比如，使用aspectj、ASM等工具进行字节码修改。

- RetentionPolicy.RUNTIME(运行时注解): 保留到源码中，同时也保留到class中，最后加载到虚拟机中。始终不会丢弃，运行期也保留该注解，因此可以使用反射机制读取该注解的信息。我们自定义的注解通常使用这种方式。
  运行时注解主要的作用:
  在运行时使用反射进行使用，参与某些业务逻辑。
  

## 反射

## 泛型

- 泛型擦除，为何会有擦除？擦除的时机。通配符。

兼容Java老版本，泛型概念是Java5才出来的。使用类型擦除不再为参数化类型创造新类了，同时在编译期间将`泛型类型`中的`类型参数`全部替换 `Object`。

- 通过擦除泛型类型信息，编译后的字节码与旧版本 JVM 兼容
- **不修改字节码结构**：泛型擦除后，JVM 无需为泛型引入新的字节码指令或类型系统，保持运行时结构简单。
- **降低复杂度**：避免为每个泛型类型生成独立的类

在 Java 的泛型实际实现中，会根据泛型类型中的类型参数有无边界，来选择是否替换为边界或 Object。

```java
public class Test {
    static class Food {}
    static class Fruit extends Food {}
    static class Apple extends Fruit {}

    public static void main(String[] args) throws IOException {
        List<? extends Fruit> fruits = new ArrayList<>();
        fruits.add(new Food());     // compile error
        fruits.add(new Fruit());    // compile error
        fruits.add(new Apple());    // compile error

        fruits = new ArrayList<Fruit>(); // compile success
        fruits = new ArrayList<Apple>(); // compile success
        fruits = new ArrayList<Food>(); // compile error
        fruits = new ArrayList<? extends Fruit>(); // compile error: 通配符类型无法实例化  

        Fruit object = fruits.get(0);    // compile success
    }
}


public class Test {
    static class Food {}
    static class Fruit extends Food {}
    static class Apple extends Fruit {}

    public static void main(String[] args) throws IOException {
        List<? super Fruit> fruits = new ArrayList<>();
        fruits.add(new Food());     // compile error
        fruits.add(new Fruit());    // compile success
        fruits.add(new Apple());    // compile success

        fruits = new ArrayList<Fruit>(); // compile success
        fruits = new ArrayList<Apple>(); // compile error
        fruits = new ArrayList<Food>(); // compile success
        fruits = new ArrayList<? super Fruit>(); // compile error: 通配符类型无法实例化      

        Fruit object = fruits.get(0); // compile error
    }
}
```

1. extends 可用于的返回类型限定，不能用于参数类型限定。
2. super 可用于参数类型限定，不能用于返回类型限定。
3. 带有 super 超类型限定的通配符可以向泛型对易用写入，带有 extends 子类型限定的通配符可以向泛型对象读取。

在java源文件中的泛型在编译后的.class文件中都将被擦除，类加载的时候擦除

- **类型擦除的本质**：在编译期检查类型安全，运行期丢弃泛型类型信息。
- **优点**：兼容旧版本 JVM，减少运行时开销。

① **协变**：**父子关系一致** → **子类也可以作为参数传进来** → **<? extends Entity>** → **上界通配符**

② **逆变**：**父子关系颠倒** → **父类也可以作为参数传进来** → **<? super Article>** → **下界通配符**

绕开类型擦除的方法通过 `Class` 对象、反射或 Kotlin 的 `reified` 关键字绕过限制。

先是 **协变** → **能读不能写** (**能用父类型去获取数据**，不确定具体类型，不能传)

接着是 **逆变** → **能写不能读** (**能传入子类型**，不确定具体类型，不能读，但可以用Object读)

[深入理解 Java 泛型：类型擦除、通配符、运行时参数类型获取](https://blog.csdn.net/hustspy1990/article/details/78048493)

[换个姿势，十分钟拿下Java/Kotlin泛型](https://juejin.cn/post/7133125347905634311)

## 代理

- 动态代理为什么要写接口

1. 生成的代理类继承了Proxy，由于java是单继承，所以只能实现接口，通过接口实现 
2. 从代理模式的设计来说，充分利用了java的多态特性，也符合基于接口编码的规范

成的代理类：$Proxy0 extends Proxy implements Person，我们看到代理类继承了Proxy类，所以也就决定了java动态代理只能对接口进行代理，Java的继承机制注定了这些动态代理类们无法实现对class的动态代理。

[java动态代理实现与原理详细分析](https://www.cnblogs.com/gonjan-blog/p/6685611.html)









