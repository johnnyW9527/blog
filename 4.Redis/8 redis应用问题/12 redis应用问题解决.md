#### 12.redis应用问题解决

##### 12.1缓存穿透

###### 12.1.1 问题描述

```
当系统中引入redis缓存后，一个请求进来后，会先从redis缓存中查询，缓存有就直接返回，缓存中没有就去db中查询，db中如果有就会将其丢到缓存中，但是有些key对应更多数据在db中并不存在，每次针对此次key的请求从缓存中取不到，请求都会压到db，从而可能压垮db。
比如用一个不存在的用户id获取用户信息，不论缓存还是数据库都没有，若黑客利用大量此类攻击可能压垮数据库
```

###### 12.1.2 解决方案

1. 对空值缓存

```
如果一个查询返回的数据为空（不管数据库是否存在），我们仍然把这个结果（null）进行缓存，给其设置一个很短的过期时间，最长不超过五分钟
```

2. 设置可访问的白名单

```
使用redis中的bitmaps类型定义一个可以访问的名单，名单id作为bitmaps的偏移量，每次访问和bitmap里面的id进行比较，如果访问的id不在bitmaps里面，则进行拦截，不允许访问
```

3. 采用布隆过滤器

```
布隆过滤器（Bloom Filter）是1970年有布隆提出的，它实际上是一个很长的二进制向量（位图）和一系列随机映射函数（哈希函数）
布隆过滤器可以用于检测一个元素是否在一个集合中，它的优点是空间效率和查询的世界都远远超过一般的算法，缺点是有一定的误识别率和删除困难。
将所有可能存在的数据哈希到一个足够大的bitmaps中，一个一定不存在的数据会被这个bitmaps拦截掉，从而避免了对底层存储系统的查询压力。
```

4. 进行实时监控

```
当发现redis的命中率开始急速降低，需要排查访问对象和访问的数据，和运维人员配合，可以设置黑名单限制对其提供服务（比如：IP黑名单）
```

##### 12.2 缓存击穿

###### 12.2.1 问题描述

> redis中某个热点key（访问量很高的key）过期，此时大量请求同时过来，发现缓存中没有命中，这些请求都打到db上了，导致db压力瞬时大增，可能会打垮db，这种情况成为缓存击穿
>
> 缓存击穿出现的现象
>
> * 数据库访问压力瞬时增大
> * redis里面没有出现大量的key过期
> * redis正常运行

###### 12.2.2 解决方案

> key可能会在某些时间点被超高并发地访问，是一种非常“热点”的数据，这个时候，要考虑一个问题：缓存被“击穿”的问题，常见的解决方案如下

1. 预先设置热门数据，适时调整过期时间

> 在redis高峰之前，把一些热门数据提前存入到redis里面，对缓存中的这些热门数据进行监控，实时调整过期时间

2. 使用锁

> 缓存中拿不到数据的时候，此时不是立即去db中查询，而是去获取分布式锁（比如redis中的setnx），拿到锁再去db中load数据；没有拿到锁的线程休眠一段时间再重试整个获取数据的方法。

##### 12.3 缓存雪崩

###### 12.3.1 问题描述

> key对应的数据存在，但是极短时间内有大量的key集中过期，此时若有大量的并发请求过来，发现缓存没有数据，大量的请求就会落到db上去加载数据，会将db击垮，导致服务奔溃。缓存雪崩与缓存击穿的区别在于：前者是大量的key集中过期，而后者是某个热点key过期。

###### 12.3.2 解决方案

> 缓存失效时的雪崩效益对底层系统的冲击非常可怕，常见的解决方案如下

1. 构建多级缓存

> nginx缓存+redis缓存+其他缓存（ehcache等）

2. 使用锁或队列

> 用加锁或者队列的方式来保证不会有大量的线程对数据库一次性进行读写，从而避免失效时大量的并发请求落到底层存储系统上，不适用高并发情况

3. 监控缓存过期，提前更新

> 监控缓存，发下缓存快过期了，提前对缓存进行更新。

4. 将缓存失效时间分散开

> 比如我们可以在原有的失效时间基础上增加一个随机值，比如1-5分钟随机，这样缓存的过期时间重复率就会降低，就很难引发集体失效的事件

##### 12.4 分布式锁

###### 12.4.1 问题描述

> 随着业务发展的需要，原单体单机部署的系统被演化成分布式集群系统后，由于分布式系统多线程、多进程且分布在不同机器上，这将使原单机部署情况下的并发控制锁策略失效，单纯的Java API并不能提供分布式锁的能力，为了解决这个问题就需要一种跨JVM的互斥机制来控制共享资源的访问，这就是分布式锁要解决的问题

###### 12.4.2 分布式锁主流的实现方案

1. 基于数据库实现分布式锁
2. 基于缓存(redis等)
3. 基于zookeeper

**每一种分布式锁解决方案都有各自的优缺点**

1. 性能：redis最高
2. 可靠性：zookeeper最高

###### 12.4.3 解决方案：redis实现分布式锁

**需要使用下面这个命令来实现分布式锁**

```
set key value NX PX 有效期(毫秒)
```

> 这条命令表示：当key不存在的时候，设置其值为value,且同时设置其有效期

```
set sku:1:info "ok" NX PX 10000
```

> 表示当sku:1:info 不存在的时候，设置值为ok,且有效期为1万毫秒

1. 上锁的过程

> 过程如下图，执行 set key value NX PX 有效期(毫秒) 命令，返回ok表示执行成功，则获取锁成功，多个客户端并发执行此命令的时候，redis可确保只有一个可以执行成功

![](C:\study\mlog\picture\7.png)

2. 为什么要设置过期时间

> 客户端获取锁后，由于系统问题，如系统宕机了，会导致锁无法释放，其他客户端就无法或锁了，所以需要给锁指定一个使用期限

3. 如果设置的有效期太短怎么办？

> 比如有效期设置了10秒，但是10秒不够业务方使用，这种情况客户端需要实现续命的功能，可以解决这个问题

4. 解决锁误删的问题

> 锁存在误删的情况：所谓误删就是自己把别人持有的锁给删掉了
>
> 比如线程A获取锁的时候，设置的有效期是10秒，但是执行业务的时候，A程序突然卡主了超过了10秒，此时这个锁就可能被其他线程拿到，比如被线程B拿到了，然后A从卡顿中恢复了，继续执行业务，业务执行完毕之后，去执行了释放锁的操作，此时A会执行del命令，此时就出现了锁的误删，导致的结果就是把B持有的锁给释放了，然后其他线程又会获取这个锁，挺严重的
>
> **如何解决呢**
>
> 获取锁的之前，生成一个全局唯一id，将这个id也丢到key对应的value中，释放锁之前，从redis中将这个id拿出来和本地的比较一下，看看是不是自己的id，如果是的再执行del释放锁的操作

5. 还是存在误删的可能（原子操作问题）

> 刚才上面说了，del之前，会先从redis中读取id，然后和本地id对比一下，如果一致，则执行删除，伪代码如下

```
step1:判断 redis.get("key").id==本地id 是否相当,如果是则执行step2
step2:del key;
```

> 此时如果执行step2的时候系统卡主了，比如卡主了10秒，然后redis才收到，这个期间锁可能又被其他线程获取了，此时又发生了误删的操作
>
> **这个问题的根本原因是：**判断和删除这2个步骤对redis来说不是原子操作导致的，怎么解决呢？需要使用Lua脚本来解决

6. 终极方案：Lua脚本来释放锁

> 将复杂的或者多步的redis操作，写为一个脚本，一次提交给redis执行，减少反复连接redis的次数，提升性能。
>
> Lua脚本类似于redis事务，有一定的原子性，不会被其他命令插队，可以完成一些redis事务的操作。
>
> 但是注意redis的LUA脚本功能，只能在redis2.6以上版本才能使用

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.core.script.DefaultRedisScript;
import org.springframework.http.MediaType;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import java.util.Arrays;
import java.util.UUID;
import java.util.concurrent.TimeUnit;
@RestController
public class LockTest {
@Autowired
private RedisTemplate<String, String> redisTemplate;
@RequestMapping(value = "/lock", produces = MediaType.TEXT_PLAIN_VALUE)
public String lock() {
    String lockKey = "k1";
    String uuid = UUID.randomUUID().toString();
    //1.获取锁,有效期10秒
        if (this.redisTemplate.opsForValue().setIfAbsent(lockKey, uuid, 10,TimeUnit.SECONDS)) {
    //2.执行业务
    // todo 业务
    //3.使用Lua脚本释放锁(可防止误删)
    String script = "if redis.call('get',KEYS[1])==ARGV[1] then return
    redis.call('del',KEYS[1]) else return 0 end";
    DefaultRedisScript<Long> redisScript = new DefaultRedisScript<>();
    redisScript.setScriptText(script);
    redisScript.setResultType(Long.class);
    Long result = redisTemplate.execute(redisScript,
    Arrays.asList(lockKey), uuid);
    System.out.println(result);
    return "获取锁成功!";
    } else {
    return "加锁失败!";
    }
  }
}
```

7. 分布式锁总结

> 为了确保分布式锁可用，我们至少需要确保分布式锁的实现同时满足以下四个条件
>
> * 互斥性，在任意时刻只能有一个客户端能够持有锁
> * 不糊发生死锁，即使有一个客户端在持有锁期间崩溃而没有释放锁，也能够保证后续其他客户端能够加锁
> * 解锁还需寄铃人，加锁和解锁必须是同一个客户端，客户端不能把别人的锁给解了
> * 加锁和解锁必须有原子性