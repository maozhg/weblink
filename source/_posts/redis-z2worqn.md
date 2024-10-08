---
title: Redis
date: '2024-09-28 00:16:55'
updated: '2024-10-07 17:12:15'
comments: true
toc: true
---



```mindmap
- redis面试题
  - 什么redis,用来做什么的
  - redis的基本数据类型
  - redis为什么这么快
  - 什么是缓存击穿，缓存穿透、缓存雪崩
  - 什么是热key问题，如何解决热key问题
  - redis过期策略和缓存淘汰策略
  - redis的常用场景
  - redis的持久化机制有哪些？优缺点
  - 怎么实现redis高可用
  - 使用redis分布式锁吗？有哪些注意点呢？
  - 使用过Redisson吗？说说他的原理
  - 什么是Redlock算法
  - Redis的跳跃表
  - Mysql与Redis如何保证双写一致性
  - 为什么redis6.0之后改多线程呢？
  - 聊聊Redis事务机制
  - Redis的Hash冲突怎么办
  - 在生成RDB期间，Redis可以同时处理写请求吗
  - Redis底层，使用什么协议
  - 布隆过滤器
```

## 1. 什么是redis，他主要用来做什么

Redis 英文 **Remote Dictionary Server**(远程字典服务)，是一个开元的使用ANSI C语言编写、支持网络、可基于内存亦可以持久化的日志型、key-value数据库，并提供多种语言的API。

因为数据是在内存中的，它的读写速度非常快。因此**广泛应用于缓存**，还可以用来做**分布式锁**。

‍

<!-- more -->

## 2. 说说Redis的基本数据结构类型

<iframe src="/widgets/widget-excalidraw/" data-src="/widgets/widget-excalidraw/" data-subtype="widget" border="0" frameborder="no" framespacing="0" allowfullscreen="true"></iframe>

有下面的五种基本类型：

* String（字符串）
* Hash（哈希）
* List（列表）
* Set（集合）
* zset（有序集合）

他还有三种特殊的数据结构类型

* Geospatial
* Hyperloglog
* Bitmap

### String类型

> String类型是Redis最基础的数据类型结构，它是二进制安全的，可以存储图片或者序列化的对象，值最大存储为512M

使用：` set key value`​,` get key`​

使用场景：

* 共享session
* 分布式锁
* 计数器
* 限流

内部编码有三种：

* int（8字节长整型）
* embstr（小于等于39字节字符串）
* raw（大于39个字节字符串）

```c
//Redis使用的SDS封装的字符串，没有使用C语言的char[]存储字符串，可以O(1)获取长度
struct sdshdr{
	unsigned int len; //标记buf的长度
	unsigned int free; //标记buf中未使用的元素个数
	char buff[]; //存放元素的坑
}
```

### Hash类型

> 在Redis中，哈希类型指的是v本身又是一个键值对（k-v）结构

使用 ` hset key field value`​, ` hget key field`​

内部编码：ziplist（压缩列表）、hashtable（哈希表）

使用场景：缓存用户信息等

**注意**：如果使用`hgetall`​哈希表元素比较多的话，可能导致Redis阻塞，可以使用`hscan`​。而如果知识获取部分field，建议使用hmget。

### List

> 列表List类型是用来存储多个有序的字符串，一个列表最多可以存储2^32-1个元素

使用：`lpush key value [value ...]`​、`lrange key start end`​

内部编码：ziplist（压缩列表）、linkedlist（链表）

应用场景：消息队列，文章列表

<iframe src="/widgets/widget-excalidraw/" data-src="/widgets/widget-excalidraw/" data-subtype="widget" border="0" frameborder="no" framespacing="0" allowfullscreen="true"></iframe>

> * lpush + lpop = Stack
> * lpush + rpop = Queue
> * lpush + ltrim = Capped Collection（有限集合）
> * lpush + brpop = Message Queue （消息队列）

### Set

> 集合类型也是用来保存多个字符串元素，但是不允许重复

使用：`sadd key element [element ...]`​、 `smembers key`​

内部编码： intset（整数集合）、hashtable（哈希表）

**注意**：smembers和lrange、hgetall都属于比较重的命令，如果元素过多存在阻塞Redis的可能性，可以使用sscan来完成。

应用场景：用户标签，生成随机数抽奖，社交需求。

### zset

> 有序集合，已排序的字符串集合，同时元素不能重复。

使用：`zadd key score member [score member ...]`​, `zrank key member`​

底层编码：ziplist（压缩列表）、skiplist（跳跃表）

应用场景：排行榜，社交需求（如用户点赞）。

### Redis的三种特殊数据类型

* Geo：Redis3.2推出的，地理位置定位，用于存储地理位置信息，并对存储的信息进行操作。
* HyperLogLog：用来做基数统计算法的数据结构，如统计网站的UV。
* Bitmaps：用一个比特位来映射某个元素的状态，在Redis中，他的底层是基于字符串烈性实现的，可以把bitmaps成作一个以比特为单位的数组。

## 3. Redis为什么这么快？

```mindmap
- Redis为什么这么快
  - 基于内存的视线
    - 对比Mysql磁盘数据库
  - 高效的数据结构
    - SDS简单动态字符串
    - 哈希
    - 跳跃表
    - 压缩列表ziplist
  - 合理的数据编码
    - String
    - List
    - Hash
    - Set
    - Zset
  - 合理的线程模型
    - 单线程模型：避免上下文切换
    - I/O多路复用
  - 虚拟内存机制
```

### 3.1 基于内存存储实现

内存读写是比磁盘快很多的，Redis基于内存存储实现的数据库，相对于数据存储在磁盘的MySQL数据库，省去磁盘的I/O的消耗。

### 3.2 高效的数据结构

我们知道，mysql索引为了提高效率，选择了B+树的数据机构。其实合理的数据结构，就能让你的应用、程序更快。

Redis的数据结构和内部编码图：

<iframe src="/widgets/widget-excalidraw/" data-src="/widgets/widget-excalidraw/" data-subtype="widget" border="0" frameborder="no" framespacing="0" allowfullscreen="true"></iframe>

#### SDS简单动态字符串

<iframe src="/widgets/widget-excalidraw/" data-src="/widgets/widget-excalidraw/" data-subtype="widget" border="0" frameborder="no" framespacing="0" allowfullscreen="true"></iframe>

> * 字符创长度处理：Redis获取字符串长度，时间复杂度为O(1)，而C语言需要从头开始遍历，复杂度为O(n)；
> * 空间预分配：字符串修改越频繁的话，内存分配越频繁，就会消耗性能，而SDS修改和空间扩充，会额外分配未使用的空间，减少性能损耗。
> * 惰性空间释放：SDS缩短时，不是回收多余的内存空间，而是free记录下多余的空间，后续有变更，直接使用free中记录的空间，减少分配。
> * 二进制安全：Redis可以存储一些二进制数据，在C语言中字符串遇到'\\0\'会结束，而SDS结构中标志字符串结束的是len属性。

#### 字典

Redis作为K-V型内存数据库，所有的键值就是用字典来存储。字典就是哈希表，比如HashMap，通过key就可以直接获取到对应的value。而哈希表的特性，在O(1)时间复杂度就可以获得对应的值。

#### 跳跃表

<iframe src="/widgets/widget-excalidraw/" data-src="/widgets/widget-excalidraw/" data-subtype="widget" border="0" frameborder="no" framespacing="0" allowfullscreen="true"></iframe>

> * 跳跃表式Redis特有的数据结构，就是在链表的基础上，增加多级索引提升查找效率。
> * 跳跃表支持平均O(logN)，最坏O(N)复杂的节点查找，还可以通过顺序性操作批处理节点。

### 3.3 合理的数据编码

Redis支持多种数据类型，每种基本类型，可能对多种数据结构，什么时候使用什么样的数据结构，使用什么编码，是redis设计者总结优化的结果。

> * String：如果存储数字的话，用int类型的编码；如果存储非数字，小于等于39字节的字符串，是embstr；大于39个字节，则是raw编码。
> * List：如果列表的元素个数小于512个，列表每个元素的值都小于64字节（默认），使用ziplist编码，否则使用linkedlist编码。
> * Hash：哈希类型元素个数小于512个，所有值小于64字节的话，使用ziplist编码，否则使用hashtable编码。
> * Set：如果集合中的元素都是整数且元素个数小于512个，使用intset编码，否则使用hashtable编码。
> * Zset：当有序集合的元素个数小于128个，每个元素的值小于64自己时，使用ziplist编码，否则使用skiplist（跳跃表）编码。

### 3.4 合理的线程模型

#### I/O多路复用

什么是IO多路复用？

> I/O：网络I/O
>
> 多路：多个网络连接
>
> 复用：公用同一个线程
>
> IO多路复用是一种同步IO模型，它实现了一个线程可以监视多个文件句柄；

#### 单线程模型

* Redis是单线程模型的，单线程避免了CPU不必要的上下文切换和竞争锁消耗。也正是因为单线程，如果某个命令执行过长（hgetall），会造成阻塞。
* Redis 6.0 引入了多线程提速，他执行命令操作内存的仍然是单线程。

### 3.5 虚拟内存机制

Redis直接自己构建了VM机制，不会像一般的系统会调用系统函数处理，会浪费一定的时间去移动和请求。

#### 什么是虚拟内存机制？

虚拟内存机制就是暂时把不经常访问的数据（冷数据）从内存交换到磁盘中，从而腾出宝贵的内存空间用于其他需要访问的数据（热数据）。通过VM功能可以实现冷热数据分离，使热数据仍在内存中、冷数据保存到磁盘。这样就可以避免因为内存不足而造成访问速度下降的问题。

## 4. 什么是缓存击穿、缓存穿透、缓存雪崩？

#### 4.1 缓存穿透问题

常见的缓存请求方式：请求来了，查缓存，有则返回，没有则查询数据库，然后把数据库的数据保存到缓存，再返回。

**缓存穿透：** 指查询一个一定不存在的数据，由于缓存是不命中时需要从数据库查询，查不到数据则不写入缓存，这将导致这个不存在的数据每次请求都要到数据库去查询，进而给数据库带来压力。

一般都是这几种情况产生的：

* 业务不合理的设计，比如大多数用户都没哟开守护，但是你的每个请求都去缓存，查询某个userid查询有没有守护。
* 业务/运维/开发失误的操作，比如缓存和数据库都被误删了。
* 黑客非法请求攻击，比如黑客故意捏造大量的非法请求，以读取不存在的业务数据。

如何避免缓存穿透？

1. 如果是非法请求，我们在API入口，对参数进行校验，过滤非法值。
2. 如果查询数据库为空，我们可以给缓存设置个空值，或者默认值。但是如果有写请求进来的话，需要更新缓存，以保证缓存的一致性，同时，最后给缓存设置适当的过期时间。（业务上比较常用，简单有效）
3. 使用布隆过滤器快速判断数据是否存在。即一个查询请求过来时，先通过布隆过滤器判断值是否存在，存在才继续往下查。

> 布隆过滤器：它由初始值为0的位图数据和N个哈希函数组成。一个对一个key进行N个Hash算法获取N个值，在比特数据组中将这N个值散列后设定为1，然后查的时候如果特定的这几个位置都是1，那么布隆过滤器判断该key存在。

#### 4.2 缓存雪崩

> 缓存雪崩是指缓存中的大批量的数据同时过期，造成数据访问数据的时候都直接访问的数据库，造成机器压力过大甚至宕机问题。

* 缓存雪崩一般是大量数据同时过期造成的，所以可以把数据的过期时间设置得离散一点，采用一个较大的固定值+一个随机值。如：5小时+0到1800毫秒的随机值。
* Redis宕机也可能造成缓存雪崩，这需要Redis集群。

#### 4.3 缓存击穿

> 缓存击穿是指：指的是热点key在某个时间点过期，这时候有大量的请求访问这个可以，造成所有的请求都去访问数据库。

和缓存雪崩有点像，但是缓存雪崩针对的是多个key，击穿针对的是单个key，还有点区别就是大量的请求访问到数据库层面不一定造成宕机。

解决方案有两种：

1. 互斥锁方案：就是请求来了之后都是去获取缓存数据，同时加载数据的操作采用互斥锁，这样当有请求在加载数据库时其他的请求只是请求缓存，只要加载成功数据库，都能访问到数据。
2. “永不过期”方案：指请求都是请求缓存，但是另外一个线程去管理这个缓存数据，设置过期的时间。

## 5. 什么是热点key问题，如何解决热点key问题？

> 在Redis中，我们把访问频率高的key，称作热点key。

如果某一热点key的请求到服务器时，访问量过大，超过服务器的瓶颈，造成机器宕机，影响服务的主要功能。

热点问题的产生原因：

* 用户消费数据大于生产的数据，如秒杀，热点新闻等读多写少的场景。
* 访问的数据访问到同一台redis机器，Hash分配到同一台机器上，超过Redis单台机器的瓶颈，产生热点key问题。

日常开发怎么识别到热点key？

* 凭借以前的经验
* 客户端统计上报
* 代理层统计上报

解决方案：

* Redis集群扩容，增加分片副本，均衡读流量；
* 将热点key分配到不同的机器中；
* 使用二级缓存，使用JVM本地缓存，减少读取Redis流量。

## 6. redis过期策略和缓存淘汰策略

​![image](https://raw.githubusercontent.com/maozhg/notebook/main/pic/20241008202331.png)​

### 6.1 Redis过期策略

我们在 `set key`​是可以设置一个过期时间 `expire key 60`​,这样key在60s后就会过期，那么过期后redis是如何处理的，可以看看redis的过期策略：

#### 定时过期

给每个设置过期时间的key都添加一个定时器，定时器到期会立即清除缓存。该策略可以立即清理缓存，对内存很友好，但是对CPU不友好，会创建大量的定时器，造成机器的吞吐量和响应时间下降。

#### 惰性过期

当访问一个key时判断key是否过期，当过期时清除缓存。该策略对CPU非常友好，但是对内存不友好，甚至在一些过期的key没有被访问，造成永久的内存占用。

#### 定期过期

> 每隔一段时间，会扫描expires字典中一定数量的key，并且清除过期的key。该策略是一个折中的方案，通过调整定时扫描的时间间隔，使cpu和内存达到最优的平衡。
>
> expires字典保存的是设置了过期时间的key和过期时间数据。key保存的是指向key空间的某个key值，value保存了该键的UNIX时间戳。

Redis中同时使用了惰性过期和定期过期策略：

* 假设Redis存放了30万个key，并且都设置了过期时间，如果每隔100ms都去检查这30万个key，会造成cpu使用过高的问题。
* 因此Redis采用定期过期策略，每隔100ms抽取一定数量的key，检查是否过期。
* 但是如果定期策略没有清除掉过期的key，但是访问时访问到了，这时采用的是惰性过期策略，会先判断一下是否过期，如果过期了会进行清除。

但是通过上面的过期策略还是会有很多的内存没有被清除掉，还有一些没有过期时间的数据，会造成缓存不断地膨胀，是否会爆掉，这里就涉及到Redis的淘汰策略。

### 6.2 Redis淘汰策略

当写入数据时内存不够用了会使用淘汰策略，一共八种：

* volatile-lru：从设置了过期时间的key中使用LRU（Least  Recently Used最近最少使用）算法进行淘汰；
* allkeys-lru：从所有的key中使用LRU；
* volatile-lfu：Redis 4.0新增，在设置了过期时间的key中使用LFU（Least Frequently Used最少使用频率）算法进行淘汰；
* allkeys-lfu：Redis 4.0新增，在所有的key中使用LFU；
* volatile-random：从设置了过期时间的key中随机选择一个淘汰；
* allkeys-random：从所有的key中随机的选择一个淘汰；
* volatile-ttl：从设置了过期时间的key中选择一个最先过期的进行淘汰；
* noeviction：默认策略，不进行淘汰，直接报错。

## 7. 说说Redis的常用场景

​![image](https://raw.githubusercontent.com/maozhg/notebook/main/pic/20241008202842.png)​

### 7.1 缓存

提到Redis就不得不说缓存，国内外的大型网站上都使用缓存，缓存不仅能提高网站的访问速度还能减少数据库的压力。并且Redis相对于mencached，不仅提供了更多的数据结构，还有RDB和AOF等持久化操作。

### 7.2 排行榜

互联网应用有各种各样的排行榜，如销量，点赞量，礼物量排行榜等。Redis提供的Zset结构能够很好的处理这样的场景。

用户每天上传视频点赞排行榜的设计：

1. 用户Jay上传了一个视频并且获得了6个点赞   `zadd  user:ranking:2024-10-8  Jay 6`​
2. 过了一段时间又获取了一个赞  `zincrby  user:ranking:2024-10-8 Jay 1`​
3. 如果某个用户John作弊，需要删除该用户，则：`zrem user:ranking:2024-10-8 John`​
4. 展示获赞数最多的三个用户  `zrevrangebyrank user:ranking:2024-10-8 0 2`​

### 7.3 计数器应用

各大网站，电商APP应用上通常需要一些计数器，比如视频播放数，商品浏览数，这些计数一般都要求是实时的，每次点击的时候都需要加一操作，对于传统数据库来说是一个很大的挑战，对于Redis来说天然具有优势。

### 7.4 共享session

对于分布式的web系统，如果用户登录的session保存在各自的服务器，那么用户每次刷新的时候可能就需要重新登录，这样是有问题的。那么将用户的登录信息保存在Redis中，这样每次访问的时候直接从Redis获取用户的登录信息，就可以解决这个问题。

### 7.5 分布式锁

基本上在所有的互联网应用中都使用了分布式部署，这样就会涉及到资源并发访问的技术问题，如秒杀，下单减库存等问题。

* 使用本地的synchronize和reentrantlock肯定不行；
* 如果数据的访问量不大，则使用数据库的悲观锁和乐观锁也可以；
* 可以使用Redis的setnx来实现分布式锁。

### 7.6 社交网络

社交网络中的 赞/踩，推送，下拉刷新等数据在传统数据库中不太好保存，但是在Redis中就很容易实现，也非常容易实现。

### 7.7 消息队列

消息队列是用来将业务解耦，流量削峰，异步处理不是实时的业务等场景。Redis的发布订阅以及阻塞队列结构可以实现简单的消息队列，但是不能和专门的消息中间件相比。

### 7.8 位运算

用于数据上亿的情况下，统计每个数据每天的使用情况，连续使用情况，比如用户的连续登录情况，每天是否登录等等。参考：[redis中setbit(位操作)的实际应用-腾讯云开发者社区-腾讯云 (tencent.com)](https://cloud.tencent.com/developer/article/1757590)

## 9. Redis的持久化机制有哪些，各有什么优缺点？

Redis是基于内存的非关系型数据库K-V数据库，基于内存的，那么如果服务器挂了，那么内存中的数据就会丢失，为了避免丢失，Redis提供了持久化的功能，把数据写入到磁盘中。

‍
