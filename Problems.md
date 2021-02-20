# 业务问题

## 如何设计一个接口

### 常规数据有效性校验

### 幂等性：对接口的多次调用所产生的结果和调用一次是一致的，数据发生改变则需要做幂等，本质上是分布式锁的问题 

- 按钮只可操作一次
- token机制，进入页面获取token，后续请求都带上token
- 页面重定向
- 数据库实现乐观锁
- select + insert or update or delete

### 安全性

- 数据加密

	- 对称加密

		- DES,AES

	- 非对称加密

		- RSA

- 数据签名
- snowflake
- 黑名单机制

### 限流机制

- 令牌桶限流

	- Guava提供了RateLimiter工具类

- 漏桶限流
- 计数器限流
- 分布式限流

	- redis+lua脚本实现

## 消息队列问题

### 如何保证消息队列的高可用

- RabbitMQ

	- 单机模式
	- 普通集群

		- 多台机器启动多个Rabbitmq实例，你创建的Queue只会存在一个Rabbitmq实例上，但是每个实例会同步queue上的实例数据，消费的时候如果连接的是另外一个实例，那么会从queue所在实例同步数据过来

	- 镜像集群

		- 你创建的queue在每个Rabbitmq上都有一个镜像，每次写消息的时候自动同步到每个实例的镜像文件

			- 缺点：消息同步到所有机器开销太大，queue太大是，负重非常高

- kafka

	- 天然分布式消息队列，一个topic下有多个partition，这些partition分布在不同的broker上，每个partition存储部分信息

		- kafka0.8以前不存在HA机制（高可用），所以只要一个broker宕机之后，这部分数据就丢失了
		- kafka0.8以后，提供了HA，就是replica副本机制，所有replica会选举出一个leader，生产消费都直接操作leader，写操作的时候leader会将数据同步到所有follower上去，当消息同步到所有的follower的时候，会发送ack到leader，消息才能被消费

### 如何保证消息不重复消费（一般在Consumer端控制）

- RabbitMQ

	- 消息唯一id
	- redis set特性

- kafka

	- 每个 Topic 可以划分多个分区（每个 Topic 至少有一个分区），同一 Topic 下的不同分区包含的消息是不同的。每个消息在被添加到分区时，都会被分配一个 offset，它是消息在此分区中的唯一编号，Kafka 通过 offset 保证消息在分区内的顺序，offset 的顺序不跨分区，即 Kafka 只保证在同一个分区内的消息是有序的
- 每个消息写进去都会有一个offset，Consumer消费完数据，每隔一段时间会提交自己消费过的offset，下次消费就从offset后面继续消费
	
	  - 在内存中维护一个set，消费消息之前先去set查询该消息是否存在，存在则表示已经消费，则丢弃
	  - 如果要写数据库，则拿唯一标识先去数据库查，数据库不存在再写入
	  - 生产者发送消息的时候可以加上唯一标识，消费时先将id保存在redis，下次消费先去redis查询是否存在这个标识，没有再消费
	
- kafka多个consumer同时消费一个topic数据

  - 一个topic可以存在多个partition，一个group可以有对个consumer进行消费，一个consumer消费多个partition，一个partition只能被一个consumer消费

  ### 如何解决消息丢失问题

- RabbitMQ（producer端解决）

  - 事务同步机制

  	- 发送数据之前开启channel.txSelect，然后发送消息，如果消息没有被Rabbitmq接收到，生产者会发生异常报错，这时候可以回滚事务channel.txRollBack,然后重试，接受消息成功之后可以提交事务channel.txCommit

  		- 非常消耗性能

  - confirm机制

  	- 每次发送消息会分配一个唯一id，Rabbitmq成功接收消息会返回一个ack，如果接收异常会回调nack接口重试

  - confirm机制

  	- 每次发送消息会分配一个唯一id，Rabbitmq成功接收消息会返回一个ack，如果接收异常会回调nack接口重试

  - queue数据持久化到磁盘

- kafka

  - 关闭自动提交offset机制
  - 设置topic至少用于2个partition副本
  - 设置ack=all

### 如何保证消息的顺序性

- RabbitMQ

	- 拆分多个queue，每个Consumer对应一个queue，然后Consumer内部使用内存队列做排队，然后分发给底层的worker

- kafka

	- 一个topic一个partition，一个Consumer，单线程消费，吞吐量太低，一般不用
	- 写N个内存queue，key相同的数据放在同一个queue，每个线程消费一个queue

### 如何解决消息堆积问题

- 临时紧急扩容，将原先Consumer停掉，然后使用扩容后新的Consumer去消费

## 缓存问题

### 缓存余数据库双写不一致--数据一致性问题

- 最经典缓存+数据库读写的模式

	- 读的时候，先读缓存，缓存没有再读数据库，然后将读出的数据放入缓存
	- 写的时候，先更新数据库，然后删除缓存

- 解决方案

	- 更新数据的时候，根据数据的唯一标识，将请求发送到一个jvm内存队列，读数据的时候如果发现缓存不存在，则执行‘读取数据+更新缓存操作’

### 缓存雪崩

- 事前

	- redis高可用方案，主从+哨兵，redis cluster，避免权限崩盘

- 事中

	- 本地ehcache缓存+hystris限流&降级，避免mysql崩溃

- 事后

	- redis持久化，一旦重启，自动从硬盘恢复数据

### 缓存穿透

- set控制到缓存当中

### 缓存击穿

- 若为热点数据则设置永不过期
- 加锁，数据查询成功之后更新缓存
- 定时任务在缓存过期前主动加载缓存

### 缓存并发竞争

- 多个客户端同一时间写同一个key，更新顺序不一样则会造成数据不一致
- watch命令--redis天然的CAS乐观锁方案

	- 客户端1执行watch命令，由watch监控此期间age值已经被修改，则让整个事务回滚，不错其他操作，watch可以监控多个键，只要有一个被其他客户端修改则回滚

### redis

- redis与memcache的区别

	- redis支持多种数据类型，memcache只支持字符串类型

- 存储类型以及使用场景
- 过期策略以及LRU实现
- redis如何保证高并发，高可用？redis主从复制的原理？redis哨兵原理？
- 有几种持久化方式？不通方式的优缺点
- redis集群模式的工作原理

## 分库分表

### 为什么要分库分表（高并发系统，数据库层面应该如何设计）？

- 分表

	- 单表数据量大，极大影响sql执行性能。单表到几百万的时候性能就会相对差很多了，就需要分表了

		- 分表策略

			- 用户id---表容量，根据userid%100=order获得对应需要操作的表
			- 每张表数据尽量控制在200w左右

- 分库

	- 并发数非常大，单库一般最多支撑到2000并发，一个健康的单库最好控制在1000左右

		- 分库策略

### 分库分表中间件

- sharding-JDBC

	- 支持分表分库，读写分离，分布式id

- MyCat

### 具体是水平拆分还是垂直拆分

- 水平拆分

	- 将一个表的数据分发到多个库的多个表当中，每个表的表结构是相同的，只是每个表存储的数据不一样，所有库表加起来的数据就是完整数据，使用多个库来抗住更多并发

- 垂直拆分

	- 将一个拥有许多字段的表拆分到多个表中，或者多个库中去，每个库表的结构不同，一般将访问频率高的字段放一个表，访问字段频率低的放一个表。因为数据库是有缓存的，访问频率高的行字段越少，性能越好

### 分库分表方式

- range拆分

	- 一般按照时间范围

- hash拆分

### 唯一id

- uuid不适合作为主键

	- uuid太长，占用空间太大，影响性能

		- uuid作为主键在进行写操作的时候，会将整个b+树节点读取到内存当中，然后将操作完的数据写回到内存当中

- snowflake分布式id生成算法

	- 长度为64bit



##### 判断单链表是否有环以及环的入口检测

* 1.辅助空间法

  * 无环：在有限步骤内可以找到链表的尾节点。即，存在一个节点指向空地址

  * 有环：永远无法找到尾节点，并且会在环内循环

  * **解法**：遍历每个节点的时候判断该节点是够重复，如果遍历结束都未出现重复节点则没有环，如果出现重复节点则有环


* 快慢指针法

  * 一个fast指针，一个slow指针，如果两个指针相遇了，说明有环，如果没相遇，说明没有环，fast和slow相遇后，将fast指向头节点，slow从相遇点以和fast相同的速度出发，则两个点再次相遇的地方就是环的入口

        /**
         * 检测链表是否有环，若有环则找到入口
         * 使用快慢指针的方式
         *
         * @param head 链表头节点
         * @return 若为空，则不存在环；不为空，则输出为入口节点
         */
        private static Node detectCircleByFastSlow(Node head) {
            // 快慢指针从头节点开始
            Node fast = head;
            Node slow = head;
            // 用于记录相遇点
            Node encounter = null;
            // fast一次走两步，所以要保证next和next.next都不为空，为空就说明无环
            while (fast.next != null && fast.next.next != null) {
                // fast走两步，slow走一步
                fast = fast.next.next;
                slow = slow.next;
                // fast和slow一样，说明相遇了，或者fast追上slow了。
                if (fast == slow) {
                    // 记录相遇点，fast或slow都一样
                    encounter = fast;
                    // 相遇了，就退出环检测过程
                    break;
                }
            }
        
            // 如果encounter为空，说明没有环，就不用继续找环入口了。
            if (encounter == null) {
                return encounter;
            }
        
            // fast回到head位置
            fast = head;
            // 只要两者不相遇，就一直走下去
            while (fast != slow) {
                // fast每次走一步，slow每次走一步，速度一样
                fast = fast.next;
                slow = slow.next;
            }
        
            // 相遇点，即为环入口
            return fast;
        }



### 如何在接口中优雅引入重试机制

* Spring重试工具包

  ```
  <dependency>
      <groupId>org.springframework.retry</groupId>
      <artifactId>spring-retry</artifactId>
  </dependency>
  <dependency>
      <groupId>org.aspectj</groupId>
      <artifactId>aspectjweaver</artifactId>
  </dependency>
  
  
  
  @Retryable
  public String hello(){
      long times = helloTimes.incrementAndGet();
      log.info("hello times:{}", times);
      if (times % 4 != 0){
          log.warn("发生异常，time：{}", LocalTime.now() );
          throw new HelloRetryException("发生Hello异常");
      }
      return "hello " + nameService.getName();
  }
  ```

* Guava Retry方式

  ```
  <dependency>
      <groupId>com.github.rholder</groupId>
      <artifactId>guava-retrying</artifactId>
      <version>2.0.0</version>
  </dependency>
  
  
  
  @Test
  public void guavaRetry() {
      Retryer<String> retryer = RetryerBuilder.<String>newBuilder()
          .retryIfExceptionOfType(HelloRetryException.class)
          .retryIfResult(StringUtils::isEmpty)
          .withWaitStrategy(WaitStrategies.fixedWait(3, TimeUnit.SECONDS))
          .withStopStrategy(StopStrategies.stopAfterAttempt(3))
          .build();
  
      try {
          retryer.call(() -> helloService.hello());
      } catch (Exception e){
          e.printStackTrace();
      }
  }
  ```

相比Spring，Guava Retry提供了几个核心特性。

- 可以设置任务单次执行的时间限制，如果超时则抛出异常。
- 可以设置重试监听器，用来执行额外的处理工作。
- 可以设置任务阻塞策略，即可以设置当前重试完成，下次重试开始前的这段时间做什么事情。
- 可以通过停止重试策略和等待策略结合使用来设置更加灵活的策略，比如指数等待时长并最多10次调用，随机等待时长并永不停止等等。