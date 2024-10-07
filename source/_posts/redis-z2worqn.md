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
