---
layout: post
title: Redis深度历险
date: 2021-12-08
tags: 计算机基础
---
<!-- ### 电子书链接
[图解HTTP](/downloads/Redis深度历险.pdf) <br/> -->

### 一、Redis基础数据结构
#### 1. String
- 键值对：` SET key value [EX seconds] [PX milliseconds] [NX|XX]`、`GET key`
- 批量键值对：`MSET key1 value1 key2 value2 key3 value3`、`MGET key1 key2 key3`
- 扩展：`setNX`、`setEX`、`expire` 都可以通过五个参数的`set`命令实现（redis2.8扩展参数）
- 计数：`incr key`计数加1、`incrby key value`计数加n；计数不能超过Long.MAX
```
> set codehole 9223372036854775807 -- Long.Max
OK
> incr codehole
(error) ERR increment or decrement would overflow
```
- String命令：[https://www.runoob.com/redis/redis-strings.html](https://www.runoob.com/redis/redis-strings.html)

#### 2. List
- 相当于java的LinkedList，是链表不是数组，底层结构：linkList、zipList、quickList
- 插入删除是O(1)，查询是O(n)
- 删除最后一个元素，数据结构自动删除，内存回收
- 可作队列（先进先出），可作栈（先进后出），2.4版本后PUSH接受多个值`LPUSH key value1 [value2]`、`LPOP key `、`RPUSH key value1 [value2]`、`RPOP key `
- 2.6版本后，弹出列表元素命令允许阻塞列表直到timeout或者发现可弹元素 `BLPOP key1 [key2 ] timeout `,`BLPOP key1 [key2 ] timeout `
- list命令：[https://www.runoob.com/redis/redis-lists.html](https://www.runoob.com/redis/redis-lists.html)

#### 3. HASH(字典)
- 相当于java的HashMap，内部实现结构也与HashMap相同，值只能是String，是无序字典。
- 当 hash 移除了最后一个元素之后，该数据结构自动被删除，内存被回收。
- 键值对：`HSET key field value`、`HGET key field`、`HDEL key field1 [field2]`、`HGETALL key`
- 批量键值对：`HMSET key field1 value1 [field2 value2 ]`、`HMGET key field1 [field2]`
- 与String相比，hash更适合储存结构性的数据，适合对部分字段进行修改或获取的场景（String会将对象序列化，只能整存整取）
- set命令：[https://www.runoob.com/redis/redis-hashes.html](https://www.runoob.com/redis/redis-hashes.html)

#### 4. Set（无序去重集合）
- 相当于java的HashSet，内部键值对无序唯一，有**去重功能**。内部实现相当于一个所有value都是null的字典（key、filed、value）。
- `SADD key member1 [member2]`、`SREM key member1 [member2]`、`SPOP key`、`SMEMBERS key`、`	SISMEMBER key member`
- 可以计算并集、交集、差异、随机数
- set命令：[https://www.runoob.com/redis/redis-sets.html](https://www.runoob.com/redis/redis-sets.html)

#### 5. ZSet(有序去重集合+有分值SCORE)
- 保证内部唯一性，有score可以用于排序，底层实现结构是跳表。
- 可按score正序或逆序列出，可按分数区间正序或逆序遍历、可获取指定对象的score
- 添加`ZADD key score1 member1 [score2 member2]`
- 计数`ZCOUNT key min max`、`ZCARD key`
- 查询`ZRANGE key start stop [WITHSCORES]`、`ZREVRANGE key start stop [WITHSCORES]`、`ZRANGEBYSCORE key min max [WITHSCORES] [LIMIT]`、`ZRANK key member`
- 删除`ZREM key member [member ...]`、`ZREMRANGEBYLEX key min max`、`ZREMRANGEBYRANK key start stop`、`ZREMRANGEBYSCORE key min max`
- zSet命令：[https://www.runoob.com/redis/redis-sorted-sets.html](https://www.runoob.com/redis/redis-sorted-sets.html)

#### 补充
- 容器性数据结构（List、Hash、Set、ZSet）共有特性：**create if not exists** 并且 **drop if no elements**

### 二、应用

#### 1.分布式锁
- 考虑setNX和expire的原子性（新set五个参数，spring-data-redis需要2+以上版本，我用过的版本是2.1.2）
- 可重入性（使用ThreadLocal变量计数，客户端代码复杂，不推荐拓展可重入锁）。

#### 2.延时队列
- 缺点：没有ack保证，即可靠性不完全可靠。
- 队列空了怎么办？答：队列空了，客户端反复pop，陷入死循环，浪费客户端cpu，拉高redisQPS。可以通过BLPOP、BRPOP解决(客户端sleep也行，但是会造成延迟问题)。
- 空闲连接自动断开？答：BLPOP/BRPOP会让Redis长时间阻塞，于是客户端会认为是闲置连接并主动断开，BLPOP/BRPOP会抛出异常。所以要做好错误重试。
- 获取锁冲突？答：1.直接抛出异常，通知用户稍后处理(人工点击重试等于错开了时间)。2.sleep后重试（危险：会影响后续任务，如果死锁会导致后续消息全部堵死）。3.将请求扔到另一个队列延后处理（适合异步）。
- 如何实现？答：用redis的ZSet。消息作为value，到期时间（延迟到几点几分几秒）作为score。然后用一个或多个线程轮询ZSet获取到期任务（多线程消费防止单线程挂掉，但也要考虑并发问题）。
- 为什么不可靠？答：1.redis未持久化，redis宕机后消息消失（redis有不定期写入磁盘的半持久化和随时写入磁盘的全持久化，前者不能保障可靠性）。2.redis没有ack机制，客户端拿到消息后宕机，消息消失。

#### 3.位图
- 位数组的扩展，用01记录多位boolean值。
- 使用命令 零存`SETBIT key offset value`、零取`GETBIT key offset`、整存`SET key value`、整取`GET key`(offset把最高位记作0，value是8位字节)
- 魔术指令 bitfield 详见书

#### 4.HyperLogLog（估数）
- `pfadd` 书中实验，数据规模在10w的时候，误差在277个，即0.277%。在标准误差0.81%的前提下，能够统计2^{64}2​64​​个数据。
- 这个数据结构可以用在数据量大、需要去重、不必十分精确的数据统计中，如网站的UV（解决精度不高的统计需求）。
- `pfmerge`可以合并2个HyperLogLog并去重。
- 缺点：pf内存占用大，在数据大的情况下占用空间是12k字节（13684 * 6bit / 8 = 12k）；在数据小的情况下会优化成稀疏存储。所以比如有千万用户，对每个用户建一个pf不合适。
- 为什么一个HyperLogLog占12k空间？在 Redis 的 HyperLogLog 实现中用到的是 16384 个桶，也就是 2^14，每个桶的 maxbits 需要 6 个 bits 来存储，最大可以表示 maxbits=63，于是总共占用内存就是 2^14 * 6 / 8 = 12k 字节。

#### 5.布隆过滤器
- 用途：**(防止缓存穿透)**并发量高的时候，先通过布隆过滤器过滤掉不存在的（过滤掉90%），再查数据库判断是否存在，减轻数据库压力。
- 原理：布隆过滤器是一个长二进制向量（数组），输入值进行多次hash函数运算，add操作把所有运算结果对应的位数标1。exist操作发现所有hash计算位置都是1，则表示命中（存在）。
- 误差：布隆过滤器记录所有数据的key，因为通过随机散列映射，可能会有2个key的散列映射计算结果相同，于是被误判存在。多个key的散列计算结果集合包含了这个key的散列结算结果，也会导致误判。
- 命令：需要redis4.0版本，添加`BF.ADD`、存在`BF.EXISTS`（1表示存在，0表示不存在）、批量添加`BF.MADD`、批量存在`BF.MEXISTS`
- 参数：自定义创建命令`bf.reserve key error_rate initial_size` 错误率越低，需要的空间越大(default 0.01)。initial_size 参数表示预计放入的元素数量(default 100)，当实际数量超出这个数值时，误判率会上升。JAVA的Redisson有封装。
- 内存实现：Guava提供了内存实现的布隆过滤器。

#### 6.限流：滑动窗口算法（原书中的简单限流）
- 原理：用zset维护一个滑动时间窗口。key标记用户+行为，value是不重复随机数 如时间戳，score是行为发生时间。用户发生行为时，清除zset中过期数据，统计zset中剩余行为数是否超过限额，最后重新设定zset的key过期时间。
- 缺点：要记录所有用户行为，不能满足“限定60s内操作不能超过100万次”之类数据量大的需求。

#### 7.限流：令牌桶算法（原书中的漏斗限流）
- 原书说是漏桶限流算法，但redis-cell不能做到匀速流出，只是能匀速生产令牌。
> 漏桶算法与令牌桶算法的区别：
> - 漏桶算法是调用方将请求(水滴)放入桶中，主机匀速向网络上发送数据包，控制平滑的网络流量。即使遇到突发流量，也会被存储起来，匀速消费。
> - 令牌桶算法是服务方匀速向网络上提供令牌，获得令牌立即能获得服务，能控制平均网络流量，但允许出现突发流量将令牌一抢而空造成峰值的状态。

- 强行限制速率，平滑网络突发流量。
- 需要redis4.0，安装redis-cell拓展模块。
- 命令`CL.THROTTLE <key> <max_burst> <count per period> <period> [<quantity>]`
```
响应：
127.0.0.1:6379> CL.THROTTLE user123 15 30 60
1) (integer) 0  //0：action is allowed，1：action is limited/blocked
2) (integer) 16 //total limit of key(max_burst + 1) 令牌容量
3) (integer) 15 //remaining limit of key 剩余令牌
4) (integer) -1 //number of seconds until the user should retry, and always -1 if the action was allowed.
5) (integer) 2 //The number of seconds until the limit will reset to its maximum capacity. 多少秒后令牌会满。
```
- [https://github.com/brandur/redis-cell](https://github.com/brandur/redis-cell)


#### 8.限流：漏桶算法
- 在redis-cell基础上与延迟队列配合。当成功流入桶中时，将请求加入延迟队列，利用延迟队列来实现匀速流出。

#### 9.GeoHash
- 地理位置信息是经纬度的二维数组，GeoHash算法把地图分割成一个个小方块，将二维数组简化成一维数组。
- 详细原理：[https://zhuanlan.zhihu.com/p/35940647](https://zhuanlan.zhihu.com/p/35940647)
- API介绍：[https://www.runoob.com/redis/redis-geo.html](https://www.runoob.com/redis/redis-geo.html)

#### 10.Scan（查找符合正则的key）
- 原生keys命令缺点：1. 没有limit offset 2.复杂度O(n)且会阻塞进程
- redis2.8支持Scan命令：`SCAN cursor [MATCH pattern] [COUNT count]`
> 1. 复杂度虽然也是 O(n)，但是它是通过游标分步进行的，不会阻塞线程;
> 2. 提供 limit 参数，可以控制每次返回结果的最大条数，limit 只是一个 hint，返回的结果可多可少;
> 3. 同 keys 一样，它也提供模式匹配功能;
> 4. 服务器不需要为游标保存状态，游标的唯一状态就是 scan 返回给客户端的游标整数;
> 5. 返回的结果可能会有重复，需要客户端去重复，这点非常重要;
> 6. 遍历的过程中如果有数据修改，改动后的数据能不能遍历到是不确定的;
> 7. 单次返回的结果是空的并不意味着遍历结束，而要看返回的游标值是否为零;

- 扩展阅读：[美团针对Redis Rehash机制的探索和实践](https://mp.weixin.qq.com/s/ufoLJiXE0wU4Bc7ZbE9cDQ)