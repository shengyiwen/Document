### 1.什么是Redis

Redis本质上是一个**KV类型的内存数据库**，整个数据库统统加载在内存当中进行操作，定期通过异步操作把数据库数据flush到硬盘上进行保存。
因为是纯内存操作，Redis的性能非常出色，每秒可以处理超过 10万次读写操作，是已知性能最快的Key-Value DB。 Redis的出色之处不仅仅是性能，
Redis最大的魅力是支持保存多种数据结构，此外单个value的最大限制是1GB， 不像memcached只能保存1MB的数据，因此Redis可以用来实现很多有用的功能，
比方说用他的List来做FIFO双向链表，实现一个轻量级的高性能消息队列服务，用他的Set可以做高性能的tag系统等等。
另外Redis也可以对存入的Key-Value设置expire时间，因此也可以被当作一个功能加强版的memcached来用。 **Redis的主要缺点是数据库容量受到物理内存的限制**，
不能用作海量数据的高性能读写，因此Redis适合的场景主要局限在较小数据量的高性能操作和运算上


### 2.Redis和Memcache相比有哪些优势

- Memcache的值比较单一，均为简单的字符串，而Redis存储的值更加丰富

- Redis的速度比Memcache要快

- Redis可以持久化数据

### 3.Redis支持的数据结构

分别支持String、List(FIFO双向链表)、Set、Sorted Set、Hash

### 4.Redis主要消耗什么物理资源

是内存类型高性能数据库，主要消耗内存

### 5.Redis全称

Remote Dictionary server

### 6.Redis有哪些数据淘汰策略

- noeviction:返回错误当内存限制达到了，并客户端尝试执行让更多内存被使用的命令

- allkey-lru:尝试回收使用最少的键，使得新添加的数据有空间存放

- volatile-lru:尝试回收使用最少的键，但仅限于在过期集合的键，使得新添加的数据有空间存放

- allkeys-random:回收随机的键，使得新添加的数据有空间存放

- volatile-random:回收随机的键，但仅限于在过期集合的键

- volatile-ttl:回收过期集合的键，并且优先回收存活时间较短的键，使得新添加的数据有空间存放

### 7.Redis中String类型最大的值容量

String类型最大的只能保存512M

### 8.为什么Redis把数据都存放在内存里

Redis为了达到最快的读写速度将数据都读到内存中，并通过异步的方式将数据写入磁盘。所以redis具有快速和数据持久化的特征。如果不将数据放在内存中，磁盘I/O速度为严重影响redis的性能。在内存越来越便宜的今天，redis将会越来越受欢迎。 如果设置了最大使用的内存，则数据已有记录数达到内存限值后不能继续插入新值

### 9.Redis集群环境如何搭建

Redis cluster3.0自带集群特征，特点在于他的分布式算法不是一致性hash，而是使用hash槽的概念，以及自身支持节点设置从节点。**就是一致性hash的升级版**

### 10.Redis支持的java客户端

Redisson、Jedis、Lettuce等，但是官方推荐使用Redisson

### 11.Redis和Redisson关系

Redisson是一个高级的分布式协调Redis客服端，能帮助用户在分布式环境中轻松实现一些Java的对象 (Bloom filter, BitSet, Set, SetMultimap, ScoredSortedSet, SortedSet, Map, ConcurrentMap, List, ListMultimap, Queue, BlockingQueue, Deque, BlockingDeque, Semaphore, Lock, ReadWriteLock, AtomicLong, CountDownLatch, Publish / Subscribe, HyperLogLog)

### 12.Jedis和Redisson对比有哪些优缺点

Jedis是Redis的Java实现的客户端，其Api提供了比较全面的Redis命令支持

Redisson实现了分布式和可扩展的Java数据结构，和Jedis相比功能较为简单，不支持字符串的操作，不支持排序、事务、管道、分区等特征。Redisson的宗旨是促进使用者对Redis的关注分离，从而让使用者能够集中精力的放在业务上

### 13.Redis的密码配置和验证

设置密码，redis.conf中配置如下

    requirepass 123

授权密码

    auth 123

### 14.Redis集群的主从复制模型是怎么样的

为了使在部分节点或者大部分节点无法通讯的情况下，集群仍然难可以使用，所以集群使用了主从复制的模型，每个节点都会有N-1个复制品

### 15.Redis主从架构会有数据丢失吗

会存在，因为不是强一致性，这意味着实际的集群中存在丢失的情况

- 异步复制导致的数据丢失：因为master -> slave的复制是异步的，所以可能有部分数据还没复制到slave，master就宕机了，此时这些部分数据就丢失了

- 脑裂导致的数据丢失：某个master所在机器突然脱离了正常的网络，跟其他slave机器不能连接，但是实际上master还运行着，此时哨兵可能就会认为master宕机了，然后开启选举，将其他slave切换成了master。这个时候，集群里就会有两个master，也就是所谓的脑裂。此时虽然某个slave被切换成了master，但是可能client还没来得及切换到新的master，还继续写向旧master的数据可能也丢失了。因此旧master再次恢复的时候，会被作为一个slave挂到新的master上去，自己的数据会清空，重新从新的master复制数据


### 16.Redis主从复制的工作原理

- 一个Slave实例，无论是第一次连接还是重新连接到Master节点，都会发送SYNC命令

- Master接收到SYNC命令后，首先会执行BGSAVE，即后台保存数据到磁盘(RDB文件)，然后同时将接收到写入和修改数据集的命令保存到缓冲区

- 当Master在后台把数据保存到快照文件完成后，Master会把快照文件发送给Slave,从而把Slave内存清空，加载文件内从

- Master也会把收集到缓存区的命令通过Redis协议发送给Slave,Slave执行这些命令实现数据同步

- Master/Slave会不断通过异步方式进行命令同步，达到数据一致性


### 17.Redis key的过期时间和永久有效如何设置

expire和persist命令

### 18.Redis回收工作如何做的

一个客户端运行了新命令添加数据，Redis会检查内存使用情况，如果大于maxMemory的限制，会根据设定好的策略进行回收。所以不断地穿越内存限制的边界，通过不断达到边界然后不断的会受到边界以内

### 19.Redis回收算法

LRU算法

### 20.Redis持久化的方式

RDB持久化方式能够在指定的时间间隔能对你的数据进行快照存储

AOF持久化方式记录每次对服务器写的操作,当服务器重启的时候会重新执行这些命令来恢复原始的数据,AOF命令以redis协议追加保存每次写的操作到文件末尾.Redis还能对AOF文件进行后台重写,使得AOF文件的体积不至于过大

**可以同时开启两种持久化方式, 在这种情况下, 当redis重启的时候会优先载入AOF文件来恢复原始的数据,因为在通常情况下AOF文件保存的数据集要比RDB文件保存的数据集要完整.**


### 21.如何保证Redis高可用和高并发

Redis主从架构，一主多从，可以满足高可用和高并发。出现实例宕机自动进行主备切换，配置读写分离缓解Master读写压力

### 22.Redis高可用的方案如何实施，注意和集群的区别

使用官方推荐的哨兵(sentinel)机制就能实现，当主节点出现故障时，由Sentinel自动完成故障发现和转移，并通知应用方，实现高可用性

它有四个主要功能：

- 集群监控，负责监控redis master和slave进程是否正常工作

- 消息通知，如果某个redis实例有故障，那么哨兵负责发送消息作为报警通知给管理员

- 故障转移，如果master node挂掉了，会自动转移到slave node上

- 配置中心，如果故障转移发生了，通知client客户端新的master地址。

### 23.Redis哨兵模式的原理

通过sentinel模式启动Redis后，自动监控master/slave的运行状态，基本原理是：心跳机制+投票裁决

每个sentinel会向其它sentinel、master、slave定时发送消息，以确认对方是否活着，如果发现对方在指定时间内未回应，则暂时认为对方宕机

若哨兵群中的多数sentinel都报告某一master没响应，系统才认为该master真正宕机，通过Raft投票算法，从剩下的slave节点中，选一台提升为master，然后自动修改相关配置

### 24.Redis哨兵模式需要注意的问题

哨兵至少需要3个实例，来保证自己的健壮性，防止脑裂

### 25.由于主从延迟导致数据过期怎么处理

- 通过scan命令扫库：当Redis中的key被scan的时候，相当于访问了该key，同样也会做过期检测，充分发挥Redis惰性删除的策略。这个方法能大大降低了脏数据读取的概率，但缺点也比较明显，会造成一定的数据库压力，否则影响线上业务的效率

- Redis加入了一个新特性来解决主从不一致导致读取到过期数据问题，增加了key是否过期以及对主从库的判断，如果key已过期，当前访问的master则返回null；当前访问的是从库，且执行的是只读命令也返回null

### 26.Redis key过期策略

- 惰性删除：当读/写一个已经过期的key时，会触发惰性删除策略，直接删除掉这个过期key，很明显，这是被动的

- 定期删除：由于惰性删除策略无法保证冷数据被及时删掉，所以 Redis 会定期主动淘汰一批已过期的key


