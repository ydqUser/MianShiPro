### 蚂蚁金服面试

####      一面

* **ArrayList和LinkedList区别**

  * ArrayList底层数组结构，查询的时候效率比增删效率更高，新增更多数据是集合会动态增长，支持fast-fail
  * LinkedList是双链表结构。增删数据的时候效率更高，支持fast-fail
  
* **什么情况下会造成内存泄露**

  * 内存泄露就是存在一些被分配的对象，这些对象有两个特点:
    * 对象是可达的
    * 对象是无用的
  * 如果满足这两个条件，就判定为内存泄露。这些对象gc时不会被回收，但是又占用着内存
  
* **什么线程死锁？如何解决**

  * 产生死锁的条件有四个：
    * 互斥条件：进程在某一时间独占资源
    * 请求与保持条件：当线程因请求资源阻塞是，对已获得资源保持不变
    * 不剥夺条件：进程已获得资源，在未使用完之前不可剥夺
    * 循环等待条件：若干资源形成一种头尾相接的循环等待关系
  * 要解决死锁问题，要从四个条件出发，破坏其中一个就行
  
* **Tcp的三次握手**

  * 建立连接时，客户端发送syn包到服务器，并进入syn_sent状态，等待服务器确认
  * 服务器确认收到连接请求，并像客户端发送ack包，服务器进入syn_recv
  * 客户端收到服务器发送的ack包，并且像服务器发送ack确认包成功，tcp连接建立成功
  
* **hashmap，怎么扩容，怎么处理数据冲突？怎么高效率的实现数据迁移？**

* 扩容：当超过容量75%的时候，会创建原来两倍大小的数组，然后将原来数据放入新的数组当中，这个过程交过reHash，重新调用hash方法映射数据在数组中的位置，多线程环境下不推荐使用hashmap，扩容时候1.8之前头插法会造成死循环
  
  * 数据冲突：entry存储的是一个数组链，如果hashcode相同，会比较key是否相同，相同的话会将数据插入entry链的最末端
  
* **说一下三次握手和四次挥手**
  * 三次握手
    * 客户端向服务端发出请求，发送syn包，并且客户端进入待发送状态
    * 服务端收到客户端请求，并且发送syn+ack包给客户端，服务端进入待接收状态
    * 客户端收到服务端的确认请求，建立连接
  * 四次挥手
    * 客户端发送断开连接请求给服务端，FIN和Squ,客户端进入半关闭阶段（你太懒了，我要跟你分手）
    * 服务端确认客户端关闭请求，发送确认请求给客户端，ack+1，服务端进入半关闭状态（好，等我收拾好东西）
    * 服务端进入LAST-ACK状态，停止向客户端发送数据，但是仍然可以接受来自客户端的数据（我的东西收拾完了，我们分手吧）
    * 客户端等待2m之后，进入closed状态，停止数据传输（好，。拜拜）
  
* **同步IO和异步IO的区别？**
  * 什么是IO:在Linux中，一切皆文件，文件就是一串二进制流，不管socket，fifo，管道还是终端，一切都是文件，一切都是流，我们对这些流的收发操作，简称IO。
  * 同步阻塞IO：BIO,用户进程被阻塞，直到内核准备好数据，返回结果，用户进场才会接触阻塞，重新运行。
    * 用户进程阻塞不占用CPU资源，适合并发量小的应用
  * 同步非阻塞IO:NIO，当内核没有准备好数据是，会立刻给用户返回一个error，用户收到error之后可以再发起read操作，直到内核数据准备好。相比于阻塞IO的区别在于调用立即返回。
    * 用户进程轮询调用，消耗CPU资源，适用于并发量小且不需要及时响应的应用
  * IO多路复用：多个进程IO可以注册在一个复用器（select）上，用户进程调用改select时，select会监听所有注册进来的io，当所有io都没有可读数据是，select会阻塞进程，当任意io有可读数据是会立即返回。
  * 异步IO：AIO，用户进程发起read之后，会传递三个参数告诉内核当操作完成时怎么通知，之后就去做其他事情。
    * 需要操作系统底层支持，回调机制，实现难度较大，非常适合高性能高并发应用
  
* **Java Gc机制？GC Root有哪些？**
  * jvm中，程序计数器，虚拟机栈，本地方法栈都是随着线程的消失而消失，栈帧的入栈出栈操作实现了内存的自动清理，所以垃圾回收主要集中在堆和方法区，程序运行期间这部分内存的使用都是动态的。
  * 识别垃圾的算法
    * 引用计数法
    * 可达性分析算法
  * 垃圾回收算法
    * 标记清楚算法
      * 造成大量内存碎片
    * 标记整理算法
      * 对象频繁复制移动消耗性能
    * 复制算法
      * 内存使用只有50%
    * 分代收集算法
      * 新生代-经过15次gc转化为老年代
        * Eden，s1，s2，8：1：1
        * YongGC
      * 老年代
        * Full GC
  * GC Root对象
    * 常量
    * 栈中对象
    * 静态属性对象
  
* **红黑树五个特性，插入删除操作，时间复杂度？**

  * 五大特性：
    * 每个节点都是黑色或者红色
    * 根节点都是黑色
    * 每个叶子节点是黑色的
    * 如果一个节点是红色的，则它的子节点必须是黑色的
    * 一个红黑树的中任取一个节点，从它所在位置到其他任何叶子节点的简单路径上所经过的黑色节点数相同。

  * 时间复杂度：O(1) < O(logn) < O(n) < O(n^2)

  

