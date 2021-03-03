
<!-- TOC -->

- [数据结构](#数据结构)
- [单线程快速的原因](#单线程快速的原因)
- [持久化](#持久化)
- [缓存击穿缓存雪崩缓存穿透](#缓存击穿缓存雪崩缓存穿透)
- [主从哨兵集群](#主从哨兵集群)
    - [哨兵](#哨兵)
    - [复制](#复制)
- [分布式锁](#分布式锁)
    - [redisson分布式锁的底层原理](#redisson分布式锁的底层原理)
    - [分布式锁的缺点](#分布式锁的缺点)
- [other](#other)
    - [双写一致性](#双写一致性)

<!-- /TOC -->


### 数据结构
- string：预分配冗余空间
    - 缓存；计数；共享session；(限速)
- list：ziplist -> quicklist
    - 消息队列
- hash: 数组+链表 
    - 渐进式rehash
    - 关系型数据库表保存用户信息
- set: 
    - 标签
- zset: skiplist 
    - [跳跃表](https://mp.weixin.qq.com/s/NOsXdrMrWwq4NTm180a6vw)
    - 排行榜

### 单线程快速的原因

- 纯内存访问
- 非阻塞IO：IO多路复用
- 单线程避免了线程切换和静态产生的消耗

### 持久化

- RDB(redis database)
    - 默认
- AOF(append-onlyfile)：以日志的形式记录每个写操作



### 缓存击穿缓存雪崩缓存穿透

降低数据库压力 

- 缓存穿透：大量恶意访问数据库中没有的数据
    - key做校验
    - Synchronized双重检测机制(大量访问的提前写入缓存)
    - 没有数据的也写入缓存，值为null。恶意攻击情况下，空值太多，会占用大量内存，要定期清理
    - 布隆过滤器
- 缓存击穿：key失效时，大量请求过来(并发用户)
    - 加锁
    - 热点数据设置缓存不失效，每次更新值的同时去更新缓存
- 缓存雪崩：大量缓存同时失效
    - 设置key分时段失效
    		

### 主从哨兵集群
[refer](https://www.cnblogs.com/liuqingzheng/p/11080495.html)

- 主从：备份 读写分离
    - 主从切换：选举
- 哨兵：监控 自动转移
    - 多个哨兵监控节点 哨兵直接通信交换监控状况
- 集群：解决单机redis容量有限
    - 哈希槽(hash slot)方式分配数据

#### 哨兵

监控、自动故障转移

- 主观下线：一个哨兵节点判定主节点down掉是主观下线。
- 客观下线：只有半数哨兵节点都主观判定主节点down掉，此时多个哨兵节点交换主观判定结果，才会判定主节点客观下线。
- 原理：基本上哪个哨兵节点最先判断出这个主节点客观下线，就会在各个哨兵节点中发起投票机制Raft算法（选举算法），最终被投为领导者的哨兵节点完成主从自动化切换的过程

#### 复制

同步+命令传播：
- 从数据库向主数据库发送sync(数据同步)命令。
- 主数据库接收同步命令后，会保存快照，创建一个RDB文件。
- 当主数据库执行完保持快照后，会向从数据库发送RDB文件，而从数据库会接收并载入该文件。
- 主数据库将缓冲区的所有写命令发给从服务器执行。
- 以上处理完之后，之后主数据库每执行一个写命令，都会将被执行的写命令发送给从数据库

主从复制在线重连：PSYNC
- 主服务器复制积压缓冲区

### 分布式锁

[refer](https://blog.csdn.net/lzhcoder/article/details/88387751)

Redis锁的过期时间小于业务的执行时间该如何续期？

**基于redis的Redisson分布式可重入锁** `RLock`JAVA对象实现了java.util.concurrent.locks.Lock接口。同时还提供了异步(Async)、反射式(Reactive)和RxJava2标准的接口

```
RLock lock = redisson.getLock("anyLock");
lock.lock();
lock.unlock();
```
Redisson内部提供了一个监控锁的，它的作用是在Redisson实例被关闭前，不断延长锁的有效期。默认情况下，检查锁的超时时间是30s，也可以通过修改`Config.lockWatchdogTimeout`来另行指定
另外Redisson还通过加锁的方法提供了`leaseTime`的参数来指定加锁的时间，超过这个时间后 锁便自动解开了

只要客户端一旦加锁成功，就会启动一个watch dog看门狗，他是一个后台线程，会每隔10秒检查一下，如果客户端还持有锁key，那么就会不断的延长锁key的生存时间。
默认情况下,加锁的时间是30秒,.如果加锁的业务没有执行完,就会进行一次续期,把锁重置成30秒.那这个时候可能又有同学问了,那业务的机器万一宕机了呢?宕机了定时任务跑不了,就续不了期,那自然30秒之后锁就解开了呗

#### redisson分布式锁的底层原理

![Redisson分布式锁的底层原理](https://gitee.com/lylw/image/raw/master/redis/redis_lock.jpg)

```lua
"if (redis.call('exists', KEYS[1]) == 0) then " +
    "redis.call('hset', KEYS[1], ARGV[2], 1); " +
    "redis.call('pexpire', KEYS[1], ARGV[1]); " +
    "return nil; " +
"end; " +
"if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +
    "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +
    "redis.call('pexpire', KEYS[1], ARGV[1]); " +
    "return nil; " +
"end; " +
"return redis.call('pttl', KEYS[1]);"

// KEYS[1]代表的是加锁的那个key ARGV[1]代表的就是锁key的默认生存时间，默认30秒    ARGV[2]代表的是加锁的客户端的ID 类似于`8743c9c0-0795-4907-87fd-6c719a6b4586:1`
```
2）锁互斥机制

<details>
  <summary>示例讲解</summary>
  <pre><code> 
     "if (redis.call('exists', KEYS[1]) == 0) then " +   // 判断如果key不存在，就进行加锁
         "redis.call('hset', KEYS[1], ARGV[2], 1); " +   // 设置一个hash数据结构
         "redis.call('pexpire', KEYS[1], ARGV[1]); " +   // 设置key的生存时间
         "return nil; " +
     "end; " +
     "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +    // 当客户端2尝试加锁，判断myLock锁key的hash数据结构中，是否包含客户端2的ID，如果不包含，则
         "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +           // 
         "redis.call('pexpire', KEYS[1], ARGV[1]); " +              // 
         "return nil; " +
     "end; " +
     "return redis.call('pttl', KEYS[1]);"      // 
     
     # KEYS[1]代表的是加锁的那个key ARGV[1]代表的就是锁key的默认生存时间，默认30秒    ARGV[2]代表的是加锁的客户端的ID 类似于`8743c9c0-0795-4907-87fd-6c719a6b4586:1`
     
     > 加锁机制
        如果该客户端面对的是一个redis cluster集群，他首先会根据hash节点选择一台机器。仅仅只是选择一台机器
        紧接着将复杂的业务逻辑封装在lua脚本中发送给redis，保证这段复杂业务逻辑执行的原子性
     > 锁互斥机制
        那么在这个时候，如果客户端2来尝试加锁，执行了同样的一段lua脚本，会咋样呢？
        很简单，第一个if判断会执行“exists myLock”，发现myLock这个锁key已经存在了。
        接着第二个if判断，判断一下，myLock锁key的hash数据结构中，是否包含客户端2的ID，但是明显不是的，因为那里包含的是客户端1的ID。
        所以，客户端2会获取到pttl myLock返回的一个数字，这个数字代表了myLock这个锁key的剩余生存时间。比如还剩15000毫秒的生存时间。
        此时客户端2会进入一个while循环，不停的尝试加锁
     > watch dog自动延期机制
        客户端1加锁的锁key默认生存时间才30秒，如果超过了30秒，客户端1还想一直持有这把锁，怎么办呢？
        简单！只要客户端1一旦加锁成功，就会启动一个watch dog看门狗，他是一个后台线程，会每隔10秒检查一下，如果客户端1还持有锁key，那么就会不断的延长锁key的生存时间
     > 可重入加锁机制
        第二个if判断会成立，因为myLock的hash数据结构中包含的那个ID，就是客户端1的那个ID，也就是“8743c9c0-0795-4907-87fd-6c719a6b4586:1”
        此时就会执行可重入加锁的逻辑，他会用： incrby myLock   8743c9c0-0795-4907-87fd-6c71a6b4586:1 1
        通过这个命令，对客户端1的加锁次数，累加1
     > 释放锁机制
        每次都对myLock数据结构中的那个加锁次数减1。
        如果发现加锁次数是0了，说明这个客户端已经不再持有锁了，此时就会用：“del myLock”命令，从redis里删除这个key。
        然后呢，另外的客户端2就可以尝试完成加锁了。
        这就是所谓的分布式锁的开源Redisson框架的实现机制
  </code></pre>
</details>

#### 分布式锁的缺点

对某个redis master实例，写入了myLock这种锁key的value，会异步复制给对应的master slave实例。但这个过程中一旦发生redis master宕机，主备切换，redis slave变成了redis master 
另一个客户端2来尝试加锁，在新的redis master上完成了加锁，而客户端1也以为自己成功加了锁。此时就会导致多个客户端对一个分布式锁完成了加锁。
这时系统在业务语义上一定会出现问题，导致各种脏数据的产生
所以这个就是redis cluster，或者是redis master-slave架构的主从异步复制导致的redis分布式锁的最大缺陷：**在redis master实例宕机的时候，可能导致多个客户端同时完成加锁**。


### other

#### 双写一致性
[refer](https://www.cnblogs.com/liuqingzheng/p/11080680.html)
- 数据库先写入，更新redis：
    - 多线程场景：两个线程同时进行数据库写入。A写入数据库，B写入数据库，B更新redis，A更新redis
    - 业务场景：写多读少的场景下，每次更新都要写入redis，没必要；如果写入redis不是单纯查询，而是复杂操作，消耗大
- 删除redis，再更新数据库：
    - A删除redis之后，有线程查，然后写入redis，这时A更新数据库，造成不一致
    - 解决方案：双删(第二次异步删除)
- 先更新数据库，再删除redis：因为数据库读比写快。
    - 非要确保一致性，可以双删。第二次删除失败的情况要解决的话，用队列重试







