



- [分布式锁](#分布式锁)

### 集群

### 主从

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










