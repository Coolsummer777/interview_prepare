# 关系型数据库的发展

1 redis缓存

2 Master/Slaver

主从复制 读写分离

3 主库写压力瓶颈--》分库分表

业务相关一个库

业务不相关一个库

4 拓展性瓶颈 大文本字段



传统关系型数据库难以支持 --》 NoSQL 

泛指非关系型数据库

无需事先为要存储的数据建立字段



## 传统数据库

ACID

Atomicity 原子性

consistency 一致性

isolation 独立性

durability 持久性



## 关系型数据库

### CAP

consistency 强一致性

Availability 可用性

Partition tolerance 分区容错性

CAP只能三选二

传统数据库 ：CA mysql 强一致性 + 高可用性

大多数网站选择：AP

CP redis



### BASE

基本可用 Bascially Aviailable

软状态Soft state

最终一致 Eventually consistency



### 分布式和集群的区别？

分布式：不同多台服务器部署不同的服务

集群：相同服务



### Redis REmote DIctionary Server

**Redis 底层是 I/O 多路复用的Reactor 模式，封装epoll函数，从而用一个线程监听多个socket，并将到达的事件传送给dispatcher，调用相应的event handler处理。**

redis默认16个库，select切换

redis单线程，因此保证了其原子性。

指令存入队列。

# redis五大数据类型

![img](https://img2018.cnblogs.com/blog/1289934/201906/1289934-20190621163930814-1395015700.png)

## String 字符串 

格式: set key value

是redis中最基本的数据类型，一个key对应一个value。

string类型是二进制安全的。意思是redis的string可以包含任何数据。比如jpg图片或者序列化的对象 。

string类型是Redis最基本的数据类型，一个键最大能存储512MB。

get

set

append

strlen

incr

decr 

**1.做缓存： 把常用信息，字符串，图片或者视频等信息放到redis中**

**2.做分布式Session，Key是SessionID，V是用户信息**

## Hash 哈希

场景：![image-20191206223750082](C:\Users\we\AppData\Roaming\Typora\typora-user-images\image-20191206223750082.png)

格式: hmset key field1 value1 field2 value2  field3 value3

​		hget key filed1

**Value值是键值对，底层是字典，可以用来用来存对象**

## List 列表

单值多value

底层是一个带头尾指针的双向链表

lpush/rpush/lrange

lpush 顶部（left)添加字符串

lpop

rpush 底部添加字符串

rpop

brpop 移出并获取列表的最后一个元素， 如果列表没有元素会阻塞列表直到等待超时或发现可弹出元素为止

应用场景： 

- lpush+lpop=Stack(栈)
- lpush+rpop=Queue（队列）
- lpush+brpop=Message Queue（消息队列）

## Set 集合

String类型的无序集合，底层是value为null的hash表



## Zset 有序集合

**zset是一个一个有序集合，set基础上加入了一个score字段，通过score排序。底层是跳跃表**

**redis使用跳表不用B+数的原因是**：redis存在内存数据库，而B+树纯粹是为了mysql这种IO数据库准备的。节点的大小设为等于一个页，一个节点的载入只需要一次I/O

优点是插入只要修改前后指针，利于范围查找

![img](https://upload-images.jianshu.io/upload_images/5868227-23909e26d4318146.png?imageMogr2/auto-orient/strip|imageView2/2/w/1072/format/webp)

应用场景： 例如视频网站需要对视频做排行榜，榜单可以按照点击量排序。

格式: zadd name score value

Redis zset 和 set 一样也是string类型元素的集合,且不允许重复的成员。

每个元素都会关联一个double类型的分数，通过分数排序。





# Redis事务

对于单个完整的命令, 其处理都是原子性的, 但是如果需要将**多个命令作为一个不可分割的处理序列**, 就需要使用事务。Redis通过multi exec watch等命令实现事务。底层通过一个事务队列来实现，通过FIFO保存入队命令。

MULTI 事务开始命令

EXEC 事务提交命令

WATCH是一个乐观锁，在EXEX命令执行之前，监视数据库键，如果至少一个被修改过，就将拒绝执行事务。

Redis不支持事务回滚，一个命令出现错误，后续命令也会继续执行下去。原因是这种复杂功能和redis简单高效的主题不符。

# Redis持久化

**什么是Redis持久化？Redis有哪几种持久化方式？优缺点是什么？**

持久化就是把内存的数据写到磁盘中去，防止服务宕机了内存数据丢失。

Redis 提供了两种持久化方式:RDB（默认） 和AOF 

## **RDB：**

rdb是Redis DataBase缩写

**将内存中数据库快照写入磁盘，生成一个二进制RDB文件。恢复时将rdb文件读入内存。可以手动也可以定期执行。生成RDB文件有两种命令，save：主进程执行，阻塞主进程，bgsave： 子进程执行。持久化结束后替换上次持久化的文件。**

比AOF高效，缺点是最后一次持久化之后的数据可能丢失。

功能核心函数rdbSave(生成RDB文件)和rdbLoad（从文件加载内存）两个函数

![img](https://img2018.cnblogs.com/blog/1481291/201809/1481291-20180925141429889-1694430603.png)

## **AOF:**

Aof是Append-only file缩写

将写命令添加到 AOF 文件（Append Only File）的末尾。

使用 AOF 持久化需要设置同步选项，从而确保写命令同步到磁盘文件上的时机。这是因为对文件进行写入并不会马上将内容同步到磁盘上，而是先存储到缓冲区，然后由操作系统决定什么时候同步到磁盘。有以下同步选项：

| 选项     | 同步频率                 |
| -------- | ------------------------ |
| always   | 每个写命令都同步         |
| everysec | 每秒同步一次             |
| no       | 让操作系统来决定何时同步 |

- always 选项会严重减低服务器的性能；
- everysec 选项比较合适，可以保证系统崩溃时只会丢失一秒左右的数据，并且 Redis 每秒执行一次同步对服务器性能几乎没有任何影响；
- no 选项并不能给服务器性能带来多大的提升，而且也会增加系统崩溃时数据丢失的数量。

随着服务器写请求的增多，AOF 文件会越来越大。Redis 提供了一种将 AOF 重写的特性，能够去除 AOF 文件中的冗余写命令。

![img](https://img2018.cnblogs.com/blog/1481291/201809/1481291-20180925141527592-2105439510.png)

每当执行服务器(定时)任务或者函数时flushAppendOnlyFile 函数都会被调用， 这个函数执行以下两个工作

aof写入保存：

WRITE：根据条件，将 aof_buf 中的缓存写入到 AOF 文件

SAVE：根据条件，调用 fsync 或 fdatasync 函数，将 AOF 文件保存到磁盘中。

**存储结构:**

 内容是redis通讯协议(RESP )格式的命令文本存储。

**比较**：

1、aof文件比rdb更新频率高，优先使用aof还原数据。

2、aof比rdb更安全也更大

3、rdb性能比aof好

4、如果两个都配了优先加载AOF

**刚刚上面你有提到redis通讯协议(RESP )，能解释下什么是RESP？有什么特点？（可以看到很多面试其实都是连环炮，面试官其实在等着你回答到这个点，如果你答上了对你的评价就又加了一分）**

RESP 是redis客户端和服务端之前使用的一种通讯协议；

RESP 的特点：实现简单、快速解析、可读性好

For Simple Strings the first byte of the reply is "+" 回复

For Errors the first byte of the reply is "-" 错误

For Integers the first byte of the reply is ":" 整数

For Bulk Strings the first byte of the reply is "$" 字符串

For Arrays the first byte of the reply is "*" 数组



# 复制

通过使用 slaveof host port 命令来让一个服务器成为另一个服务器的从服务器。

一个从服务器只能有一个主服务器，并且不支持主主复制。

## 连接过程

1. master创建快照文件，发送给slave，并在发送期间使用缓冲区记录执行的写命令。快照文件发送完毕之后，开始向slave发送存储在缓冲区中的写命令；
2. slave丢弃所有旧数据，载入master发来的快照文件，之后slave接受master发来的写命令；
3. master每次写，就向slave发送相同的写命令。

## 主从链

随着负载不断上升，主服务器可能无法很快地更新所有从服务器，或者重新连接和重新同步从服务器将导致系统超载。为了解决这个问题，可以创建一个中间层来分担主服务器的复制工作。中间层的服务器是最上层服务器的从服务器，又是最下层服务器的主服务器。

[![img](https://camo.githubusercontent.com/db582544b7e7d6b094f7e051eb2f76f0b2f8b3ff/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f33393561396538332d623161312d346131642d623137302d6430383165376262356261622e706e67)](https://camo.githubusercontent.com/db582544b7e7d6b094f7e051eb2f76f0b2f8b3ff/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f33393561396538332d623161312d346131642d623137302d6430383165376262356261622e706e67)



# 哨兵

Sentinel（哨兵）可以监听集群中的服务器，并在主服务器进入下线状态时，自动从从服务器中选举出新的主服务器。 

**Redis 有哪些架构模式？讲讲各自的特点**

 **单机版**

![img](https://img2018.cnblogs.com/blog/1481291/201809/1481291-20180925142100480-1152515615.png)

特点：简单

问题：

1、内存容量有限 2、处理能力有限 3、无法高可用。

**主从复制**

**![img](https://img2018.cnblogs.com/blog/1481291/201809/1481291-20180925142118041-1727225479.png)**

Redis 的复制（replication）功能允许用户根据一个 Redis 服务器来创建任意多个该服务器的复制品，其中被复制的服务器为主服务器（master），而通过复制创建出来的服务器复制品则为从服务器（slave）。 只要主从服务器之间的网络连接正常，主从服务器两者会具有相同的数据，主服务器就会一直将发生在自己身上的数据更新同步 给从服务器，从而一直保证主从服务器的数据相同。

特点：

1、master/slave 角色

2、master/slave 数据相同

3、降低 master 读压力在转交从库

问题：

无法保证高可用

没有解决 master 写的压力

**哨兵**

**![img](https://img2018.cnblogs.com/blog/1481291/201809/1481291-20180925142143478-1454265814.png)**

Redis sentinel 是一个分布式系统中监控 redis 主从服务器，并在主服务器下线时自动进行故障转移。其中三个特性：

监控（Monitoring）：  Sentinel 会不断地检查你的主服务器和从服务器是否运作正常。

提醒（Notification）： 当被监控的某个 Redis 服务器出现问题时， Sentinel 可以通过 API 向管理员或者其他应用程序发送通知。

自动故障迁移（Automatic failover）： 当一个主服务器不能正常工作时， Sentinel 会开始一次自动故障迁移操作。

特点：

1、保证高可用

2、监控各个节点

3、自动故障迁移

缺点：主从模式，切换需要时间丢数据

没有解决 master 写的压力

**集群（proxy 型）：**

**![img](https://img2018.cnblogs.com/blog/1481291/201809/1481291-20180925142206124-913246424.png)**

Twemproxy 是一个 Twitter 开源的一个 redis 和 memcache 快速/轻量级代理服务器； Twemproxy 是一个快速的单线程代理程序，支持 Memcached ASCII 协议和 redis 协议。

特点：1、多种 hash 算法：MD5、CRC16、CRC32、CRC32a、hsieh、murmur、Jenkins 

2、支持失败节点自动删除

3、后端 Sharding 分片逻辑对业务透明，业务方的读写方式和操作单个 Redis 一致

缺点：增加了新的 proxy，需要维护其高可用。

 

failover 逻辑需要自己实现，其本身不能支持故障的自动转移可扩展性差，进行扩缩容都需要手动干预

**集群（直连型）：**

**![img](https://img2018.cnblogs.com/blog/1481291/201809/1481291-20180925142304757-1498788186.png)**

从redis 3.0之后版本支持redis-cluster集群，Redis-Cluster采用无中心结构，每个节点保存数据和整个集群状态,每个节点都和其他所有节点连接。

特点：

1、无中心架构（不存在哪个节点影响性能瓶颈），少了 proxy 层。

2、数据按照 slot 存储分布在多个节点，节点间数据共享，可动态调整数据分布。

3、可扩展性，可线性扩展到 1000 个节点，节点可动态添加或删除。

4、高可用性，部分节点不可用时，集群仍可用。通过增加 Slave 做备份数据副本

5、实现故障自动 failover，节点之间通过 gossip 协议交换状态信息，用投票机制完成 Slave到 Master 的角色提升。

缺点：

1、资源隔离性较差，容易出现相互影响的情况。

2、数据通过异步复制,不保证数据的强一致性

 

**什么是一致性哈希算法？什么是哈希槽？**

将数据key使用相同的函数Hash计算出哈希值，并确定此数据在环上的位置，从此位置沿环顺时针“行走”，第一台遇到的服务器就是其应该定位到的服务器。

这两个问题篇幅过长 网上找了两个解锁的不错的文章

https://www.cnblogs.com/lpfuture/p/5796398.html

https://blog.csdn.net/z15732621582/article/details/79121213

 

 

**Redis常用命令？**

Keys pattern

*表示区配所有

以bit开头的

查看Exists key是否存在

Set

设置 key 对应的值为 string 类型的 value。

setnx

设置 key 对应的值为 string 类型的 value。如果 key 已经存在，返回 0，nx 是 not exist 的意思。

删除某个key

第一次返回1 删除了 第二次返回0

Expire 设置过期时间（单位秒）

TTL查看剩下多少时间

返回负数则key失效，key不存在了

Setex

设置 key 对应的值为 string 类型的 value，并指定此键值对应的有效期。

Mset

一次设置多个 key 的值，成功返回 ok 表示所有的值都设置了，失败返回 0 表示没有任何值被设置。

Getset

设置 key 的值，并返回 key 的旧值。

Mget

一次获取多个 key 的值，如果对应 key 不存在，则对应返回 nil。

Incr

对 key 的值做加加操作,并返回新的值。注意 incr 一个不是 int 的 value 会返回错误，incr 一个不存在的 key，则设置 key 为 1

incrby

同 incr 类似，加指定值 ，key 不存在时候会设置 key，并认为原来的 value 是 0

Decr

对 key 的值做的是减减操作，decr 一个不存在 key，则设置 key 为-1

Decrby

同 decr，减指定值。

Append

给指定 key 的字符串值追加 value,返回新字符串值的长度。

Strlen

取指定 key 的 value 值的长度。

persist xxx(取消过期时间)

选择数据库（0-15库）

Select 0 //选择数据库

move age 1//把age 移动到1库

Randomkey随机返回一个key

Rename重命名

Type 返回数据类型

**08**

**使用过Redis分布式锁么，它是怎么实现的？**

 

先拿setnx来争抢锁，抢到之后，再用expire给锁加一个过期时间防止锁忘记了释放。

**如果在setnx之后执行expire之前进程意外crash或者要重启维护了，那会怎么样？**

set指令有非常复杂的参数，这个应该是可以同时把setnx和expire合成一条指令来用的！

**09**

**使用过Redis做异步队列么，你是怎么用的？有什么缺点？**

一般使用list结构作为队列，rpush生产消息，lpop消费消息。当lpop没有消息的时候，要适当sleep一会再重试。

缺点：

在消费者下线的情况下，生产的消息会丢失，得使用专业的消息队列如rabbitmq等。

**能不能生产一次消费多次呢？**

使用pub/sub主题订阅者模式，可以实现1:N的消息队列。

**10**

**什么是缓存穿透？如何避免？什么是缓存雪崩？何如避免？**

缓存穿透

一般的缓存系统，都是按照key去缓存查询，如果不存在对应的value，就应该去后端系统查找（比如DB）。一些恶意的请求会故意查询不存在的key,请求量很大，就会对后端系统造成很大的压力。这就叫做缓存穿透。

如何避免？

1：对查询结果为**空**的情况也进行**缓存**，缓存时间设置短一点，或者该key对应的数据insert了之后清理缓存。

2：对一定不存在的key进行过滤。**布隆过滤器**是一个 bit 向量或者说 bit 数组，过滤一定不存在的key

缓存雪崩

当缓存服务器重启或者大量缓存集中在某一个时间段失效，这样在失效的时候，会给后端系统带来很大压力。导致系统崩溃。

如何避免？

1：在缓存失效后，通过加锁或者队列来控制读数据库写缓存的线程数量。比如对某个key只允许一个线程查询数据和写缓存，其他线程等待。

2：做二级缓存，A1为原始缓存，A2为拷贝缓存，A1失效时，可以访问A2，A1缓存失效时间设置为短期，A2设置为长期

3：不同的key，设置不同的过期时间，让缓存失效的时间点尽量均匀

# 问题

Redis 有哪些数据结构？ 

介绍下 Redis 的基本数据结构 

说一下这 5 种数据结构的底层实现 

说一下你看过的 Redis 源码 

Redis 的哪种数据类型用到了跳表结构？

list 如何实现的异步消息队列？ 

看你项目中用 Redis 中的 List 来实现异步队列，说一下具体是怎么做的？是如何基于 Redis 来实现异步的？有没有一个拉取消息的过程？还是说基于 Redis 你就把它放到队列里，然后有人来处理还是说订阅处理 

Redis 在单线程下实现高并发的？核心的机制是什么？ 

内存，IO多路复用，单线程

![img](https://upload-images.jianshu.io/upload_images/7368936-fe23b577eef07aa3.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

IO 多路复用模型有哪些？ 

select 和 epoll 有什么区别？ 

redis的数据结构,集群是怎么做的

Redis缓存一致性 

redis和 数据库是怎么保持一致性的？ 

## redis的更新策略（先操作数据库还是先操作缓存）、

只能降低不一致概率，不能保证强一致性。

先删缓存，后写数据库的缺点：A线程写数据，B线程读数据，A线程删除了缓存，B线程读数据发现缓存没有命中从数据库中读数据，B线程把读出的旧数据写到Redis里，A线程把新数据写回去。

延时双删策略

1）先删除缓存

2）再写数据库

3）休眠500毫秒

4）再次删除缓存

Redis缓存的实现方式 	

java用jedis

## Redis的缓存淘汰策略

删除方式：定期删除+惰性删除

定期删除是每隔100ms抽一些key检查是否过期

惰性删除是查询key时检查是否过期，过期则删除

有六种策略，一种是不驱逐，满了就报错。剩下的分为两类，一类是对所有key操作

allkeys-lru: 所有key通用; 优先删除最近最少使用(less recently used ,LRU) 的 key。
allkeys-random: 所有key通用; 随机删除一部分 key。

另一类是对设置了过期时间的key操作

volatile-lru: ; 优先删除最近最少使用(less recently used ,LRU) 的 key。

volatile-random: ; 随机删除一部分 key。
volatile-ttl: ; 优先删除TTL(time to live,TTL) 短的key。

分布式锁的实现方式，zk实现和redis实现哪个比较好

## redis的热点key问题

**热点key：**　　缓存中的某些Key(可能对应用与某个促销商品)对应的value存储在集群中一台机器，使得所有流量涌向同一机器，成为系统的瓶颈，该问题的挑战在于它无法通过增加机器容量来解决。

**热点key的解决方案：**

- 客户端热点key缓存：用一个`HashMap`，将热点key对应value并缓存在客户端本地（或者服务器JVM），并且设置一个失效时间。对于每次读请求，将首先检查key是否存在于本地缓存中，如果存在则直接返回，如果不存在再去访问分布式缓存的机器。
- 将热点key分散为多个子key，然后存储到redis集群的不同机器上，当通过热点key去查询数据时，通过某种hash算法随机选择一个子key，然后再去访问缓存机器，将热点分散到了多个子key上。

# Redis并发竞争key

多个redis的client同时set key

**分布式锁+时间戳**


# 补充问题

我将 Redis 面试中常见的题目划分为如下几大部分： 

1. Redis 的概念理解 
2. Redis 基本数据结构详解 
3. Redis 高并发问题策略 
4. Redis 集群结构以及设计理念 
5. Redis 持久化机制 
6. Redis 应用场景设计 

部分涉及到的题目如下：

- 什么是 Redis? 
- Redis 的特点有哪些？ 
- Redis 支持的数据类型 
- 为什么 Redis 需要把所有数据放到内存中？ 
- Redis 适用场景有哪些？ 
- Redis常用的业务场景有哪些？ 
- Mem*** 与 Redis 的区别都有哪些？     
- Redis 相比 mem***d 有哪些优势？ 
- Redis常用的命令有哪些？ 
- Redis 是单线程的吗？ 
- Redis 为什么设计成单线程的？ 
- 一个字符串类型的值能存储最大容量是多少？ 
- Redis各个数据类型最大存储量分别是多少? 
- Redis 持久化机制有哪些？ 区别是什么？ 
- 请介绍一下 RDB, AOF两种持久化机制的优缺点？ 
- 什么是缓存穿透？怎么解决？ 
- 什么是缓存雪崩？ 怎么解决？ 
- Redis支持的额Java客户端有哪些？ 简单说明一下特点。 
- 缓存的更新策略有几种？分别有什么注意事项？ 
- 什么是分布式锁？有什么作用？   
- 分布式锁可以通过什么来实现？  
- 介绍一下分布式锁实现需要注意的事项？  
- Redis怎么实现分布式锁？  
- 常见的淘汰算法有哪些？  
- Redis 淘汰策略有哪些？ 
- Redis 缓存失效策略有哪些？  
- Redis 的持久化机制有几种方式？  
- 请介绍一下持久化机制 RDB, AOF的优缺点分别是什么？ 
- Redis 通讯协议是什么？有什么特点？ 
- 请介绍一下 Redis 的数据类型 SortedSet(zset) 以及底层实现机制？ 
- Redis 集群最大节点个数是多少？ 
- Redis 集群的主从复制模型是怎样的？ 
- Redis 如何做内存优化？ 
- Redis 事务相关命令有哪些？ 
- 什么是 Redis 事务？原理是什么？ 
- Redis 事务的注意点有哪些？  
- Redis 为什么不支持回滚？  
- 请介绍一下 Redis 集群实现方案 
- 请介绍一下 Redis 常见的业务使用场景？ 
- Redis 集群会有写操作丢失吗？为什么？ 
- 请介绍一下 Redis 的 Pipeline (管道)，以及使用场景 
- 请说明一下 Redis 的批量命令与 Pipeline 有什么不同？  
- Redis 慢查询是什么？通过什么配置？ 
- Redis 的慢查询修复经验有哪些？ 怎么修复的？  
- 请介绍一下 Redis 的发布订阅功能 
- 请介绍几个可能导致 Redis 阻塞的原因 
- 怎么去发现 Redis 阻塞异常情况？ 
- 如何发现大对象 
- Redis 的内存消耗分类有哪些？内存统计使用什么命令？  
- 简单介绍一下 Redis 的内存管理方式有哪些？  
- 如何设置 Redis 的内存上限？有什么作用？ 
- 什么是 bigkey？ 有什么影响？  
- 怎么发现bigkey? 
- 请简单描述一下 Jedis 的基本使用方法？ 
- Jedis连接池链接方法有什么优点？ 
- 冷热数据表示什么意思？  
- 缓存命中率表示什么？  
- 怎么提高缓存命中率？ 
- 如何优化 Redis 服务的性能？ 
- 如何实现本地缓存？请描述一下你知道的方式 
- 请介绍一下 Spring 注解缓存 
- 如果 AOF 文件的数据出现异常， Redis服务怎么处理？  
- Redis 的主从复制模式有什么优缺点?  
- Redis sentinel (哨兵) 模式优缺点有哪些？ 
- Redis 集群架构模式有哪几种？ 
- 如何设置 Redis 的最大连接数？查看Redis的最大连接数？查看Redis的当前连接数？  
- Redis 的链表数据结构的特征有哪些？  
- 请介绍一下 Redis 的 String 类型底层实现？ 
- Redis 的 String 类型使用 SSD 方式实现的好处？ 
- 设计一下在交易网站首页展示当天最热门售卖商品的前五十名商品列表?





#### 	