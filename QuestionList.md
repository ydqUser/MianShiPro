#### 阿里巴巴

**1.对象如何进行深拷贝，clone除外**

* 构造函数：通过调用构造函数进行深拷贝，形参如果是基本类型和字符串则可以直接赋值
* 重载 clone方法，super.clone()其实是浅拷贝，所以在重写User类的clone()方法时，address对象需要调用address.clone()重新赋值。
* Gson序列化：先将对象序列化成json，再将对象反序列化
* Jackson序列化：同上

**2.happen-before原则**

* JMM内存模型通过happen-before原则保证跨线程的内存可见性，如果a线程的写操作和b线程的读操作之间存在happen-before原则，尽管a线程和b线程处于不同的线程中执行，但是jmm保证a操作对b线程的可见性
* java 8大happens-before原则：
  * 单线程happen-before原则：在同一个线程中，书写在前面的操作happen-before后面的操作。
  * 锁的happen-before原则：同一个锁的unlock操作happen-before此锁的lock操作。
  * volatile的happen-before原则：对一个volatile变量的写操作happen-before对此变量的任意操作(当然也包括写操作了)。
  * happen-before的传递性原则：如果A操作 happen-before B操作，B操作happen-before C操作，那么A操作happen-before C操作。
  * 线程启动的happen-before原则：同一个线程的start方法happen-before此线程的其它方法。
  * 线程中断的happen-before原则：对线程interrupt方法的调用happen-before被中断线程的检测到中断发送的代码。
  * 线程终结的happen-before原则：线程中的所有操作都happen-before线程的终止检测。
  * 对象创建的happen-before原则：一个对象的初始化完成先于他的finalize方法调用。

**3.jvm调优的实践**

* 默认配置
  * **-server**：如果tomcat是运行在生产环境中的，这个参数必须加上,-server参数可以使tomcat以server模式运行,这个模式下将拥有：更大、更高的并发处理能力，更快更强捷的JVM垃圾回收机制，可以有更大的负载与吞吐量。
  * **-Xms<size>和-Xmx<size>**：前者表示jvm初始化堆的大小，后者表示jvm堆的最大值，一般把两个值设置成一样是最优做法，否则会导致jvm有较为频繁的GC，影响系统性能。
  * **-XX:MaxPermSize=256m**：初始化jvm非堆（持久代，永久代，方法区）最大值
  * **-Dsun.net.client.defaultConnectTimeout=60000**：连接超时设置
* GC策略调整
  * **-XX:+UseParNewGC**：设置年轻代垃圾收集器为并行收集，可与CMS同时使用
  * **-XX:+UseConcMarkSweepGC**：设置老年代收集器为CMS，CMS流程：初始标记(CMS-initial-mark) -> 并发标记(CMS-concurrent-mark) -> 重新标记(CMS-remark) -> 并发清除(CMS-concurrent-sweep) ->并发重设状态等待下次CMS的触发(CMS-concurrent-reset)。
  * **-Xss**：设置每个线程堆栈大小，一般每个堆栈大小为1M
* 记录GC日志
* 调整eden和survivor内存分配策略

**4.单例对象会不会在gc的时候被回收**

* 单例对象不会被jvm回收，但是单例类会被回收

**5.redis如果list较大，怎么优化**

* string类型大key
  * 将整存整取的大对象拆分成多个key-value，然后使用mget命令获取值

* hash，list，zset，set大key
  * hash类型可以对value的hash取模，分散在不同服务器上
* 删除大key是可以使用scan命令分批删除

**6.redis集群的使用**

* redis单机
* redis主从
  * 保证数据可靠性，读写分离策略
* redis哨兵
  * 实现了高可用特性
  * slave节点不作为备份节点
* redis cluster
  * 基于一致性hash来对数据进行分布

**7.场景题：设计数据库连接池**

* 限制连接池中最多、可以容纳的连接数目，避免过度消耗系统资源。
* 当客户请求连接，而连接池中所有连接都已被占用时，该如何处理呢？一种方式是让客户一直等待，直到有空闲连接，另一种方式是为客户分配一个新的临时连接。
* 当客户不再使用连接，需要把连接重新放回连接池。
* 连接池中允许处于空闲状态的连接的最大项目。假定允许的最长空闲时间为十分钟，并且允许空闲状态的连接最大数目为5，

**8.场景题：秒杀场景的设计**

* 1.针对url，ip，gps进行限流
* 2.商品详情和库存使用缓存，每次下单成功发送消息扣减库存并且去更新缓存
* 3.令牌桶算法限流防止超卖

![image-20210223165229920](C:\Users\86158\AppData\Roaming\Typora\typora-user-images\image-20210223165229920.png)

-------------------------------

#### 美团

**1.OOM故障处理**

* java.lang.OutOfMemoryError:Javaheap space  堆内存oom
  * 原因分析：
    * 请求创建了一个大对象
    * 上游系统流量飙升，通常是各类促销，秒杀活动导致
    * Finalizer对象没有被回收
    * 内存泄露
  * 解决方案：
    * 1.针对大部分情况，可以通过‘-Xmx’参数调高jvm的堆内存空间
    * 如果是超大对象，可以检查其合理性
    * 如果是业务峰值压力，可以考虑添加机器资源，或者限流降级处理
    * 如果是内存泄露，需要找到内存泄露的对象
* java.lang.OutOfMemoryError:GC overhead limit exceeded    
  * 原因：应用程序已经基本耗尽索引可用内存
  * 解决方案：同javaheap space 类似
* java.lang.OutOfMemoryError: Permgen space
  * 1.程序启动报错，修改 -XX:MaxPermSize参数，增加永久代空间
* java.lang.StackOverflowError   
  * 增加栈内存大小

**2.有没有用过分布式锁，怎么实现的，讲讲原理**

* redis 实现分布式锁
  * setNx命令，返回1的话则获取锁，返回0则没有获取锁
  * 需要配合lua脚本删除key
* zk实现分布式锁
  * 创建临时节点，创建节点成功则获取锁，释放锁的时候删除相应节点

**3.redis的跳表用在哪，为什么用跳表**

* 使用场景
  * 跳表可以实现有序集合
* 使用原因
  * 使用跳表按照区间查找数据比红黑树效率要高
  * 对于按照区间查找数据这个操作，跳表可以做到 O(logn) 的时间复杂度定位区间的起点，然后在原始链表中顺序往后遍历就可以了。这样做非常高效。

**4.mysql优化经验**

* 1.字段设计
  * 第一范式：字段原子性，不可分割
    * 关系型数据库，默认满足第一范式
  * 第二范式：消除对主键的部分依赖
    * 即表中加上一个与业务逻辑无关的字段作为主键
  * 第三范式：消除对主键的传递依赖
    * B字段依赖于A，C字段又依赖于B。比如上例中，任课老师是谁取决于是什么课，是什么课又取决于主键`id`。因此需要将此表拆分为两张表日程表和课程表
* 2.存储引擎
  * ![image-20210224165340488](C:\Users\86158\AppData\Roaming\Typora\typora-user-images\image-20210224165340488.png)
* 3.索引
  * 普通索引
  * 唯一索引
  * 主键索引
  * 全文索引
  * hash索引
  * order by后面字段最好建立索引
  * join on涉及的字段加上索引
  * 索引覆盖
    * 如果要查询的字段都建立过索引，那么引擎会直接在索引表中查询而不会访问原始数据（否则只要有一个字段没有建立索引就会做全表扫描），因此尽量再select后面直接必要的查询字段
  * like 查询  
    * 不以%开头，‘like%’形式是可以走索引的
* 4.缓存
  * 缓存失效问题：表做改动是，该表所有的缓存都会被删除
* 5.分库分表
  * mysql表分区算法
    * hash
    * key：对整型数据取模处理
    * range算法：根据条件分区
  * 水平拆分：简历多个结构相同的表存储数据
  * 垂直拆分：将表字段拆分到多个表
* 6.集群
* 7.慢查询sql









#### 腾讯

**1.kafka为什么快**

* 写入数据：
  * kafka会把收到的消息都写入硬盘中，数据不会丢失。为了优化写入速度采用两个技术
    * 顺序写入
    * MMFile-Memory Mapped File
      * kafka数据并不是实时写入硬盘的，利用了操作系统的page来实现文件和物理内存的映射，完成映射之后物理内存的操作才会被同步到硬盘上
* 读取数据
  * 基于sendfile实现Zero Copy
    * 传统模式下：文件数据从硬盘--内核buf--用户buf--socket相关buf
    * sendfile模式下：硬盘--内核buf--socket相关buf
* 总结：kafka速度快的秘诀在于，他把所有的消息都变成了一个批量的文件，并且进行合理的批量压缩，减少网络io，读取数据是配合sendfile暴力输出

**2.kafka生产端怎么实现幂等的**

* kafka为了实现producer的幂等性引入了PID,sequence number
  * 每个新的Producer在初始化的时候会被分配一个唯一的pid，这个id对用户是不可见的
  * 对于每个producer，改producer对应的<Topic, partition>都对应一个从0开始单调递增的sequence number
  * Boker端保存了这个sequence number，对于接收的每条消息，只有比当前sequence number大1才会接收。这样就避免了消息重复提交。但是只能保证单个producer针对对应的topic的幂等性，不能保证不通partition的幂等

* kafka事务
  * 生产者生成消息和消费者提交偏移量在一个事务内，这也避免了重复消费问题

**3.kafka的slave的同步机制**

* kafka副本
  * kafka中的topic每个partition都有一个预写式日志文件，每个partition都由一系列有序的，不可变的消息组成，这些消息被联系追加到partition中，partition每个消息都有一个连续的序列号叫做offset，确定它在分区日志中唯一的位置。
  * kafka的每个topic的partition有n个副本，其中N是topic的复制因子。kafka通过多副本机制实现故障转移，当Kafka集群中一个Broker实现情况下仍然保证服务可用。在kafka中发生复制是确保partition的预写式日志有序的写到其他节点上。N个replicas中，其中一个replica为leader，其他都为follower，leader处理partition的所有读写请求。follower会定期去复制leader上的数据。

**4.kafka怎么进行消息写入的ack**

* ack = 1
  * producer发送消息之后，只要收到分区的leader副本成功写入消息，就会收到来自服务端的响应
* ack = 0
  * producer只要发送完消息就不管了，一般配置ack = 0可以达到最大吞吐量
* ack = -1 、all
  * producer发送消息之后，收到分区所有副本都成功写入的消息才会得到服务端的响应

**5.为什么实现equals方法一定要实现hash方法**

* 如果不重写hashcode方法，那么我们new一个新的对象，当原对象.equals新对象等于true时，两者hashcode不相等，颠覆了认知

**6.new一个对象的过程**

* 1.检查这个类有没有被加载
* 2.在堆或者TLAB空间上为对象分配内存空间
* 3.为对象的实例属性赋值
* 4.设置对象头信息
* 5.对象引用信息入栈
* 6.对象init初始化

**7.类加载过程**

* 加载
  * 将class文件加载到内存当中，反射生成一个Class类对象代表这个类，作为类访问入口
* 验证
  * 验证类加载信息是否符合jvm规范
* 准备
  * 为类变量分配内存并且设置初始值
* 解析
  * 将常量池内的符号引用替换为直接引用
* 初始化
  * 执行类构造器<clinit>（）方法的过程
* java程序初始化顺序
  * 父类静态遍历
  * 父类静态代码块
  * 子类静态变量
  * 子类静待代码块
  * 父类非静态变量
  * 父类非静态代码块
  * 父类构造方法
  * 子类~~
  * 子类~~
  * 子类~~

**8.mysql的事务隔离级别**

* 读未提交
* 读已提交
* 不可重复读
* 串行化
* 事务并发的问题：
  * 脏读：事务a读取了事务b更新的数据，然后b回滚了。a读到的就是脏数据
  * 不可重复读：事务a多次读取同一数据，事务b在a读取过程中修改了数据，导致a读取同一数据是不一致
  * 幻读：系统管理员A将数据库中所有学生的成绩从具体分数改为ABCDE等级，但是系统管理员B就在这个时候插入了一条具体分数的记录，当系统管理员A改结束后发现还有一条记录没有改过来，就好像发生了幻觉一样，这就叫幻读。

#### YY

**1.JVM调优思路**

* 1.内存模型及垃圾收集算法
  * New --年轻代
    * eden
    * s1
    * s2
  * Tenured --老年代
  * 永久代
  * **年轻代和老年代可以通过设置-Xmx参数调整大小。永久代可以通过设置-XX：permSize，-XX:MaxPermSize参数设置大小**
* 垃圾回收算法
  * 标记-清楚算法
    * 利用率高，但是会产生大量内存碎片
  * 标记整理算法
    * 利用率高，无内存碎片，标记和清楚效率都不高
  * 复制算法
    * 内存利用率很低，只有50%的空间
* 新生代：复制算法

|      收集器       |   收集对象和算法 | 收集器类型         |
| :---------------: | ---------------: | ------------------ |
|      Serial       | 新生代，复制算法 | 单线程             |
|      ParNew       | 新生代，复制算法 | 并行的多线程收集器 |
| Parallel Scavenge | 新生代，复制算法 | 并行的多线程收集器 |

* 老年代：标记清除算法和标记整理算法

| 收集器                  | 收集对象和算法                            | 收集器类型         |
| ----------------------- | ----------------------------------------- | ------------------ |
| Serial Old              | 老年代，标记整理算法                      | 单线程             |
| Parallel Old            | 老年代，标记整理算法                      | 并行的多线程收集器 |
| CMS（Conc Mark Sweep ） | 老年代，标记清除算法                      | 并行和并发收集器   |
| G1（Garbage First）     | 跨新生代和老年代，复制算法 + 标记整理算法 | 并行和并发收集器   |

* **注**
  * 并行：垃圾收集的多线程的同时进行
  * 并发：垃圾收集的多线程和应用线程的多线程同时进行



