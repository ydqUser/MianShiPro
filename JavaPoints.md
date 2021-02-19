<!-- TOC -->

- [Java知识点梳理](#java知识点梳理)
    - [Java](#java)
        - [集合](#集合)
        - [基础](#基础)
        - [多线程](#多线程)
        - [Java数据结构](#java数据结构)
    - [框架](#框架)
        - [Spring](#spring)
        - [Mybatis](#mybatis)
    - [MySql](#mysql)
        - [存储引擎](#存储引擎)
        - [索引](#索引)
        - [锁](#锁)
        - [查询性能优化](#查询性能优化)
        - [事务](#事务)
        - [log](#log)
        - [分库分表](#分库分表)
    - [中间件](#中间件)
        - [Redis--一种数据结构存储系统](#redis--一种数据结构存储系统)
        - [MQ](#mq)
        - [Zookeeper---cp模型](#zookeeper---cp模型)
        - [Nginx](#nginx)
        - [etcd](#etcd)
        - [ElasticSerach](#elasticserach)
    - [分布式](#分布式)
        - [分布式事务](#分布式事务)
        - [分布式锁](#分布式锁)
        - [分布式数据一致性](#分布式数据一致性)
        - [分布式框架-RPC](#分布式框架-rpc)
    - [JVM](#jvm)
        - [内存区域](#内存区域)
        - [类加载机制](#类加载机制)
        - [识别垃圾算法](#识别垃圾算法)
        - [垃圾回收算法](#垃圾回收算法)
        - [为什么新生代要分成eden，s1，s0三个区？ 比例大小为什么是8：1：1？](#为什么新生代要分成edens1s0三个区-比例大小为什么是811)
        - [实战场景](#实战场景)
    - [算法](#算法)
        - [贪心](#贪心)
        - [快排](#快排)
        - [堆排](#堆排)
        - [分治](#分治)
        - [动态规划](#动态规划)
        - [二叉树](#二叉树)
        - [链表反转](#链表反转)
        - [成环](#成环)
        - [环节点](#环节点)
    - [查看是否存在更多持久对象和临时对象](#查看是否存在更多持久对象和临时对象)

<!-- /TOC -->


# Java知识点梳理

## Java

### 集合

- ConcurrentHashMap

	- jdk1.7采用的是segment+hashEntry分段锁实现线程安全

		- Segment继承Reentrantlock，默认segment的大小是16，因此支持16个线程并发

			- put过程先获取到当前segment，如果segment为空则先初始化，使用Reentrantlock加锁，再将值插入

	- jdk1.8之后采用的是synchronized+CAS实现线程安全

		- put过程判断hash数组，如果数组为空则使用CAS+自旋初始化数组，如果当前数组为空是，则通过CAS写入数据

- Hashmap

	- jdk1.7：数组+链表

		- put时采用的是链表头插法，多线程扩容时容易造成链表死循环问题

	- jdk1.8：数组+链表+红黑树

		- put时采用的是链表尾插法，多线程扩容时修复了链表死循环问题

	- put过程

		- 先对key 取hashcode值，然后数组长度-1进行对2取模，找到数组桶位置之后，判断数组是否为空，为空则先初始化数组，然后将元素直接插入数组当中，如果key值相同则覆盖value，如果不相同则插入链表尾部，然后判断数组容量是否大于当前容量x负载因子，超过则扩容为2倍大小，链表超过8会转化为红黑树

			- resize：对key重新计算hashcode然后添加到新数组

	- 扩容机制就是对所有key重新计算hash值，拷贝到新的数组

		- 容量为当前2倍

	- 2的幂

- ArrayList

	- 数组

		- 查询效率更快，增删效率低，线程不安全

- LinkedList

	- 双向链表

		- 新增删除更快，查询效率低

- Vector

	- 内部所有方法都是synchronized修饰的，每个方法单独使用的时候都是线程安全的

### 基础

- IO

	- BIO:同步阻塞

		- jdk1.4之前，使用的是BIO，需要先在服务端启动一个ServerSocket，然后在客户端启动Socket来对服务端进行通信，默认情况下服务端需要对每个请求建立一堆线程等待请求，而客户端发送请求后，先咨询服务端是否有线程响应，如果没有则会一直等待或者遭到拒绝请求，如果有的话，客户端会线程会等待请求结束后才继续执行

			- BIO是一个连接一个线程。

	- NIO：同步非阻塞

		- 解决了BIO的并发问题，但是当线程数过多是，服务端可能会拒绝掉客户端请求，导致系统崩掉

			- 子主题 1

	- AIO：异步非阻塞

		- 进行读写请求是，会先请求api中的read/write方法，两种均为异步方式，先将数据写或读入缓冲区，然后通知程序，并在完成后主动调用回调函数

			- AIO是一个有效请求一个线程。

- tcp三次握手

	- 过程

		- 第一步：client向server发出带有syn的请求，进入sent状态
		- 第二步：server收到client的请求，并向client发出syn+ack的请求确认信息，进入待接收状态
		- 第三步：client确认server收到自己发送的询问请求，建立连接开始传输数据

	- 为什么是三次握手

		- 假如A,B两个序号，第一次A告诉B我的序号是A，B回复A确认之后他是A，并且告诉A我的序号是B，最后A回复B确认他知道了序号B，然后开始建立连接

			- 两次的话确认不了双方的序号，三次以上则多余操作

	- tcp如何保证可靠性？

		- tcp中每个字节的数据都进行了编号，数据进行传输是会校验序列号

- tcp四次挥手

	- 过程

		- 第一步：client发送fin告诉server关闭传输
		- 第二步：server发送fin+ack告诉client确认收到请求，并告诉client你发完了我才会关闭
		- 第三步：server发送fin告诉的client已经传输完了
		- 第四步：client确认传输完成，server端close，client等待4分钟关闭

- HTTP和HTTPS

	- HTTP

		- 超文本无状态传输协议

			- 对用户的操作没有记忆功能

		- 明文传输
		- 属于网络五层模型中运输层协议

	- HTTPS

		- 需要到CA申请证书，具有安全性的ssl、tls加密传输协议，通过http+ssl、tls协议构建加密传输

### 多线程

- synchronized

	- 概念

		- synchronized是java提供的原子性内置锁，这种内置的并且使用者看不到的锁也被称为监视器锁

	- 原理

		- java提供的一个原子性内置锁，使用synchronized之后，会在编译后的同步代码前后加上minitorenter和minitoexit字节码指令，他依赖计算机底层的互斥锁实现，执行minitorenter指令之后会为对象加锁，计数器+1，其他程序进入等待队列，执行minitorexit之后计数器-1。当计数器为0的时候释放锁，其他进程可以获取锁，然后synchronized是排它锁，当一个线程获取锁之后，其他线程必须等待这个线程释放锁

	- 对象头

		- Mark Word（存储对象的hashCode，锁标志位和分代年龄）

			- 包含对象的hashcode，分代年龄，轻量级锁指针，重量级锁指针，gc标记，偏向锁线程id

		- Klass  Point （存储类元数据的指针，虚拟机通过这个指针来识别这个对象是哪个类的实例）
		- Minitor

			- Owner（会指向持有monitor的线程）

	- 特性保证

		- 可重入性（一个线程可以多次执行synchronized，重复获取锁）

			- 对象头中计数器记录获取锁的次数

		- 原子性

			- 单一线程持有

		- 可见性

			- 线程解锁前会把变量值强行刷新到主内存中

		- 有序行

			- as-if-serial

				- java中不管怎么排序都不影响单线程程序的执行结果

	- synchronized和Lock的区别

		- 1.synchronized是jvm一个关键字，Lock是一个API接口
		- 2.synchronized可以锁住方法和代码块，lock只能锁住代码块
		- 3.synchronized自动释放锁，lock需要手动释放
		- 4.synchronized非公平锁，lock可以控制是否公平锁

			- lock通过CAS设置锁资源的state值，相同则可以获取锁

		- 5.synchronized不能知道锁状态，lock可以

	- 锁优化机制

		- jdk之后进行了很大的优化，已经不是一个重量级锁了，优化机制包括自适应锁，自旋锁，偏向锁，锁粗化，锁消除和轻量级锁

			- 自旋锁：有时候锁的占用时间很短，为了避免线程状态来回切换的性能损耗，会让线程进行一个自循环过程
			- 自适应锁：就是自适应的自旋锁，由上一个持有锁线程决定
			- 偏向锁：第一个持有过锁的线程再次获取锁，并且没有其他线程竞争的时候，不用执行加锁和解锁的过程
			- 轻量级锁：CAS方式获取锁
			- 锁消除：如果jvm检测到同步代码块不存在锁竞争过程，则会进行锁消除

		- 锁升级过程：无锁--偏向锁--轻量级锁--重量级锁

			- 1.初次进入同步代码块，通过cas操作Mark word，将对象头中偏向锁标志置为1，当第二次进入同步代码块时，判断这把锁的偏向锁id是不是当前线程，如果是，则直接执行同步代码快内容，没有加锁过程
			- 2.如果第二次进入同步代码块时存在锁竞争，则升级为轻量级锁，没有获取到锁的线程在外进行自选等待
			- 3.当自选时间过长还没获取到锁则升级为重量级锁

- Reentrantlock

	- 概念

		- Lock接口的一个实现类，需要显式获取锁和释放锁

	- 源代码

		- 继承Lock接口，内部类Sync继承AQS，加锁lock（）过程是通过传入参数不同，使用FairSync或者NonFairSync实现的

			- FairSync公平锁
			- NonFairSync非公平锁

	- 原理

		- reentrantlock是基于AQS队列实现的
		- AQS（abstractQueueSynchronizer）：一个用于构建锁和同步工具的框架

			- AQS内部维护一个state状态为，通过CAS改变状态值，加锁成功则状态值设置为1，释放锁则设置状态为0

- CAS

	- 概念

		- CompareAndSwap，比较并交换，通过处理器的指令来保证原子性操作

	- 缺点

		- ABA问题

			- Java中通过AtomincStampReference增加前置和后置标志位解决

		- CAS自选时间过长，对CPU消耗比较大
		- 只能保证当前变量的原子操作

- volatile

	- 可见性
	- 有序性

		- 使用volatile修饰的变量，被所有线程共享，使用内存屏障来防止指令重排，解决了内存可见性问题

- ThreadLocal

	- 概念

		- 线程本地变量副本，每个线程都维护一个本地变量副本，实现线程安全，采用空间换时间方式

	- 原理

		- 每个threadLocal维护一个threadLocalMap表，key为threadLocal本身，value才是存储的数据object，key使用的弱引用方式，所以当发生gc的时候回存在key为null的entry，所以需要使用remove方法避免内存泄露问题

- 线程池

	- 关键参数

		- 1.最大线程数 maxPoolSize
		- 2.核心线程数 corePoolSize
		- 3.工作队列  workquene

			- LinkedBlockingQueue

				- 无界 当心内存溢出

			- ArrayBlockingQueue

				- 有界队列，加锁保证安全，队列满了一直阻塞，空了则会唤醒线程

		- 4.线程存活时间   keepALiveTime
		- 5.拒绝策略   rejectedExcutionHandle

	- 为什么先放阻塞队列，队列满了之后再去开非核心线程？

		- 线程池创建的时候需要获取mainlock锁，会影响并发效率，使用阻塞队列将第一步和第三部隔开，起一个缓存作用
		- 引入阻塞队列是为了在执行execute的时候避免获取全局锁

	- 线程池种类

		- newFixedThredPool
		- newCacheThreadPool
		- newSIngleTheadExecutor
		- newScheduledThewadPool

- JMM

	- 多线程操作下的一系列规范约束

		- 可见性

			- volatile关键字

		- 有序性

			- volatile，synchronized

		- 原子性

			- monitor指令操作

		- happen-before原则

			- 如果线程a和线程b存在happen-before关系，则a线程对b线程内存可见

- juc并发包

	- Atomic原子类

		- AtomicInteger

			- addAndGet
			- compareAndSet

		- AtomicReference

			- 以原子方式更新对象引用

	- locks

		- Lock

			- 相比synchronized更广泛的锁操作

		- Reentrantlock
		- ReadWritelock

			- 读的共享锁和写的独占锁

		- ReentrantReadWritelock（ReadWriteLock接口的实现类）

			- 1.获取顺序，读或写的顺序不强制，但可以实现公平策略
			- 2.重出
			- 3.锁降级，允许写锁降为读锁
			- 4.支持中断锁

	- CountDownLatch

		- 一个同步计数器，允许一个或者多个线程在另一组线程完成之前一直等待
		- CountDownLatch.await()

			- 让多个参与者启动后阻塞，然后在主线程调用countDownLatch.countdown()方法将计数减为0，实现多个线程在同一时刻并发执行

	- CyclicBarrier

		- 一种屏障，栅栏

			- 一组线程会相互等待，直到所有线程都到达一个同步点

	- Semaphore

		- 信号量

			- 用来控制同一时间，资源的可被访问线程数

	- CopyOnWriteArrayList

		- CopyOnWriteArrayList适合于多线程场景下使用，其采用读写分离的思想，读操作不上锁，写操作上锁，且写操作效率较低
		- CopyOnWriteArrayList基于fail-safe机制，每次修改都会在原先基础上复制一份，修改完毕后在进行替换
		- add操作是通过lock加锁，然后复制一个新的数组，使用新的数组替换老数组

	- 多线程相关类

### Java数据结构

- tree

	- 二叉树

		- 每个节点最多只有两个子节点的树

			- 集合堆

	- 二叉搜索树

		- 在二叉树的基础上，若左树不为空，则左树上的节点都小于根节点的值，若右树不为空，则右树上的值都大于根节点

	- 二叉平衡树

		- 所有左右子树高度差小于1的二叉树

	- B树

		- 根节点至少有两个子节点，所有叶子节点都在同一层

	- B+树

		- 叶子节点内可以存储数据

## 框架

### Spring

- IOC：控制反转

	- 思想

		- 反转

			- 将自主创建对象变成交给Spring管理

		- 控制什么？

			- 创建管理bean的权利，控制整个bean的生命周期

	- 实践

		- DI依赖注入

			- 配置文件讲外部资源注入到内部，容器加载了外部资源文件，对象，然后把这些资源注入到程序内部对象中，维护了程序内外对象之间的依赖关系

- AOP：面向切面编程

	- 定义：将那些与业务无关，却被多个模块共同调用的东西，封装起来，减少代码的重复
	- 实现原理

		- jdk动态代理：利用反射为需要增强的方法生成代理类，调用invocationHandle的invoke执行相应方法

			- 基于接口

		- cglib动态代理：利用asm开源包，生成一个需要代理类的子类，子类中拦截父类中的方法

			- 基于类

	- Spring Aop和AspectJ AOP有啥区别

		- Spring AOP是运行时增强，AspectJ AOP是编译时增强

- Bean生命周期

	- 实例化bean对象
	- 如果涉及到属性值，则通过set方法对属性进行赋值
	- 判断是否实现Aware接口，实现则调用相关方法，比如BeanNameAware，BeanClassLoaderAware
	- 通过BeanPostProcessor类的postProcessBeforeInitialization进行Bean的前置通知处理
	- 初始化Bean
	- 判断配置文件中是否包含init-method属性，有则执行相关方法
	- 通过BeanPostProcessor类的postProcessAfterInitialization进行Bean的后置通知处理
	- 判断是否实现Disposable接口，有则执行destroy方法销毁bean

- 容器启动流程
- 用到的设计模式

	- 单例模式

		- 对象默认单例

	- 工厂模式

		- BeanFactory生产Bean过程

	- 代理模式

		- SpringAop基于代理模式

	- 模板方法

		- JdbcTemplate

	- 适配器模式

		- SpringAOP的增加或通知，用到了适配器模式

	- 包装器模式

		- 配置多个数据源时使用

- 事务隔离级别

	- 比数据库事务隔离级别多一种默认级别

- 事务传播行为

	- 支持当前事务情况

		- required

			- 如果已经存在事务，则加入当前事务，如果没有则新建事务

		- support

			- 如果已经存在事务则加入当前事务，如果不存在则以非事务方式运行

		- mandatory

			- 如果已经存在则加入当前事务，如果不存在则抛出异常

	- 不支持当前事务的情况

		- requires-new

			- 新建一个事务，如果当前已存在事务则将当前事务挂起

		- not_support

			- 以非事务方式运行，如果当前存在事务，则将当前事务挂起

		- never

			- 以非事务方式运行，如果当前存在事务则抛出异常

	- 其他情况

		- nested

			- 如果当前存在事务，则创建一个事务作为当前事务的嵌套事务执行，如果不存在事务则为require

- SpringMvc

	- 用户请求进来，交给dispatchServlet处理
	- dispatchServlet根据请求调用handleMapping解析对应的handle
	- 解析到相应的handle（controller）后，由handleAdapter根据handle处理相应业务逻辑
	- 处理完成之后返回ModelAndView视图
	- 视图解析器会将view封装成jsp返回到dispatchServlet在返回给用户

### Mybatis

- 基本配置

	- Mybatis配置文件
	- mapper.xml映射文件
	- 通过SQLSessionFactory创建SQLSession

- XML映射DAO接口

	- dao接口的方法名会映射Mapper文件中的segment的id值，每个mapper文件中的sql标签都会解析成一个mapperSegment文件

- 分页
- xml文件中id是否可以重复

	- 原因就是namespace+id是作为Map<String, MapperStatement>的key使用的，如果没有namespace，就剩下id，那么，id重复会导致数据互相覆盖。有了namespace，自然id就可以重复，namespace不同，namespace+id自然也就不同。

- 一二级缓存

## MySql

### 存储引擎

- Innodb

	- 支持行锁
	- 支持事务
	- 支持在线热备份

- MylSAM

	- 支持表锁
	- 不支持事务
	- 支持空间数据索引

- InnoDB与MylSAM比较

	- 事务：innodb不支持，Mylsam支持事务
	- 锁：innodb支持行锁，mylsam支持表锁
	- innodb支持在线热备份
	- mylsam支持空间索引

### 索引

- B+树原理

	- 数据结构

		- B+树是B树的一种变形，他是基于B树和叶子节点顺序访问指针实现的，通常用于数据库和操作系统的文件系统中

			- B树是一种自平衡树数据结构

		- B+树有两种类型节点，内部节点（也称索引节点）和叶子节点，内部节点不存储数据，只存储索引，叶子节点
		- 内部节点的key都按照从小到大的顺序排列，对于内部节点一个key，左边树中所有key都小于它，右边树种的key都大于他，叶子节点中的记录数据也按照key从小到大排序
		- 每个叶子节点都存有相邻叶子节点的指针

	- 操作

		- 查询
		- 插入
		- 删除

	- 常见树的特性

		- AVL树：平衡二叉树，高度平衡，相比红黑树查询更快，删除效率慢，rebalance更高
		- 红黑树：近似平衡的，rebalance的概率更低
		- B/B+树：多路查找树，磁盘io低，一般用于数据库系统中

	- B+树和红黑树的比较

		- 磁盘IO次数低

			- B+树一个叶子节点可以存储多个元素，相对平衡二叉树高度更低，所以磁盘IO次数更低
			- 磁盘预读特性

	- B+树和B树比较

		- B+树的磁盘IO低
		- B+树相对B树（所有非叶子节点最多拥有两个儿子），冗余了一些非叶子节点，提高了范围查询效率

- MySQL索引

	- B+树索引

		- 二分查找，查询效率快
		- B+树有序，排序，分组效率快
		- 聚簇索引：就是按照每张表的主键构建一颗B+树，同时叶子节点存放的数据就是整张表的行记录数据

			- 叶子节点的数据域记录完整的数据记录

		- 辅助索引

			- 叶子节点的数据域记录这主键的值，如果查询不是索引列或者主键，则需要回表

				- 回表：回表就是先通过数据库索引扫描出数据所在的行,再通过行主键id取出索引中未提供的数据,即基于非主键索引的查询需要多扫描一棵索引树

	- 哈希索引

		- 快速精确查询，但不支持范围查询
		- 适合场景：等值查询场景，例如 Redis，memcache等这些NoSQL中间件
		- 哈希表

			- 一种key-value存储各种类型数据的数据结构

	- 全文索引

		- 只有MylSAM支持

	- 空间数据索引

		- 优点在于范围查找

- 索引优化

	- 独立的列

		- 索引不能是表达式的一部分，也不能是函数的参数

	- 多列索引

		- 在需要使用多个列作为查询条件是，简历多列联合索引比单列索引性能更好

	- 索引列的顺序

		- 将选择行最强的索引列放在最前面
		- 索引的选择性：不重复的索引和记录总数的比值

	- 前缀索引

		- 对于BLOG,TEXT和VARCHAR类型的数据，必须使用前缀索引，只索引开始的那部分字符

	- 覆盖索引

		- 定义：索引包含所有需要查询的字段的值
		- 优点

			- 索引通常小于数据行的大小，只读取索引能大大减少数据量访问
			- 无需回表

	- 联合索引

		- 最左匹配原则

			- 比如创建（b，c，d）三列联合索引，会先使用b列建立索引，如果b列值相同则使用c列

		- 相当于建立了三个索引

- 索引的优点
- 索引的使用条件

### 锁

- 表锁
- 行锁
- 间隙锁
- 全局锁
- 读写锁

	- 读

		- for update
		- 行锁

### 查询性能优化

- 使用Explain分析Select查询语句

	- select_type

		- SIMPLE  简单查询
		- UNION   联合查询
		- SUBQUERY  子查询

	- table

		- 查询的表

	- type

		- system

			- the table has only one row,this is a special case of  the const join type

		- const

			- 只有一条查询结果&主键、唯一索引

		- eq_ref

			- 连接查询&主键/唯一索引只有一条查询结果

		- ref

			- 非唯一索引

		- range

			- 使用索引进行范围查询

		- index

			- 查询字段是索引的一部分，覆盖索引
			- 使用主键排序

		- all

			- 全表扫描

	- possible_keys

		- 可选择的索引

	- key

		- 实际使用的索引

	- rows

		- 扫描的行数

- 优化数据访问

	- 减少请求的数据量

		- 只查询必要的列
		- 只返回必要的行
		- 缓存重复查询的数据，例如使用redis缓存用户登录的数据

- 重构查询方式

	- 做分页，分批查询

### 事务

- ACID

	- 原子性   Atomincity

		- 事务被视为最小不可分割单元，一个事务内所有操作要么全都成功要么全都失败

	- 隔离性

		- 一个事务在最终提交前，对其他事务是不可见的

	- 一致性

		- 数据库在事务执行前后都保持一致性，在一致性状态下，所有事务读取到的结果都一直

	- 持久性

		- 一旦事务提交，其所作的事情将永久保存在数据库中

- ACID之间的关系
- 隔离级别

	- READ UNCOMMITED-读未提交

		- 最低级别

	- READ COMMITED-读已提交

		- 读时加共享锁，有更新操作是会转换为排它锁

	- REPEATABLE READ-可重复读（mysql默认隔离级别）

		- 读时加共享锁，写时加排它锁

	- SERIALIZABLE-串行化

- 锁

	- 锁类型

		- 共享锁

			- 允许事务读一行数据

		- 排它锁

			- 允许事务删除或者更新一行数据

		- 意向共享锁
		- 意向排它锁

	- MVCC多版本并发控制

		- 每个用户连接数据库是，读取到的数据都是数据库某一时刻的快照版本，b事务未提交之前，a始终读取到的是某一特定时刻的数据库快照，直到b事务提交才会读到b事务的修改内容
		- 实现方式

			- 1.将数据记录的多个版本保存在数据库当中，当这些数据不需要是，垃圾收集器回收这些数据
			- 2.只在数据库保存最新版本数据，但是在使用undo时动态重构旧版本数据

	- 锁算法

		- Record Lock

			- 单个行记录上的锁

		- Gap Lock

			- 间隙锁，锁定一个范围内的数据

		- Next-key Lock

			- Record Lock + Gap Lock,锁定一个范围数据，也锁定当前记录本身

	- 锁问题

		- 脏读
		- 不可重复读
		- 幻读
		- 丢失更新

### log

- undo log

	- 事务回滚，mvcc

- redo log

	- 记录内存操作记录

- bin log

### 分库分表

- 水平切分
- 垂直切分

## 中间件

### Redis--一种数据结构存储系统

- redis为什么快

	- 单进程单线程，操作基于内存，io多路复用模型，非阻塞io

		- 子主题 1

- 存储数据结构

	- String

		- 简单key-value

	- Hash

		- 类似map结构，一般是将一种结构化的数据（比如一个对象）缓存在redis，写缓存的时候可以操作hash的某个字段

	- Set

		- 无序集合，自动去重

	- SortSet

		- 排序set

	- List

		- lrange实现高性能分页

- 部署方式

	- 主从模式
	- 哨兵模式
	- 集群模式

- 数据同步机制

	- 主从同步

		- 基于指令流，offset

	- 快照同步

		- 通过RDB文件同步

- 持久化方式

	- RDB：不定时将内存写入磁盘存储,fork子进程去做持久化的，比AOF速度要快

		- 突然宕机会导致数据不是最新的

	- AOF：将每一个写和删操作都持久化到日志当中

- 内存淘汰机制

	- 定期删除

		- 默认每100ms就去扫描一次时候存在时间过期的key，如果存在则直接删除

	- 惰性删除

		- 查询的时候如果key过期就删除掉

	- 定时删除

		- 创建定时器删除，消耗cpu资源

	- LRU

		- 删除最少使用的key

- redis大key问题

	- 使用hash函数来分桶，将数据分散到多个桶中，减少单个key大小

- Redis分布式锁

	- setnx指令（SET IF NOT EXIST）

		- jedis.set(lockKey, requestId, SET_IF_NOT_EXIST, SET_WITH_EXPIRE_TIME, expireTime)
		- 1.使用key加锁，因为key唯一
		- 2.value：requetid  判断所得持有者

	- 线程a被挂起，超时释放锁，然后b获取锁，写数据，a线程唤醒，写不了数据问题

		- 引入锁续约机制

- 缓存

	- 缓存穿透：使用一个数据库中不存在数据的key发送大量请求

		- 解决方案：1.value不存在也去缓存这个key         2.使用布隆过滤器

			- Bloom Filter原理

				- 当一个元素被加入集合时，通过K个散列函数将这个元素映射成一个位数组中的K个点，把它们置为1。检索时，我们只要看看这些点是不是都是1就（大约）知道集合中有没有它了：如果这些点有任何一个0，则被检元素一定不在；如果都是1，则被检元素很可能在。这就是布隆过滤器的基本思想

	- 缓存击穿：当key失效时，大量请求进来

		- 解决方案：加锁，当线程获取到锁是，将数据写入到缓存当中，后续线程则直接返回缓存数据
		- 如果数据为热点数据的话，可以设置缓存不失效，然后每次更新值的同时去更新缓存

	- 缓存雪崩：大量缓存同时失效

		- 解决方案：将所有的key分时段失效

- redis和memcache区别

	- memcache

		- 利用多核的优势，效率非常高
		- 可以存储图片，视频等数据，但是都是key-value形式存储
		- 数据无法持久化，重启服务器则数据全都丢失

### MQ

- RocketMq：一个纯Java，分布式，队列模型的开元消息中间件，阿里自研

	- 核心组成模块

		- broker

			- 接受生产者发来的消息并存储，消费者从这里获取消息

		- namesrv

			- 保存消息的TopicName

		- Consumer
		- producer

			- 连接2个broker

	- 完整的通信流程

		- producer和broker简历一个长连接
		- 定期从nameserver获取Topic消息
		- 向Master Broker发送心跳
		- 发送消息给Master Broker
		- Consumer从master或者slave获取消息

	- 优点

		- 单机吞吐量高，可用性，消息可靠性，支持10级消息堆积，不会因为消息堆积对性能产生影响

	- 重复消费问题

		- Consumer消费成功会返回一个Consumer_success标志

	- 消息去重问题

		- 建立一个消息表，每条消息增加一个id，id作为表的主键

	- 高可用

		- Nameserver集群
		- Broker主从

- Kafka：基于zookeeper的天然分布式消息队列

	- 结构组成

		- Broker，Topic，Partition，producer，Consumer

			- 一台kafka服务器称为一个broker，一个topic下有多个partition，这些partition分布在不同的broker上，所以kafka是天然分布式消息队列

	- kafka为什么快

		- 架构层面

			- 一个topic多partition并行处理
			- ISR副本同步机制

		- 磁盘优化

			- partition顺序写入磁盘
			- mmap

				- 操作系统内核到应用程序存储空间不需要经过任何复制。相对于传统读写少了一次复制

		- 网络优化

			- sendfile实现零拷贝，减少上下文切换
			- producer批量发送

	- 生产消费流程

		- produce将消息丢入broker中，broker将消息分布在不同的partition中，consumer从partition中消费消息，每个Consumer对每个Topic都有一个offset来标记读取到哪条数据

	- 消息丢失问题

		- ack值

			- 0：消息发送完之后不用确认是否成功收到
			- 1：消息发送之后，确保leader收到消息
			- all或者-1：消息发送之后确保leader收到，并且所有存在同步的follower完成消息备份后

	- 重复消费问题

		- offset属性

			- 保存在消费端

	- 高可用

		- 文件存储机制

			- partition分成一个个segment文件存储在硬盘上，

		- 复制原理和同步方式

			- ISR队列

				- kafka写操作只在副节点，leader会维持一个isr同步队列，将一些慢响应follower剔除，将一些追上leader数据的副本加进来

	- kafka对zookeeper的依赖

		- 探测borker和Consumer的添加和删除
		- 负责维护partition的从属者关系，如果主分区者挂了需要选出新的主分区者
		- 维护topic和partition的关系

### Zookeeper---cp模型

- 是什么？

	- 主要服务于分布式系统，可以用来做服务注册中心，配置管理中心，分布式锁等待

- 数据结构

	- ZNode

		- 短暂，临时： 当客户端与服务端断开连接，会自动删除节点
		- 持久： 当客户端与服务端断开连接，不会删除节点

- 分布式锁

	- Zookeeper 允许临时创建有序的子节点，这样客户端获取节点列表时，就能够当前子节点列表中的序号是否是最小的判断是否能够获得锁

- 选举机制

	- 如果创建的都是带序号的点，zookeeper每次会选择序号最小的节点称为master，如果master挂掉了则会删除节点，在选取最小序号节点成为新的master

- zab协议

	- 消息广播

		- leader将写请求包装成一个事务，并添加全局唯一的事务id，并广播发送到follower，超过半数follower节点确认过之后就提交事务

	- 崩溃回复

		- leader节点宕机之后，会通过投票选举Zxid最大的server为leader节点，

	- 数据同步

		- 新的leader会将获得的最新的一份proposal副本同步给所有follower

- 如何实现分布式一致性

### Nginx

- 正向代理

	- 客户端请求转发到指定服务器

- 反向代理

	- 以代理服务器方式接受请求，然后请求转发到内部服务器，并将从内部服务器获取到的结果返回出去

- 负载均衡策略

	- 指定权重
	- 最少访问次数
	- 响应时间
	- ip_hash
	- url_hash

### etcd

### ElasticSerach

- 定义：一个实时的分布式分析，搜索，存储引擎
- 为什么要使用elasticsearch？

	- name like%A%模糊查询是不走索引的，数据库数据量大了之后效率非常低

- 数据结构

	- 倒排索引

- 核心概念

	- index ：相当于数据库的table
	- type：新版本已废除
	- document：相当于数据库一行记录
	- field：相当于cloumn
	- Mapping：相当于schema

- 查询操作

	- 创建TransportClient

- 架构

	- 一个主节点，多个从节点
	- 主分片可以写入，读取，副分片可以读取
	- 一个index，多个分片提高吞吐率
	- 副片主要为了实现高可用

## 分布式

### 分布式事务

- 数据库层面事务

	- 2pc提交协议：一种强一致性协议，通过引入一个事务协调者的角色来调节事务的参与者的资源提交和回滚，二阶段分别指准备和提交两个阶段，具有阻塞性，并且协调者存在单点故障

		- 准备：协调者给各个参与者发送准备命令，除了提交事务其他事情都做完了，资源锁定了
		- 提交：当所有参与者都返回成功，则提交或者回滚事务
		- *  如果第一阶段提交成功第二阶段提交失败

			- 1.第二阶段是提交事务：失败这会不断重试，直到提交成功，最后还是失败则人工介入
			- 2.第二阶段如果是回滚事务：会不断重试，直到所有参与者都回滚成功，不然会一直阻塞

	- 3pc提交协议：3PC 的出现是为了解决 2PC 的一些问题，相比于 2PC 它在参与者中也引入了超时机制，并且新增了一个阶段使得参与者可以利用这一个阶段统一各自的状态

		- 准备阶段：不会取锁定资源，首先会询问参与者是否有条件接受这个事务
		- 预提交阶段：统一参与者的状态的一个作用
		- 提交阶段：
		- 引入了参与者超时的机制

- 业务层面事务

	- TCC:Try--Confirm--Concle

		- 实现比较复杂，业务耦合性比较大

- 使用场景

	- 1.电商系统中，用户下单购买商品之后，订单表需要新增一条数据，对应的商品库存数-1
	- 2.A向B转账，a账户-100，b账户+100
	- 3.用户使用积分兑换礼品

### 分布式锁

- zookeeper

	- zookeeper创建临时顺序节点，判断是否属于序号最小节点，是的话则获取锁，锁释放则删除节点
	- 性能没redis高

- redis

	- jedis.set(String key, String value, String nxxx, String expx, int time)
	- 性能比较高

- 数据库

	- 简历一张主键id表，运用死锁，排它锁。比较耗费资源

### 分布式数据一致性

### 分布式框架-RPC

- grpc

	- 常见调用模式

		- 简单模式

			- 客户端发一次请求，服务端响应一次参数

		- 服务端数据流模式

			- 客户端发一次请求，服务端发送一段持续的数据流

		- 客服端流数据模式

			- 客户端持续向服务端发送数据流，结束之后服务端只返回一个响应

		- 双向数据流模式

			- 客户端和服务端实现实时交互

	- 使用方式

		- 定义接口文档
		- 工具生成服务端、客户端代码
		- 服务端业务代码
		- 客户端建立grpc连接后，使用自动生成代码调用函数

	- 基于protobuf实现的接口定义和约束

		- protobuf相对传统的json的序列化协议来说，会有几十倍的性能提升

	- 客户端和服务端都会维护一个stub，stub负责封装命令和参数，并且加载实现约定好的协议文件，比如调用哪个服务的哪个方法，然后通过本地的rpc将网络包发送到服务端，服务端stub对这个包进行解码，获取接口和参数这些信息

- SpringCloud----ap模型

	- Eureka：主管服务注册与发现

		- Eureka Server：提供服务注册服务，各节点启动后，会在Eureka Server上进行注册，服务注册表中会存储所有可用节点信息
		- Eureka Client: java客户端，具备一个内置的使用轮询负载均衡算法的负载均衡器，应用启动后，默认每30秒会向Eureka Server发送心跳，如果Eureka Server在90秒内未收到客户端心跳，将会将该节点从服务注册表中移除 
		- 自我保护机制

			- 如果在15分钟内，超过85%的节点都没有心跳，Eureka会认为客户端和注册中心之间存在网络故障，这时候Eureka不再移除未检测到心跳的节点，同时会接受新节点的注册，但是不会同步到其他节点上，保证该节点可用

		- 如何保证可用性

			- Eureka各个节点都是平等的，几个节点挂掉不会影响正常节点的工作，如果Eureka客户端进行服务注册时发现Eureka Server连接失败会自动切换到其他节点

	- Ribbon：负载均衡工具，服务对服务间的负载均衡

		- 负载均衡策略

			- 随机
			- 轮询
			- 一致性hash
			- hash
			- 权重

		- Ribbon和Nginx区别

			- Ribbon是先将服务注册表缓存到本地，然后在实现相应的负载均衡策略，即在客户端实现负载均衡。Nginx则是客户端将请求交给Nginx去实现负载均衡，属于服务端负载均衡

	- Feign：服务调用方式，内部集成Ribbon，是一个声明式的web服务端，只需要实现一个接口，加上Feign注解之后就可以直接调用方法
	- Zuul：服务网关，通过一定的路由配置，判断某一个url由哪个服务来处理，包含路由和过滤两个功能，是各种服务的统一入口，外部请求的负载均衡

		- 核心主要是通过一系列的过滤器filters实现的，过滤器之间没有直接通信，而是通过Request Context上下文进行数据传递

			- 过滤器种类

				- PRE:前置过滤器

					- 在请求到路由之前拦截。一般使用这种过滤器进行鉴权，限流，参数校验等

				- ROUTING：路由

					- 将请求路由到微服务，构建发送给微服务的请求，然后由ribbon转发到微服务

				- POST:后置

					- 这种过滤器在请求转发到路由之后执行，用来为响应添加标准的返回

				- ERROR:错误

					- 在其他阶段发生错误是执行

		- zuul和nginx区别

			- zuul实现了认真鉴权，动态路由等，在没有专门负责的路由开发时可以使用

		- zuul和ribbon的区别

			- zuul相当于是controller层请求的路由，ribbon相当于service层请求路由

	- Hystrix：保证在一个依赖出问题的情况下整体服务不会失败，避免了级联故障。为了解决雪崩效应

		- 服务雪崩

			- 假设微服务a调用服务b和服务c，服务b调用服务c和其他服务，调用链上面某个服务间的调用超时或者不可用，则调用a服务占用的资源会越来越多，进而引起系统崩溃，这就是雪崩效应

		- 服务熔断

			- 一种对服务雪崩的微服务链路保护机制，当链路中某个服务不可以或者响应时间过长时，快速返回错误信息

		- 服务降级

			- 整体资源不够用了，先将某些服务关掉，后面再开启回来

	- config：一套集中式的，动态的配置管理设施服务

		- 可以获取git仓库的配置文件，给其他服务使用，从而将各个服务的配置在git仓库中统一管理
		- 缺点：git中修改了配置项，无法实时更新，需要重启config_server服务

## JVM

### 内存区域

- 堆

	- 并发情况下，给对象分配内存过程中如何保证线程安全

		- TLAB：虚拟机在堆内存的eden区划分出来的一块线程专用空间，在线程初始化的时候会为每个线程分配一块线程私有TLAB空间，在这上面进行内存分配

			- TLAB空间其他线程可以读取，但是不能在上面分配内存

		- TLAB存在的问题

			- 1.对象需要分配的内存超过TLAB的剩余空间大小，则直接在堆中对该对象进行内存分配

				- 虚拟机定义了一个refill_waste值--最大浪费空间，当对象需要分配的内存大于refill_waste值时会在堆中分配内存，当小于这个值是会抛弃当前TLAB，重新申请TLAB空间

			- 2.对象需要分配的内存超过TLAB的剩余空间大小，则重新申请TLAB空间

	- 永久代/方法区

		- 受jvm管理，容易发生oom，java8之后放入元空间，不受jvm管理

- 虚拟机栈
- 本地方法栈
- 本地内存

	- 线程共享区域，堆外内存

		- 内存回收机制

			- 堆外内存的申请和释放

				- JDK的byteBuffer类提供allocateDirect，底层通过unsafe.allocateMemory资源申请，freeMemory进行资源释放

- 程序计数器

	- 程序执行的字节码的行号指示器

### 类加载机制

- 加载

	- 通过类名获取二进制字节流，将字节流中的存储结构转换为运行时数据结构，然后在内存中能成一个class对象，作为访问的入口

- 连接

	- 验证

		- 文件格式验证
		- 元数据验证
		- 字节码验证
		- 符号引用验证

	- 准备

		- 对类变量进行内存分配，并进行值的初始化

	- 解析

		- 将符号引用替换成直接引用

- 初始化

	- 类初始化，执行类构造器，执行类加载器clinit方法

### 识别垃圾算法

- 引用计数法：对象被引用则计数器+1，为0则对象没有被引用

	- 存在循环引用问题

- 可达性分析算法：从GC Root节点出发，往下遍历，如果对象不存在任何一条调用链上则可以回收

	- GC Root对象

		- 虚拟机栈中引用对象
		- 常量
		- 方法静态变量
		- native方法修饰的对象

	- 如果对象不可达，则GC之前会判断对象是否执行finalize方法，如果未执行则会通过finalize方法将对象与GC Root关联

### 垃圾回收算法

- 标记清楚

	- 产生大量碎片

- 复制算法

	- 浪费内存空间

- 标记整理

	- 每次垃圾回收都会频繁移动对象，效率低下

- 分代收集算法

	- 年轻代

		- Minor GC

			- 存活对象转移到s0或s1，新生代超过存活超过15次gc则会变成老年（新生代对象占用s区域一半大小也会直接变成老年代），

	- 老年代

		- Full GC

			- full gc会回收新生代和老年代（整个堆），会造成Stop The World，性能消耗很大

				- STW：gc期间所有线程挂起，只有gc线程工作

- 垃圾收集器

	- 在新生代工作的回收器

		- serial

			- 单线程垃圾收集器，进行垃圾回收的时候回造成STW，client模式下默认垃圾回收器

		- parNew

			- 多线程垃圾收集器，进行垃圾回收的时候减少了造成STW的时间，server模式下默认垃圾回收器

		-  ParallelScavenge

			- 多线程，使用复制算法回收垃圾，

	- 在老年代工作的回收器

		- Serial Old

			- 单线程垃圾收集器，进行垃圾回收的时候回造成STW，client模式下默认垃圾回收器

		- Parallel Old

			-  ParallelScavenge的老年代版本，使用标记整理算法

		- CMS

			- 以实现最短STW时间为目标的垃圾收集器，采用的是标记清除算法
			- 分代

				- 年轻代

					- eden
					- s1
					- s2
					- minor gc

				- 老年代

					- full gc

				- 永久代

			- 缺点

				- 基于标记清楚算法，产生大量碎片
				- 对cpu资源较敏感

	- G1收集器

		- 分区概念，弱化分代
		- 标记整理算法，分配大对象前不会提前full gc
		- 充分利用cpu资源，减少stw停顿时间
		- 收集步骤

			- 初始标记，从gc roots开始可达对象
			- 可达性分析算法找出存活对象
			- 最终标记
			- 筛选回收

### 为什么新生代要分成eden，s1，s0三个区？ 比例大小为什么是8：1：1？

- 因为eden区每次minor GC的时候对象都直接转为老年代的话老年代很快就会占满，会触发full gc，所以分成三个区域
- 新生代内存达到80%的时候会进行minor gc，所以为8：1：1

### 实战场景

- 性能优化

	- 设置堆的最大值最小值
	- 原则就是减少gc stw

- full gc内存泄露问题排查

	- jasvism
	- dump
	- 监控配置 自动dump

## 算法

### 贪心

- 获取能实现结果的最优解

  package greedy;
  
  import java.util.ArrayList;
  import java.util.HashMap;
  import java.util.HashSet;
  
  /**
   * @program: text
   * @description: 贪婪算法
   * @author: min
   * @create: 2019-10-17 19:50
   **/
  public class GreedyAlgorithm {
    public static void main(String[] args) {
      //创建广播电台,放入到 Map
      HashMap<String, HashSet<String>> broadcasts = new HashMap<String, HashSet<String>>(); //将各个电台放入到 broadcasts
      HashSet<String> hashSet1 = new HashSet<String>();
      hashSet1.add("北京");
      hashSet1.add("上海");
      hashSet1.add("天津");
      HashSet<String> hashSet2 = new HashSet<String>();
      hashSet2.add("广州");
      hashSet2.add("北京");
      hashSet2.add("深圳");
      HashSet<String> hashSet3 = new HashSet<String>();
      hashSet3.add("成都");
      hashSet3.add("上海");
      hashSet3.add("杭州");
      HashSet<String> hashSet4 = new HashSet<String>();
      hashSet4.add("上海");
      hashSet4.add("天津");
      HashSet<String> hashSet5 = new HashSet<String>();
      hashSet5.add("杭州");
      hashSet5.add("大连");
      //加入到 map
      broadcasts.put("K1", hashSet1);
      broadcasts.put("K2", hashSet2);
      broadcasts.put("K3", hashSet3);
      broadcasts.put("K4", hashSet4);
      broadcasts.put("K5", hashSet5);
      //allAreas 存放所有的地区
      HashSet<String> allAreas = new HashSet<String>();
      allAreas.add("北京");
      allAreas.add("上海");
      allAreas.add("天津");
      allAreas.add("广州");
      allAreas.add("深圳");
      allAreas.add("成都");
      allAreas.add("杭州");
      allAreas.add("大连");
      //创建 ArrayList, 存放选择的电台集合
      ArrayList<String> selects = new ArrayList<String>();
      //定义一个临时的集合， 在遍历的过程中，存放遍历过程中的电台覆盖的地区和当前还没有覆盖的地区的交集
      HashSet<String> tempSet = new HashSet<String>();
      //定义给 maxKey ， 保存在一次遍历过程中，能够覆盖最大未覆盖的地区对应的电台的 key
      //如果 maxKey 不为 null , 则会加入到 selects
      String maxKey = null;
      while (allAreas.size() != 0) { // 如果 allAreas 不为 0, 则表示还没有覆盖到所有的地区
        //每进行一次 while,需要
        maxKey = null;
        //遍历 broadcasts, 取出对应 key
        for (String key : broadcasts.keySet()) {
          //每进行一次 for
          tempSet.clear();
          //当前这个 key 能够覆盖的地区
          HashSet<String> areas = broadcasts.get(key);
          tempSet.addAll(areas);
          //求出 tempSet 和 allAreas 集合的交集, 交集会赋给 tempSet
          tempSet.retainAll(allAreas);
          //如果当前这个集合包含的未覆盖地区的数量，比 maxKey 指向的集合地区还多
          //就需要重置 maxKey
          // tempSet.size() >broadcasts.get(maxKey).size()) 体现出贪心算法的特点,每次都选择最优的
          if (tempSet.size() > 0 &&
              (maxKey == null || tempSet.size() > broadcasts.get(maxKey).size())) {
            maxKey = key;
          }
        }
  
        //maxKey != null, 就应该将 maxKey 加入 selects
        if (maxKey != null) {
          selects.add(maxKey);
          //将 maxKey 指向的广播电台覆盖的地区，从 allAreas 去掉
          allAreas.removeAll(broadcasts.get(maxKey));
        }
      }
      System.out.println("得到的选择结果是" + selects);//[K1,K2,K3,K5]
    }
  }

### 快排

- 递归循环，交换位置

  public class QuickSort {
  	
  	/**
  	 * 根据下标交换数组的两个元素
  	 * @param arr 数组
  	 * @param index1 下标1
  	 * @param index2 下标2
  	 */
  	public static void swap(int[] arr, int index1, int index2) {
  		int temp = arr[index1];
  		arr[index1] = arr[index2];
  		arr[index2] = temp;
  	}
  	
  	/**
  	 * 递归循环实现快排
  	 * @param arr 数组
  	 * @param startIndex 快排的开始下标
  	 * @param endIndex 快排的结束下标
  	 */
  	public static void quickSort(int[] arr, int startIndex, int endIndex) {
  		if(arr != null && arr.length > 0) {
  			int start = startIndex, end = endIndex;
  			//target是本次循环要排序的元素，每次循环都是确定一个元素的排序位置，这个元素都是开始下标对应的元素
  			int target = arr[startIndex];
  			//开始循环，从两头往中间循环，相遇后循环结束
  			while(start<end) {
  				//从右向左循环比较，如果比target小，就和target交换位置，让所有比target小的元素到target的左边去
  				while(start < end) {
  					if(arr[end] < target) {
  						swap(arr, start, end);
  						break;
  					}else {
  						end--;
  					}
  				}
  				
  				//从左向右循环比较，如果比target大，就和target交换位置，让所有比target大的元素到target的右边去
  				while(start < end) {
  					if(arr[start] > target) {
  						swap(arr, start, end);
  						break;
  					}else {
  						start++;
  					}
  				}
  			}
  			//确定target的排序后，如果target左边还有元素，继续递归排序
  			if((start-1)>startIndex) {
  				quickSort(arr, startIndex, start-1);
  			}
  			//确定target的排序后，如果target右边还有元素，继续递归排序
  			if((end+1)<endIndex) {
  				quickSort(arr, end+1, endIndex);
  			}
  		}
  	}
  	
  	public static void main(String[] args) {
  		int[] arr = new int[]{4,1,8,5,3,2,9,10,6,7};
  		quickSort(arr,0,9);
  		for (int i = 0; i < arr.length; i++) {
  			System.out.print(arr[i]+",");
  		}
  	}
  }

### 堆排

- 初始化一个堆，然后通过不断调节父子节点，最大的数放在堆尾，最小放头节点

### 分治

### 动态规划

### 二叉树

### 链表反转

### 成环 

### 环节点

## 查看是否存在更多持久对象和临时对象

*XMind - Trial Version*