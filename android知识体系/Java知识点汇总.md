# 多线程并发

- synchronized 修饰 static 方法、普通方法、类、方法块区别
- synchronized 底层实现原理
- volatile 的作用和原理
- 说两个线程同步的集合类
- 构造一个出现死锁的情况
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

**LinkBlockingQueue和ConcurrentHashMap**

以ThreadLocal对象的**弱引用**作为key，ThreadLocal里“存放”的数据作为value，放在该Map中。



**乐观锁**

CAS 其实是一种乐观锁，一般有三个值，分别为：赋值对象，原值，新值，在执行的时候，会先判断内存中的值是否和原值相等，相等的话把新值赋值给对象，否则赋值失败，整个过程都是原子性操作，没有线程安全问题。

[面试官：说说多线程并发问题](https://juejin.cn/post/6844903941830869006)

[面试必备：Kotlin 线程同步的 N 种方法](https://juejin.cn/post/6981952428786597902)

[不可不说的Java“锁”事](https://tech.meituan.com/2018/11/15/java-lock.html)

# Jvm管理

- 对象被设为null会被回收吗

一个对象的引用执行null，并不会被立即回收的，还需要执行`finalize()`方法

1. 堆(heap) : 他是最大的一块区域，用于存放对象实例和数组，是全局共享的.（也称为逻辑堆，主要用来存放对象实例与数组，对于所有的线程来说他是共享的，对于Heap堆区是动态分配内存的，所以空间大小和生命周期都不是明确的，而GC的主要作用就是自动释放逻辑堆里实例对象所占的内存，而在逻辑堆中还分为新生代与老年代，用来区分对象的存活时间，在新生代中还被细致的分为 Eden SurvivorFrom以及SurvivorTo这三部分.）
2. 栈(stack) : 全称为虚拟机栈，主要存储基本数据类型，以及对象的引用，私有线程（在每一个对象被创建的时候，在堆栈区都有一个对他的引用，在这里我们可以这样理解。）每个栈帧存储了方法的局部变量表、操作数栈、动态连接和方法返回地址等信息
3. 方法区(Method Area) : 在class被加载后的一些信息 如常量，静态常量这些被放在这里，在Hotspot里面我们将它称之为永生代（方法区主要存储（类加载器）ClassLoader加载的类信息，在这里我们可以理解为已经编译好的代码储存区，所以存储包括类的元数据，常量池，字段，静态变量与方法内的局部变量以及编译好的字节码，等等）它用于存储已被虚拟机加载的类信息、常量、静态变量、及时编译器编译后的代码等数据

在《Java虚拟机规范中》，对这个区域规定了两种异常状况：

1. 如果线程请求的栈深度大于虚拟机所允许的深度，将抛出 StackOverflowError
2.  如果Java虚拟机栈可以动态扩展，当扩展时无法申请到足够的内存，就会抛出OutOfMemoryError异常

 执行 a = a++ 操作，原先已经执行了 a++ 操作，这个时候将 a++ 中 a 赋值给 int a ，所以会将栈中的数据赋值到 局部变量表中，所以这个时候局部变量表中的数据就是88了

[JVM那点事-对象的自救计划（对象被设为null会被回收吗？）](https://www.jianshu.com/p/0618241f9f44)

[JVM内存结构——运行时数据区](https://www.cnblogs.com/zhengbin/p/5617023.html)

[【死磕JVM】一道面试题引发的“栈帧”！！！](https://www.cnblogs.com/mingyueyy/p/14538754.html)

# 集合

- Java 集合，介绍下ArrayList 和 HashMap 的使用场景，底层实现原理
- ArrayList 与 LinkedList 的区别
- ArrayList、HashMa和LinkedList的默认空间是多少？扩容机制是什么



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

HashMap基于hashing原理，我们通过put()和get()方法储存和获取对象。当我们将键值对传递给put()方法时，它调用键对象的hashCode()方法来计算hashcode，让后找到bucket位置来储存值对象。当获取对象时，通过键对象的equals()方法找到正确的键值对，然后返回值对象。HashMap使用链表来解决碰撞问题，当发生碰撞了，对象将会储存在链表的下一个节点中。 HashMap在每个链表节点中储存键值对对象。

[列举2个线程安全的集合类_Java面试题：Java中的集合及其继承关系](https://blog.csdn.net/weixin_28878185/article/details/113534446)

[《我们一起进大厂》系列-HashMap](https://juejin.cn/post/6844904017269637128)

[吊打面试官》系列-ConcurrentHashMap & Hashtable](https://juejin.cn/post/6844904023003250701)

[hash的问题](https://www.jianshu.com/p/45fa4e80b631)

# Serializable和Parcelable

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


CPU密集型：核心线程数 = CPU核数 + 1
IO密集型：核心线程数 = CPU核数 * 2

注：IO密集型（某大厂实践经验）
    核心线程数 = CPU核数 / （1-阻塞系数）   例如阻塞系数 0.8，CPU核数为4
    则核心线程数为20

[Android 线程和线程池一篇就够了](https://juejin.cn/post/6844903480193187854)

[java线程池如何配置核心线程数](https://www.cnblogs.com/xy-ouyang/p/14718195.html)

# 类加载

其中系统类加载器包括3种，分别是Bootstrap ClassLoader、 Extensions ClassLoader和 App ClassLoader。

## 双亲委托模式

所谓双亲委托模式就是首先判断该Class是否已经加载，如果没有则不是自身去查找而是委托给父加载器进行查找，这样依次的进行递归，直到委托到最顶层的Bootstrap ClassLoader，如果Bootstrap ClassLoader找到了该Class，就会直接返回，如果没找到，则继续依次向下查找，如果还没找到则最后会交由自身去查找。

采取双亲委托模式主要有两点好处：

1. 避免重复加载，如果已经加载过一次Class，就不需要再次加载，而是先从缓存中直接读取。
2. 更加安全，如果不使用双亲委托模式，就可以自定义一个String类来替代系统的String类，这显然会造成安全隐患，采用双亲委托模式会使得系统的String类在Java虚拟机启动时就被加载，也就无法自定义String类来替代系统的String类，除非我们修改
   类加载器搜索类的默认算法。还有一点，只有两个类名一致并且被同一个类加载器加载的类，Java虚拟机才会认为它们是同一个类，想要骗过Java虚拟机显然不会那么容易

[Android解析ClassLoader（一）Java中的ClassLoader](http://liuwangshu.cn/application/classloader/1-java-classloader-.html)

[深入探讨 Java 类加载器](https://blog.csdn.net/gongxiao1993/article/details/81351988)

# Kotlin

- kotlin扩展函数底层原理
- Kotlin inner 关键字

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

[深入理解 Java 泛型：类型擦除、通配符、运行时参数类型获取](https://blog.csdn.net/hustspy1990/article/details/78048493)

## 代理

- 动态代理为什么要写接口

1. 生成的代理类继承了Proxy，由于java是单继承，所以只能实现接口，通过接口实现 
2. 从代理模式的设计来说，充分利用了java的多态特性，也符合基于接口编码的规范

成的代理类：$Proxy0 extends Proxy implements Person，我们看到代理类继承了Proxy类，所以也就决定了java动态代理只能对接口进行代理，Java的继承机制注定了这些动态代理类们无法实现对class的动态代理。

[java动态代理实现与原理详细分析](https://www.cnblogs.com/gonjan-blog/p/6685611.html)









