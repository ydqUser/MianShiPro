
<!-- TOC -->

- [Thread](#thread)
- [锁](#锁)
    - [死锁 活锁 饥饿](#死锁-活锁-饥饿)
    - [Synchronised](#synchronised)
    - [wait/notify](#waitnotify)
    - [lock](#lock)
- [volatile](#volatile)
    - [原子性、可见性、有序性](#原子性可见性有序性)
    - [语义](#语义)
    - [原理和实现机制](#原理和实现机制)
    - [使用场景](#使用场景)
- [ThreadLocal](#threadlocal)
        - [说明](#说明)
        - [设计思想](#设计思想)
        - [使用场景](#使用场景-1)
    - [注意事项](#注意事项)
        - [重要方法](#重要方法)
- [AQS](#aqs)
    - [AQS](#aqs-1)
    - [CountDownLatch](#countdownlatch)
- [线程池](#线程池)
    - [问题](#问题)
- [HashMap和ThreadLocalMap区别](#hashmap和threadlocalmap区别)

<!-- /TOC -->

lock接口在多线程和并发编程中最大的优势是它们为读和写分别提供了锁


https://www.cnblogs.com/dolphin0520/category/1426288.html

#### Thread

![image](https://gitee.com/lylw/image/raw/master/Thread/Thread.jpg)

https://www.cnblogs.com/dolphin0520/p/3920357.html

#### 锁

##### 死锁 活锁 饥饿

- 死锁：指两个或两个以上的进程（或线程）在执行过程中，因争夺资源而造成的一种互相等待的现象，若无外力作用，它们都将无法推进下去。
    - 产生死锁的必要条件：
        1、互斥条件：所谓互斥就是进程在某一时间内独占资源。
        2、请求与保持条件：一个进程因请求资源而阻塞时，对已获得的资源保持不放。
        3、不剥夺条件:进程已获得资源，在末使用完之前，不能强行剥夺。
        4、循环等待条件:若干进程之间形成一种头尾相接的循环等待资源关系。
- 活锁：任务或者执行者没有被阻塞，由于某些条件没有满足，导致一直重复尝试，失败，尝试，失败。
    - 活锁和死锁的区别在于，处于活锁的实体是在不断的改变状态，所谓的“活”， 而处于死锁的实体表现为等待；活锁有可能自行解开，死锁则不能。
- 饥饿：一个或者多个线程因为种种原因无法获得所需要的资源，导致一直无法执行的状态。
    - Java 中导致饥饿的原因：
        - 1、高优先级线程吞噬所有的低优先级线程的 CPU 时间。
        - 2、线程被永久堵塞在一个等待进入同步块的状态，因为其他线程总是能在它之前持续地对该同步块进行访问。
        - 3、线程在等待一个本身也处于永久等待完成的对象(比如调用这个对象的 wait 方法)，因为其他线程总是被持续地获得唤醒。
5、Java 中用到的线程调度算法是什么？
采用时间片轮转的方式。可以设置线程的优先级，会映射到下层的系统上面的优先级上，如非特别需要，尽量不要用，防止线程饥饿。

##### Synchronised

##### wait/notify

- `wait` 
- `notify/notifyall`

wait/notify/notifyall的判断条件要放到while循环中，不能用if，否则会发生虚假唤醒的情况
##### lock

#### volatile

refer：[关键字解析](https://www.cnblogs.com/dolphin0520/p/3920373.html)

##### 原子性、可见性、有序性
- 原子性：一个操作或多个操作，要么全部执行并且执行的过程不会被任何因素打断，要么就都不执行
- 可见性：多个线程修改一个变量时，一个线程修改了这个变量的值，其他线程能够立即看得到修改的值
- 有序性：程序执行的顺序按照代码的先后顺序执行。指令重排序不会影响单个线程的执行，但是会影响到线程并发执行的正确性

java内存模型提供的保证
- 原子性：对基本数据类型变量的读取和赋值操作是原子性操作。`x++和 x = x+1包括3个操作：读取x的值，进行加1操作，写入新的值`
- 可见性：volatile保证可见性
- 有序性：重排序过程不会影响到单线程程序的执行，却会影响到多线程并发执行的正确性。volatile关键字来保证一定的“有序性”

java内存模型具备一些先天的有序性，即happens-before原则。如果两个操作的执行顺序不能由happens-before原则推导出来，那么就不能保证有序性。happens-before原则(先行发生原则)：
- 程序次序规则：一个线程内，按照代码顺序，书写在前面的操作先行发生于书写在后面的操作。可能会指令重排，但会保证单线程执行结果的正确性
- 锁定规则：一个unlock操作先行发生于后面对同一个锁的lock操作
- volatile变量规则：对一个变量的写操作先行发生于后面对这个变量的读操作
- 传递规则：如果操作A先行发生于操作B，而操作B又先行发生于操作C，则可以得出操作A先行发生于操作C
- 线程启动规则：Thread对象的start()方法先行发生于此线程的每个一个动作
- 线程中断规则：对线程interrupt()方法的调用先行发生于被中断线程的代码检测到中断事件的发生
- 线程终结规则：线程中所有的操作都先行发生于线程的终止检测，我们可以通过Thread.join()方法结束、Thread.isAlive()的返回值手段检测到线程已经终止执行
- 对象终结规则：一个对象的初始化完成先行发生于他的finalize()方法的开始

##### 语义
两层语义：
- 保证了不同线程对这个变量进行操作时的可见性
- 禁止指令重排序

as-if-serial语义：不管怎么重排序，单线程程序的执行结果不能被改变

变量使用volatile修饰后：
- 使用volatile关键字会强制将修改的值立即写入主存
- 使用volatile关键字修饰，某一线程对变量修改时，会导致其他线程工作内存中缓存变量的缓存行无效
- 由于其他线程工作内存中缓存变量的缓存行无效，所以再次读取变量时会去主存读取

volatile的原子性、可见性、有序性：
- volatile无法保证原子性，可以采用synchronized、Lock、AtomicInteger
- volatile能在一定程度上保证有序性。

volatile禁止指令重排有两层意思：
- 当程序执行到volatile变量的读操作或者写操作时，在其前面的操作的更改肯定全部已经进行，且结果已经对后面的操作可见；在其后面的操作肯定还没有进行；
- 在进行指令优化时，不能将在对volatile变量访问的语句放在其后面执行，也不能把volatile变量后面的语句放到其前面执行

##### 原理和实现机制
观察加入volatile关键字和没有加入volatile关键字时所生成的汇编代码发现，加入volatile关键字时，会多出一个lock前缀指令

lock前缀指令实际上相当于一个内存屏障（也成内存栅栏），内存屏障会提供3个功能
- 它确保指令重排序时不会把其后面的指令排到内存屏障之前的位置，也不会把前面的指令排到内存屏障的后面；即在执行到内存屏障这句指令时，在它前面的操作已经全部完成
- 它会强制将对缓存的修改操作立即写入主存
- 如果是写操作，它会导致其他CPU中对应的缓存行无效

##### 使用场景
需要具备两个条件：
- 对变量的写操作不依赖于当前值
- 该变量没有包含在具有其他变量的不变式中

使用场景：
- 状态标记量
- double check

JMM(java内存模型)，它主要是为了解决多线程下的共享内存操作问题，为了保证数据的一致性，我们在自己的工作内存操作修改变量后，会提交到主内存中进行覆盖，并且使其他线程中工作内存中的共享变量删除，使得其他线程在自己的工作内存中访问不到该共享变量副本，只能到主内存中去访问。这样就很好的保证了数据的可见性


#### ThreadLocal

###### 说明
线程本地变量。每个Thread有一个ThreadLocal.ThreadLocalMap threadLocals 

[ThreadLocal解析](https://juejin.cn/post/6844904090506362887)

[ThreadLocal源码解析](https://juejin.cn/post/6844903974454329358)

###### 设计思想
每个Thread维护一个ThreadLocal.ThreadLocalMap哈希表，这个哈希表的key是ThreadLocal实例本身，value是要存储的值

![ThreadLocal设计说明](https://gitee.com/lylw/image/raw/master/ThreadLocal/ThreadLocal_design.jpeg)

- 计算hash index的值
    - `int h = k.threadLocalHashCode & (len - 1)` k为ThreadLocal对象，len为ThreadLocalMap数组的长度
    - `private final int threadLocalHashCode = nextHashCode();`
    - `private static int nextHashCode() {
        return nextHashCode.getAndAdd(HASH_INCREMENT);
    }`  nextHashCode为一个static的AtomicInteger变量，每次一个新的ThreadLocal对象计算hash值时，都会加HASH_INCREMENT
    - private static AtomicInteger nextHashCode = new AtomicInteger();

- 如何解决hash冲突
    - 采用了开放定址法。没有使用HashMap的链地址法。开放定址法：根据hashcode计算获得数组地址下标时，如果该位置已经被占用了，那么向后一位或多位进行判断
- 采用开放定址法的原因：
    - ThreadLocal有一个神奇的属性`HASH_INCREMENT = 0x61c88647`,并利用AtomicInteger进行累加，能够将哈希值均匀地分布在2的N次方的数组里
    - ThreadLocal存放的数据量不会特别大，而且key是弱引用，会被垃圾回收，采用开放定址法会更省空间，而且查询效率高
- 为什么使用弱引用
    - 如果key使用强引用，会出现一个问题，引用的ThreadLocal的对象被回收了，但是ThreadLocalMap还持有ThreadLocal的强引用，如果没有手动删除，ThreadLocal不会被回收，则会导致内存泄漏。  
    - 如果key使用弱引用，引用的ThreadLocal的对象被回收了，由于ThreadLocalMap持有ThreadLocal的弱引用，即使没有手动删除，ThreadLocal也会被回收
- key是弱引用，会被垃圾回收，value是强引用，不会被垃圾回收，会发生内存泄漏。解决办法：
    - `get`、`set`和`remove`方法中都会清除无效的Entry
- 如何扩容
    - ThreadLocalMap中有一个阈值`threshold=table长度*2/3`。当`size>=threshold`时，遍历table并删除key为null的元素，如果删除后`size>=threshold*3/4`时，需要进行扩容操作
    - 扩容操作比较简单，但是会先判断key是否为null,如果为null,将对应的value也设置为null，帮助gc
- ThreadLocal内存溢出问题
    - `expungeStaleEntry`是帮助垃圾回收的，get和set方法都有调`expungeStaleEntry`方法，但是不调get和set的时候就会可能面临着内存溢出。不再使用的时候调用`remove`方法，加快垃圾回收，避免内存溢出
    - 如果没有调用get/set/remove方法，线程结束的时候，也就没有强引用再指向ThreadLocalMap了，但是危险的是，如果是线程池的，线程执行完代码的时候并没有结束，只是归还给线程池，这时候ThreadLocalMap 和里面的元素是不会回收掉的

###### 使用场景
- 数据库的连接，如果不用ThreadLocal,一个线程执行查询操作，一个线程却先执行了关闭操作，显然这样是不行的。但是也有人问，直接在每个方法中自己设置了一连接就行了，但是这样会导致服务器压力大，并且严重影响程序执行性能。
- 存储单个线程上下文信息，比如存储id等
- 减少参数传递。比如做一个trace工具，输出工程处理过程的全部信息，由于在工程中各处随时取用，所以可以放入ThreadLocal
##### 注意事项
- 线程复用会造成脏数据
- 不及时remove会造成数据泄露
- 方法：及时remove

###### 重要方法
- get
    - 获取当前线程的ThreadLocalMap,以当前的ThreadLocal为key,调用getEntry()查找，如果找到，就返回该值
    - getEntry方法中，如果存在hash冲突，则根据规则向后遍历，如果遇到key为null，则调用expungeStaleEntry方法；如果找到，即返回
    - 如果没找到entry，如果当前map不为空的话，则设置当前ThreadLocal为key的value为null.(其实value是根据initialValue方法初始化的，该方法默认实现为返回null)
    - 如果map为空，则要创建一个map,并设置当前ThreadLocal为key,value为null
- set
    - 根据hash计算数组中的位置下标，如果当前位置存在节点且与key相等，则进行值覆盖
    - ++**如果该位置key为null**++，则调用`replaceStaleEntry`(说明被回收，需要替换掉被回收的值，新的值放在这里)
    - 如果该位置没有存放节点，则新建一个Entry实例，并执行`cleanSomeSlots(i, sz)`方法，并判断是否扩容
- expungeStaleEntry(int staleSlot)
    - 对于在staleSlot和下一个null的entry之间 的entry 做处理操作 
    - key为null的，value/entry置null，size--；key不为null的，重新计算hash值，放入正确的地方
    - 返回staleSlot之后entry为null的index
- replaceStaleEntry(ThreadLocal<?> key, Object value, int i) 
    - 参数说明：只有set的时候会调此方法，当tab[i].get()为null时调用。key/value为要set的键值对，i为index
    - 替换的原因：如果不替换，会把新的值放入，这样会存在两个相同的key
- cleanSomeSlots(int i, int len)
    - 从i向后遍历，寻找过期的对象进行清除
    - 方法的循环结束条件是n>>>1!=0,也就是这个方法在没有遇到过期对象的时候，会执行log2(n)的扫描。这里没有选择扫描全部是为了性能的平衡


<details>
  <summary><font>源码解读</font></summary>
  <pre><code>
  
  <h5>expungeStaleEntry</h5>
  
    /**
     * 对于在staleSlot和下一个null的entry之间 的entry 做处理操作 
     * 如果key为null，则value置null，entry置null，size--
     * 如果key不为null，则重新计算hash值，将entry放在正确的地方，其现在的entry置null
     *
     * @param staleSlot 有null key的index
     * @return staleSlot之后 下一个为null的entry的index值 (所有在staleSlot和这个值之间的都会被check是否要清除)
     */

private int expungeStaleEntry(int staleSlot) {

    Entry[] tab = table;
    int len = tab.length;
    
    // expunge entry at staleSlot
    tab[staleSlot].value = null;
    tab[staleSlot] = null;
    size--;

    // Rehash until we encounter null
    Entry e;
    int i;
    for (i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
        if (k == null) {
            e.value = null;
            tab[i] = null;
            size--;
        } else {
            // 如果该entry key不为null，则计算key的hash值是否为此index，
            // 如果不是，则说明该位置的entry之前存放时是存在hash冲突的，则将该位置entry置为null，根据新计算的hash值放入entry
            int h = k.threadLocalHashCode & (len - 1);
            if (h != i) { 
                tab[i] = null;

                while (tab[h] != null)
                    h = nextIndex(h, len);
                tab[h] = e;
            }
        }
    }
    return i;
}  
  
  
<h5>replaceStaleEntry</h5>

/*
     * @param  key the key
     * @param  value the value to be associated with key
     * @param  staleSlot index of the first stale entry encountered while
     *         searching for key. key通过hash计算的index位置
     */
private void replaceStaleEntry(ThreadLocal<?> key, Object value, int staleSlot) {

    Entry[] tab = table;
    int len = tab.length;
    Entry e;
    
    // 向前找 entry !=null 但是 key == null的index作为slotToExpunge
    // 从staleSlot往前遍历，目的是把前面已经被回收的也一起释放出来(寻找key被回收，value未回收，entry未回收的)
    // 同时也避免存在很多过期的对象占用，导致这个时候刚好来了一个新的元素达到阈值而触发一次新的rehash
    int slotToExpunge = staleSlot;
    for (int i = prevIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = prevIndex(i, len))
        if (e.get() == null)
            slotToExpunge = i;

    // 向后遍历，这两个遍历为了在左边找到第一个空的entry，右边遇到的第一个空的entry之间 查询所有过期的对象
    // 注意：在右边如果找到需要设置值的key相同的时候就开始清理，然后返回，不再继续遍历下去
    for (int i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();

        // 如果找到了key值，说明之前已经存在相同的key，则为了保持hash table 的顺序，需要交换i 和 staleSlot的位置。
        if (k == key) {
            e.value = value;

            tab[i] = tab[staleSlot];
            tab[staleSlot] = e;

            // 说明上面第一个往前的循环没有发现过期的key
            // staleSlot的位置过期，也因为前面已经将前面过期的对象的位置交换到index=i上了,所以需要清理的位置是i
            if (slotToExpunge == staleSlot)
                slotToExpunge = i;
                
            // 清理过期数据    
            cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
            return;
        }

        // 如果第一个向前遍历的时候没有找到任何过期的对象，那么需要把slotToExpunge设置为向后遍历的第一个对象的位置
        // 因为如果整个数组都没有找到要设置的key的时候，该key会在staleSlot的位置上；如果找到了，上面交换的时候也把有效值移到staleSlot上
        // 综上所述，staleSlot位置上不管怎么样，存放的都是有效的值，所以不需要清理的
        if (k == null && slotToExpunge == staleSlot)
            slotToExpunge = i;
    }

    // 如果key在数组中没有存在，新建一个放进去
    tab[staleSlot].value = null;
    tab[staleSlot] = new Entry(key, value);

    // 如果有其他已经过期的对象，那么需要清理
    if (slotToExpunge != staleSlot)
        cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
}
<h5>replaceStaleEntry</h5>

// 从i开始往后遍历，寻找过期对象进行清除
private boolean cleanSomeSlots(int i, int n) {

    boolean removed = false;
    Entry[] tab = table;
    int len = tab.length;
    do { // 保证do被执行一次
        i = nextIndex(i, len);
        Entry e = tab[i];
        
        // 如果遇到过期对象的时候，重新赋值n = len即数组长度
        if (e != null && e.get() == null) {
            n = len;
            removed = true;
            i = expungeStaleEntry(i); // 每次调用expungeStaleEntry来帮助垃圾回收
        }
    } while ( (n >>>= 1) != 0); //无符号右移动一位，可以简单理解为除以2
    // 在没有遇到过期对象的时候，会执行log2(n)扫描，这里没有选择全部扫描是为了性能的平衡
    return removed;
}
  </code></pre>
</details>

#### AQS

##### AQS

`共享状态`属于 AQS 中的概念，在 AQS 中分为两种模式，一种是`独占模式`，一种是 `共享模式`。



##### CountDownLatch

#### 线程池

[refer](https://blog.csdn.net/breakout_alex/article/details/105454980)

线程池的优点：
- 重用存在并空闲的线程，减少对象创建、消亡的开销，性能佳。
- 可有效控制最大并发线程数，提高系统资源的使用率，同时避免过多资源竞争，避免堵塞。
- 提供定时执行、定期执行、单线程、并发数控制等功能。

java线程池的常见参数：
- corePoolSize：核心线程数量，会一直存在，除非allowCoreThreadTimeOut设置为true
- maximumPoolSize：线程池允许的最大线程池数量
- keepAliveTime：线程数量超过corePoolSize，空闲线程的最大超时时间
- unit：超时时间的单位
- workQueue：工作队列，保存未执行的Runnable 任务
- threadFactory：创建线程的工厂类
- handler：当线程已满，工作队列也满了的时候，会被调用。被用来实现各种拒绝策略。

执行流程
![执行流程](https://gitee.com/lylw/image/raw/master/Thread/thread_pool_process.jpeg)

拒绝策略：
- AbortPolicy(抛出一个异常，默认的)
- DiscardPolicy(直接丢弃任务)
- DiscardOldestPolicy（丢弃队列里最老的任务，将当前这个任务继续提交给线程池）
- CallerRunsPolicy（交给线程池调用所在的线程进行处理)

线程池异常处理
- 在任务代码中try/catch捕获异常
- submit执行，通过Future对象的get方法接收抛出的异常
- 实例化时，传入自己的ThreadFactory，为线程设置UncaughtExceptionHandler，在uncaughtException方法中处理异常
- 重写ThreadPoolExecutor的afterExecute方法，处理传递的异常引用 

<details>
  <summary>线程池异常处理示例</summary>
  <pre><code> 
  
     public static void testFuture() {
         ExecutorService threadPool = Executors.newFixedThreadPool(10);
         for (int i=0; i<10; i++) {
             Future future = threadPool.submit(() -> {
                 List<String> lists = new ArrayList<String>();
                 System.out.println(lists.get(1));
             });
             try {
                 future.get();
             } catch (Exception e) {
                 e.printStackTrace();
             }
         }
     }
         
         
     // 为线程设置setUncaughtExceptionHandler    
     public static void testExceptionHandler(){
         ExecutorService threadPool = Executors.newSingleThreadExecutor(r->{
             Thread t = new Thread(r);
             t.setUncaughtExceptionHandler((t1,e) -> {
                 System.out.println(t1.getName() +e);
             });
             return t;
         });
         threadPool.execute(()->{
             List<String> lists = new ArrayList<String>();
             System.out.println(lists.get(1));
         });
     }
     
     
     // 重写afterExecute方法
     class ExtendedExecutor extends ThreadPoolExecutor {
         // 这可是jdk文档里面给的例子。。
         protected void afterExecute(Runnable r, Throwable t) {
             super.afterExecute(r, t);
             if (t == null && r instanceof Future<?>) {
                 try {
                     Object result = ((Future<?>) r).get();
                 } catch (CancellationException ce) {
                     t = ce;
                 } catch (ExecutionException ee) {
                     t = ee.getCause();
                 } catch (InterruptedException ie) {
                     Thread.currentThread().interrupt(); // ignore/reset
                 }
             }
             if (t != null)
                 System.out.println(t);
         }
     }}
  </code></pre>
</details>


各种阻塞队列的说明(都是线程安全的)：
- ArrayBlockingQueue：一个由数组结构组成的有界阻塞队列 FIFO。 
- LinkedBlockingQueue：一个由链表结构组成的阻塞队列 FIFO，容量可以选择设置。不设置的话，将是无边界的阻塞队列，**吞吐量通常高于ArrayBlockingQueue**；newFixedThreadPool使用了这个队列 
- PriorityBlockingQueue：一个支持优先级排序的无界阻塞队列。 
- DelayQueue：延迟队列。根据指定的执行时间从小到大排序，否则根据插入到队列的先后排序。newScheduledThreadPool使用 
- SynchronousQueue：同步队列，一个不存储元素的阻塞队列，每个插入操作必须等到另一个线程调用移除操作，否则插入操作一直处于阻塞状态，吞吐量通常要高于LinkedBlockingQuene，newCachedThreadPool线程池使用了这个队列。 
- LinkedTransferQueue：一个由链表结构组成的无界阻塞队列
- LinkedBlockingDeque：一个由链表结构组成的双向阻塞队列

> 使用无界队列会导致内存飙升【建议不要直接通过Executors静态工厂构建线程池，而是通过ThreadPoolExecutor的方式，这样的处理方式让写的同学更加明确线程池的运行规则，规避资源耗尽的风险】

线程池种类：
- newCachedThreadPool创建一个可缓存线程的线程池，如果线程池长度超过处理需要，可灵活回收空闲线程，若无可回收，则新建线程。
    - 使用SynchronousQueue，是一个没有容量的阻塞队列。每个插入操作必须等待另一个线程的对应移除操作。这意味着，如果主线程提交任务的速度高于线程池中处理任务的速度时，CachedThreadPool会不断创建新线程。极端情况下，CachedThreadPool会因为创建过多线程而耗尽CPU资源
- newFixedThreadPool 创建一个定长线程池，可控制线程最大并发数，超出的线程会在队列中等待。
- newScheduledThreadPool 创建一个周期线程池，支持定时及周期性任务执行。
- newSingleThreadExecutor 创建一个单线程化的线程池，它只会用唯一的工作线程来执行任务，保证所有任务按照指定顺序(FIFO, LIFO, 优先级)执行。





##### 问题
- Executor和Executors区别
    - Executors工具类的不同方法按照我们的需求创建了不同的线程池，来满足业务的需求。
    - Executor 接口对象能执行我们的线程任务。
    - ExecutorService 接口继承了 Executor 接口并进行了扩展，提供了更多的方法我们能获得任务执行的状态并且可以获取任务的返回值。
    - 使用 ThreadPoolExecutor 可以创建自定义线程池。
    - Future 表示异步计算的结果，他提供了检查计算是否完成的方法，以等待计算的完成，并可以使用 get()方法获取计算的结果。
     
     

#### HashMap和ThreadLocalMap区别

数据结构区别
- HashMap是数组+链表(红黑树)的数据结构，ThreadLocalMap的数据结构仅仅是数组
- HashMap是链地址法解决hash冲突的，ThreadLocalMap是开放地址法解决hash冲突的
- HashMap里面的Entry内部类的引用都是强引用，ThreadLocalMap的Entry类的key是弱引用，value是强引用

链地址法和开放地址法的优缺点
- 开放地址法
    - 容易产生堆积问题，不适于大规模的数据存储
    - 散列函数的设计对冲突会有很大的影响，插入时可能会出现多次冲突的现象
    - 如果删除的元素是多个冲突元素中的一个，需要对后面的元素作处理，实现较复杂
- 链地址法
    - 处理冲突简单，且无堆积现象，平均查找长度短
    - 链表中的结点是动态申请的，适合构造表不能确定长度的情况
    - 删除结点的操作易于实现。只要简单地删去链表上相应的结点即可
    - 指针需要额外的空间，故当结点规模较小时，开放定址法较为节省空间
