1. java volatile 关键字    
   a. 可见性，而普通的共享变量不能保证可见性，因为普通共享变量被修改之后，什么时候被写入主存是不确定的，当其他线程去读取时，此时内存中可能还是原来的旧值，因此无法保证可见性   
   b. 有序性：    
     实例化一个对象其实可以分为三个步骤：   
   （1）分配内存空间。

    （2）初始化对象。

    （3）将内存空间的地址赋值给对应的引用。

    但是由于操作系统可以对指令进行重排序，所以上面的过程也可能会变成如下过程：

    （1）分配内存空间。

    （2）将内存空间的地址赋值给对应的引用。

    （3）初始化对象

    如果是这个流程，多线程环境下就可能将一个未初始化的对象引用暴露出来，从而导致不可预料的结果。因此，为了防止这个过程的重排序，我们需要将变量设置为volatile类型的变量。    
   c. 原子性，olatile关键字用于声明简单类型变量，如int、float、 boolean等数据类型。如果这些简单数据类型声明为volatile，对它们的操作就会变成原子级别的。但这有一定的限制 自己操作自己不行： n++ 和 n+=1 不行 （需要使用synchronized），n = m +1 可以
   
   
   
   
   
3. 线程池构造函数的用途
```
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
    this.acc = System.getSecurityManager() == null ?
            null :
            AccessController.getContext();
    this.corePoolSize = corePoolSize;
    this.maximumPoolSize = maximumPoolSize;
    this.workQueue = workQueue;
    this.keepAliveTime = unit.toNanos(keepAliveTime);
    this.threadFactory = threadFactory;
    this.handler = handler;
}

corePoolSize: 线程池核心线程数。
allowCoreThreadTimeOut: 核心线程是否会超时
maximumPoolSize： 最大线程数
TimeUnit：keepAliveTime的时间单位，如TimeUnit.SECONDS。
keepAliveTime：非核心线程的闲置超时时间， 超过这个时间非核心线程就会被回收。
workQueue：线程池中的任务队列
threadFactory：提供创建新线程功能的线程工厂
rejectedExecutionHandler： 无法被线程池处理的任务的处理器，一般是因为任务数超出了workQueue的容量。
```
5. hashmap不同版本的实现
```
HashMap在1.7中和1.6的主要区别：
加入了jdk.map.althashing.threshold这个jdk的参数用来控制是否在扩容时使用String类型的新hash算法。
把1.6的构造方法中对表的初始化挪到了put方法中。
1.6中的tranfer方法对旧表的节点进行置null操作（存在多线程问题），1.7中去掉了。

1.8的HashMap相比于1.7有了很多变化
Entry结构变成了Node结构，hash变量加上了final声明，即不可以进行rehash了
插入节点的方式从头插法变成了尾插法
引入了红黑树
tableSizeFor方法、hash算法等等


```
7. cpu缓存的伪共享
```
缓存一致性协议针对的是最小存取单元：缓存行。依照64字节的缓存行为例，内存中连续的64字节都会被加载到缓存行中，除了目标数据还会有其他数据。

如下图所示，假如变量x和变量y共处在同一缓存行中，core1需要操作变量x，core2需要操作变量y。

core1修改缓存行内的变量x后，按照缓存一致性协议，core2需将缓存行置为失效，core1将最新缓存行数据写回内存。
core2需重新从内存中加载包含变量y的缓存行数据，并放置缓存。如果core2修改变量y，需要core1将缓存行置为失效，core2将最新缓存写回内存。
core1或其他处理器如需操作同一缓存行内的其他数据，同上述步骤。

解决
1. Disruptor,基础类型long在java中占用8字节，在额外填充7个long类型的变量，这样在从内存中获取目标变量放入缓存行时，可以达到缓存行中除了目标变量，剩下都是填充变量（由于无业务含义，其他cpu不会对其进行修改）。曲线救国，解决了缓存行伪共享的问题。思想：空间换时间。
2. Jdk8中引入了@sun.misc.Contended这个注解来解决缓存伪共享问题。使用此注解有一个前提，必须开启JVM参数-XX:-RestrictContended，此注解才会生效。
此注解在一定程度上同样解决了缓存伪共享问题。但底层原理并非缓存行填充，而是通过对对象头内存布局的优化，将那些可能会被同一个线程几乎同时写的字段分组到一起，避免形成竞争，来达到避免伪共享的目的。此处不再铺开讲述，有兴趣的可阅读文章：并发编程网-有助于减少伪共享的@Contended注解和此文开头提及Aleksey Shipilev的这封邮件
```
