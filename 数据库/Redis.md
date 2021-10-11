# Redis

##  一、 概述

Redis 是速度非常快的非关系型（NoSQL）内存键值数据库，可以存储键和五种不同类型的值之间的映射。

键的类型只能为字符串，值支持五种数据类型：字符串、列表、集合、散列表、有序集合。

Redis 支持很多特性，例如将内存中的数据持久化到硬盘中，使用复制来扩展读性能，使用分片来扩展写性能。

速度快，完全基于内存，使用 C 语言实现，网络层使用 epoll 解决高并发问题，单线程模型避免了不必要的上下文切换及竞争条件；

**为什么使用redis?**

+ Redis可以干什么事儿?

1. **排行榜**，如果使用传统的关系型数据库来做，非常麻烦，而利用 Redis 的 SortSet 数据结构能够非常方便搞定；
2. **计算器/限速器**，利用 Redis 中原子性的自增操作，我们可以统计类似用户点赞数、用户访问数等，这类操作如果用 MySQL，频繁的读写会带来相当大的压力；限速器比较典型的使用场景是限制某个用户访问某个 API 的频率，常用的有抢购时，防止用户疯狂点击带来不必要的压力；
3. **好友关系**，利用集合的一些命令，比如求交集、并集、差集等，可以方便搞定一些共同好友、共同爱好之类的功能；
4. **简单消息队列**，除了 Redis 自身的发布/订阅模式，我们也可以利用 List 来实现一个队列机制，比如到货通知、邮件发送之类的需求，不需要高可靠，但是会带来非常大的 DB 压力，完全可以用 List 来完成异步解耦；
5. **Session 共享**，以 PHP 为例，默认 Session 是保存在服务器的文件中，如果是集群服务，同一个用户过来可能落在不同机器上，这就会导致用户频繁登陆；采用 Redis 保存 Session 后，无论用户落在那台机器上都能够获取到对应的 Session 信息。

+ Redis 不能干什么事儿?

1. 比如，用 Redis 去保存用户的基本信息，虽然它能够支持持久化，但是它的持久化方案并不能保证数据绝对的落地，并且还可能带来 Redis 性能下降，因为持久化太过频繁会增大 Redis 服务的压力。
2. 简单总结就是数据量太大、数据访问频率非常低的业务都不适合使用 Redis，数据太大会增加成本，访问频率太低，保存在内存中纯属浪费资源。

## 二、 数据类型

**基本数据结构**

+ String: 字符串
+ Hash: 散列
+ List: 列表
+ Set: 集合
+ Sorted Set: 有序集合
+ 位图 Bitmap

**String**

1. 介绍：String 是一组字节。在 Redis 数据库中，字符串是二进制安全的。这意味着它们具有已知长度，并且不受任何特殊终止字符的影响。可以在一个字符串中存储最多 512 MB的内容。
2. 常用命令： set,get,strlen,exists,decr,incr,setex 等。
3. 应用场景： 一般常用在需要计数的场景，比如用户的访问次数、热点文章的点赞转发数量等。

使用 SET 命令在 name 键中存储字符串 "xing"，然后使用 GET 命令查询 name。
```
127.0.0.1:6379> set name "xing"
OK
127.0.0.1:6379> get name
"xing"
```

SET 和 GET 是 Redis 命令，name 是 Redis 中使用的 key，xing 是存储在 Redis 中的字符串值。
批量设置 :
```
127.0.0.1:6379> mset key1 value1 key2 value2 # 批量设置 key-value 类型的值
OK
127.0.0.1:6379> mget key1 key2 # 批量获取多个 key 对应的 value
1) "value1"
2) "value2"
```

**Hash**

1. 介绍 ：hash 类似于 JDK1.8 前的 HashMap，内部实现也差不多(数组 + 链表)。不过，Redis 的 hash 做了更多优化。哈希是键值对的集合。在 Redis 中，哈希是字符串字段和字符串值之间的映射。因此，它比较适合存储对象。
2. 常用命令： hset,hmset,hexists,hget,hgetall,hkeys,hvals 等。
3. 应用场景: 系统中对象数据的存储。
hmset key field value [field value ...]

```
127.0.0.1:6379> HMSET user username "ajeet" password "javatpoint" alexa "2000"
OK
127.0.0.1:6379> HGETALL
1) "username"
2) "ajeet"
3) "password"
4) "javatpoint"
5) "alexa"
6) "2000"
127.0.0.1:6379>HGET user username
"ajeet"
```

HMSET 和 HGETALL 是 Redis 的命令，而 user 是键。每个 hash 可以存储 2^32 -1 键值对（40多亿）。

**List**

1. 介绍：Redis 列表定义为字符串列表，按插入顺序排序。可以将元素添加到 Redis 列表的头部或尾部。Redis 的 list 的实现为一个 双向链表，即可以支持反向查找和遍历，更方便操作，不过带来了部分额外的内存开销。
2. 常用命令: rpush,lpop,lpush,rpop,lrange,llen 等。
3. 应用场景: 发布与订阅或者说消息队列、慢查询。

```
127.0.0.1:6379> lpush list1 redis
(integer) 1
127.0.0.1:6379> lpush list1 mongodb
(integer) 2
127.0.0.1:6379> lpush list1 rabbitmq
(integer) 3
127.0.0.1:6379> lpush list1 rocketmq
(integer) 4
127.0.0.1:6379> lrange list1 0 10
1) "rocketmq"
2) "rabbitmq"
3) "mongodb"
4) "redis"
```

列表的最大长度为 2^32 – 1 个元素（超过 40 亿个元素）。

**Set**

1. 介绍 ： set 类似于 Java 中的 HashSet 。Redis 中的 set 类型是一种无序集合，集合中的元素没有先后顺序。当你需要存储一个列表数据，又不希望出现重复数据时，set 是一个很好的选择，并且 set 提供了判断某个成员是否在一个 set 集合内的重要接口，这个也是 list 所不能提供的。可以基于 set 轻易实现交集、并集、差集的操作。比如：你可以将一个用户所有的关注人存在一个集合中，将其所有粉丝存在一个集合。Redis 可以非常方便的实现如共同关注、共同粉丝、共同喜好等功能。这个过程也就是求交集的过程。
2. 常用命令： sadd,spop,smembers,sismember,scard,sinterstore,sunion 等。
3. 应用场景: 需要存放的数据不能重复以及需要获取多个数据源交集和并集等场景。

集合是通过哈希表实现的，所以添加，删除，查找的复杂度都是 O(1)。
sadd 命令:

添加一个 string 元素到 key 对应的 set 集合中，成功返回 1，如果元素已经在集合中返回 0。

```
127.0.0.1:6379> sadd mySet value1 value2 # 添加元素进去
(integer) 2
127.0.0.1:6379> sadd mySet value1 # 不允许有重复元素
(integer) 0
127.0.0.1:6379> smembers mySet # 查看 set 中所有的元素
1) "value1"
2) "value2"
127.0.0.1:6379> scard mySet # 查看 set 的长度
(integer) 2
127.0.0.1:6379> sismember mySet value1 # 检查某个元素是否存在set 中，只能接收单个元素
(integer) 1
127.0.0.1:6379> sadd mySet2 value2 value3
(integer) 2
127.0.0.1:6379> sinterstore mySet3 mySet mySet2 # 获取 mySet 和 mySet2 的交集并存放在 mySet3 中
(integer) 1
127.0.0.1:6379> smembers mySet3
1) "value2"

```

**Sorted Set**

1. 介绍： 和 set 相比，sorted set 增加了一个权重参数 score，使得集合中的元素能够按 score 进行有序排列，还可以通过 score 的范围来获取元素的列表。有点像是 Java 中 HashMap 和 TreeSet 的结合体。
2. 常用命令：`zadd,zcard,zscore,zrange,zrevrange,zrem` 等。
3. 应用场景： 需要对数据根据某个权重进行排序的场景。比如在直播系统中，实时排行信息包含直播间在线用户列表，各种礼物排行榜，弹幕消息（可以理解为按消息维度的消息排行榜）等信息。

zset的成员是唯一的,但分数(score)却可以重复。

zadd 命令:

添加元素到集合，元素在集合中存在则更新对应score

```
zadd key score member 
```

```
127.0.0.1:6379> zadd key 0 redis
(integer) 1
127.0.0.1:6379> zadd key 1 mongodb
(integer) 1
127.0.0.1:6379> zadd key 2 abc
(integer) 1
127.0.0.1:6379> zadd 0 rabbitmq
(error) ERR wrong number of arguments for 'zadd' command
127.0.0.1:6379> zadd key 0 rabbitmq
(integer) 1
127.0.0.1:6379> zrangebyscore key 0 100
1) "rabbitmq"
2) "redis"
3) "mongodb"
4) "abc"
```

**位图 Redis Bitmap**
1. 介绍： bitmap 存储的是连续的二进制数字（0 和 1），通过 bitmap, 只需要一个 bit 位来表示某个元素对应的值或者状态，key 就是对应元素本身 。我们知道 8 个 bit 可以组成一个 byte，所以 bitmap 本身会极大的节省储存空间。
2. 常用命令： setbit 、getbit 、bitcount、bitop
3. 应用场景： 适合需要保存状态信息（比如是否签到、是否登录...）并需要进一步对这些信息进行分析的场景。比如用户签到情况、活跃用户情况、用户行为统计（比如是否点赞过某个视频）。
Redis Bitmap 通过类似 map 结构存放 0 或 1 ( bit 位 ) 作为值。、
Redis Bitmap 可以用来统计状态，如`日活`是否浏览过某个东西。
Redis setbit 命令用于设置或者清除一个 bit 位。
Redis setbit 命令语法格式
```
SETBIT key offset value
```
```
# SETBIT 会返回之前位的值（默认是 0）这里会生成 7 个位
127.0.0.1:6379> setbit mykey 7 1
(integer) 0
127.0.0.1:6379> setbit mykey 7 0
(integer) 1
127.0.0.1:6379> getbit mykey 7
(integer) 0
127.0.0.1:6379> setbit mykey 6 1
(integer) 0
127.0.0.1:6379> setbit mykey 8 1
(integer) 0
# 通过 bitcount 统计被被设置为 1 的位的数量。
127.0.0.1:6379> bitcount mykey
(integer) 2
```
## 三、数据结构

### 字典
字典（dictionary）， 又名映射（map）或关联数组（associative array）， 是一种抽象数据结构， 由一集键值对（key-value pairs）组成， 各个键值对的键各不相同， 程序可以添加新的键值对到字典中， 或者基于键进行查找、更新或删除等操作。
字典在 Redis 中的应用广泛， 使用频率可以说和 SDS 以及双端链表不相上下， 基本上各个功能模块都有用到字典的地方。
其中， 字典的主要用途有以下两个：
1. 实现数据库键空间（key space）；
2. 用作 Hash 类型键的底层实现之一；

### 跳跃表

跳跃表（[skiplist](http://en.wikipedia.org/wiki/Skip_list)）是一种随机化的数据， 由 William Pugh 在论文[《Skip lists: a probabilistic alternative to balanced trees》](http://www.cl.cam.ac.uk/teaching/0506/Algorithms/skiplists.pdf)中提出， 跳跃表以有序的方式在层次化的链表中保存元素， 效率和平衡树媲美 —— 查找、删除、添加等操作都可以在对数期望时间下完成， 并且比起平衡树来说， 跳跃表的实现要简单直观得多。

![../_images/skiplist.png](https://redisbook.readthedocs.io/en/latest/_images/skiplist.png)

从图中可以看到， 跳跃表主要由以下部分构成：

- 表头（head）：负责维护跳跃表的节点指针。
- 跳跃表节点：保存着元素值，以及多个层。
- 层：保存着指向其他元素的指针。高层的指针越过的元素数量大于等于低层的指针，为了提高查找的效率，程序总是从高层先开始访问，然后随着元素值范围的缩小，慢慢降低层次。
- 表尾：全部由 `NULL` 组成，表示跳跃表的末尾。

跳跃表在 Redis 的唯一作用， 就是实现有序集数据类型。

跳跃表将指向有序集的 `score` 值和 `member` 域的指针作为元素， 并以 `score` 值为索引， 对有序集元素进行排序。

## 四、使用场景

### 计数器

可以对 String 进行自增自减运算，从而实现计数器功能。

Redis 这种内存型数据库的读写性能非常高，很适合存储频繁读写的计数量。

### 缓存

将热点数据放到内存中，设置内存的最大使用量以及淘汰策略来保证缓存的命中率。

### 查找表

例如 DNS 记录就很适合使用 Redis 进行存储。
查找表和缓存类似，也是利用了 Redis 快速的查找特性。但是查找表的内容不能失效，而缓存的内容可以失效，因为缓存不作为可靠的数据来源。

### 消息队列

List 是一个双向链表，可以通过 lpush 和 rpop 写入和读取消息
不过最好使用 Kafka、RabbitMQ 等消息中间件。

### 会话缓存

可以使用 Redis 来统一存储多台应用服务器的会话信息。
当应用服务器不再存储用户的会话信息，也就不再具有状态，一个用户可以请求任意一个应用服务器，从而更容易实现高可用性以及可伸缩性。

### 分布式锁实现

场景：在分布式场景下，无法使用单机环境下的锁来对多个节点上的进程进行同步。
可以使用 Redis 自带的 SETNX 命令实现分布式锁，除此之外，还可以使用官方提供的 RedLock 分布式锁实现。
先拿setnx来争抢锁，抢到之后，再用expire给锁加一个过期时间防止锁忘记了释放。
**如果在setnx之后执行expire之前进程意外crash或者要重启维护了，那会怎么样？**
set指令有非常复杂的参数，这个应该是可以同时把setnx和expire合成一条指令来用的！

## 五、Redis与Memcached

两者都是非关系型内存键值数据库，主要有以下不同：

### 数据类型
Memcached 仅支持字符串类型，而 Redis 支持五种不同的数据类型，可以更灵活地解决问题。

### 数据持久化
Redis 支持两种持久化策略：RDB 快照和 AOF 日志，而 Memcached 不支持持久化。

### 分布式
Memcached 不支持分布式，只能通过在客户端使用一致性哈希来实现分布式存储，这种方式在存储和查询时都需要先在客户端计算一次数据所在的节点。
Redis Cluster 实现了分布式的支持。

### 内存管理机制
在 Redis 中，并不是所有数据都一直存储在内存中，可以将一些很久没用的 value 交换到磁盘，而 Memcached 的数据则会一直在内存中。

Memcached 将内存分割成特定长度的块来存储数据，以完全解决内存碎片的问题。但是这种方式会使得内存的利用率不高，例如块的大小为 128 bytes，只存储 100 bytes 的数据，那么剩下的 28 bytes 就浪费掉了。
### 实现机制
Memcached 是多线程，非阻塞 IO 复用的网络模型；
Redis 使用单线程的多路 IO 复用模型。 （Redis 6.0 引入了多线程 IO ）

## 六、 键的过期时间
Redis 可以为每个键设置过期时间，当键过期时，会自动删除该键。
对于散列表这种容器，只能为整个键设置过期时间（整个散列表），而不能为键里面的单个元素设置过期时间。

## 七、数据淘汰策略

可以设置内存最大使用量，当内存使用量超出时，会施行数据淘汰策略。

Redis 具体有 6 种淘汰策略：

|      策略       |                         描述                         |
| :-------------: | :--------------------------------------------------: |
|  volatile-lru   | 从已设置过期时间的数据集中挑选最近最少使用的数据淘汰 |
|  volatile-ttl   |   从已设置过期时间的数据集中挑选将要过期的数据淘汰   |
| volatile-random |      从已设置过期时间的数据集中任意选择数据淘汰      |
|   allkeys-lru   |       从所有数据集中挑选最近最少使用的数据淘汰       |
| allkeys-random  |          从所有数据集中任意选择数据进行淘汰          |
|   noeviction    |                     禁止驱逐数据                     |

作为内存数据库，出于对性能和内存消耗的考虑，Redis 的淘汰算法实际实现上并非针对所有 key，而是抽样一小部分并且从中选出被淘汰的 key。

使用 Redis 缓存数据时，为了提高缓存命中率，需要保证缓存数据都是热点数据。可以将内存最大使用量设置为热点数据占用的内存量，然后启用 allkeys-lru 淘汰策略，将最近最少使用的数据淘汰。

Redis 4.0 引入了 volatile-lfu 和 allkeys-lfu 淘汰策略，LFU 策略通过统计访问频率，将访问频率最少的键值对淘汰。
## 八、 持久化
Redis 是内存型数据库，为了保证数据在断电后不会丢失，需要将内存中的数据持久化到硬盘上。
### RDB 持久化
将某个时间点的所有数据都存放到硬盘上。
可以将快照复制到其它服务器从而创建具有相同数据的服务器副本。
如果系统发生故障，将会丢失最后一次创建快照之后的数据。
如果数据量很大，保存快照的时间会很长。

### AOF 持久化
将写命令添加到 AOF 文件（Append Only File）的末尾。
使用 AOF 持久化需要设置同步选项，从而确保写命令同步到磁盘文件上的时机。这是因为对文件进行写入并不会马上将内容同步到磁盘上，而是先存储到缓冲区，然后由操作系统决定什么时候同步到磁盘。有以下同步选项：

|   选项   |         同步频率         |
| :------: | :----------------------: |
|  always  |     每个写命令都同步     |
| everysec |       每秒同步一次       |
|    no    | 让操作系统来决定何时同步 |

- always 选项会严重减低服务器的性能；
- everysec 选项比较合适，可以保证系统崩溃时只会丢失一秒左右的数据，并且 Redis 每秒执行一次同步对服务器性能几乎没有任何影响；
- no 选项并不能给服务器性能带来多大的提升，而且也会增加系统崩溃时数据丢失的数量。

随着服务器写请求的增多，AOF 文件会越来越大。Redis 提供了一种将 AOF 重写的特性，能够去除 AOF 文件中的冗余写命令。