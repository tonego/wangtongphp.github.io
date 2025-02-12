## 《Redis设计与实现》
https://docs.google.com/presentation/d/13BuOO_vV-NjHlKybtlV6QNsS-PpSwk_jImtX4eK6a0Q/edit#slide=id.g103645edc4d_0_321

### skiplist

###### skiplist vs 红黑树
skiplist好实现，区间批量操作ZRANGE性能一样好，并发场景锁范围小，可动态调整树高及内存占用。
但数量级少随机性差导致偶发性的性能问题，高度扩容后顺序大量插入数据会导致扩容前数据索引不会被高级索引cover

### 4.字典
业务必知： 我是用string还是hash呢？   -- 多数业务选hash
  ziplist相比占内存空间更小，若10个field近一倍的内存差别
  若不同field expire的需求强烈优string，不过也可通过field记录过期时间业务来判断
* Redis使用MurmurHash2算法来计算键的哈希值
* rehash。 ht[1]的大小为第一个大于等于ht[0].used x 2的2n（2的n次方幂）
* 自动扩展： 未执行BGSAVE|BGREWRITEAOF负载因子大于等于1。正在执行BGSAVE命令或者BGREWRITEAOF命令，并且哈希表的负载因子大于等于5。
* 负载因子= 哈希表已保存节点数量/ 哈希表大小。 load_factor = ht[0].used / ht[0].size
* 哈希表的负载因子小于0.1时，程序自动开始对哈希表执行收缩操作。
* 渐进式rehash：操作一个键的话，程序会先在ht[0]里面进行查找，如果没找到的话，就会继续到ht[1]里面进行查找
* 在rehash进行期间，每次对字典执行添加、删除、查找或者更新操作时，程序除了执行指定的操作以外，还会顺带将ht[0]哈希表在rehashidx索引上的所有键值对rehash到ht[1]，当rehash工作完成之后，程序将rehashidx属性的值增一
* 每次在进行SET/GET操作时，都会保证向前遍历旧数组1～10步，最终ht[0]将被遍历完，而ht[1]将越来越多

Java：
HashMap：我们最常用的Map，HashMap的值是没有顺序的，他是按照key的HashCode来实现的，就是根据key的HashCode 值来存储数据,根据key可以直接获取它的Value，同时它具有很快的访问速度。HashMap最多只允许一条记录的key值为Null(多条会覆盖);允许多条记录的Value为 Null。非同步的。
TreeMap: 能够把它保存的记录根据key排序,默认是按升序排序，也可以指定排序的比较器，当用Iterator 遍历TreeMap时，得到的记录是排过序的。TreeMap不允许key的值为null。非同步的。
Hashtable: 与 HashMap类似,不同的是:key和value的值均不允许为null;它支持线程的同步，即任一时刻只有一个线程能写Hashtable,因此也导致了Hashtale在写入时会比较慢，只有hashtable是继承自Dictionary抽象类的，hashMap和treeMap都继承自AbstractMap抽象类，LinkedHashMap继承自hashMap。
LinkedHashMap: 保存了记录的插入顺序，在用Iterator遍历LinkedHashMap时，先得到的记录肯定是先插入的.在遍历的时候会比HashMap慢。key和value均允许为空，非同步的。

### 7.压缩列表 ZIPLIST
```
zlbytes zltail zllen entry1 entry2{previous_entry_length encoding content} zlend
```
* 压缩列表是从表尾向表头遍历的，
* previous_entry_length 1字节或5字节； 如果前一节点的长度大于等于254字节，那么previ-ous_entry_length属性的长度为5字节：其中属性的第一字节会被设置为0xFE（十进制值254），而之后的四个字节则用于保存前一节点的长度。
* encoding 字节数组00开头1字节长，01 2， 10 5； 整数编码11开头1字节长
* 连锁更新。有多个连续的长度介于250字节到253字节之间的节点e1-eN，这些都只需1字节长的previous_entry_length，若e1增加长度便会引发连锁更新。这时最坏复杂度为O(N²)，出现的几率可以忽略

### 8.对象
```
STRING: INT EMBSTR RAW
LIST: ZIPLIST LINKEDLIST
HASH: ZIPLIST HT
SET: INTSET HT
ZSET: ZIPLIST SKIPLIST
```
### 9. 数据库

##### 9.5 过期key删除策略
定时删除策略的缺点是，它对CPU时间是最不友好的：在过期键比较多的情况下，删除过期键这一行为可能会占用相当一部分CPU时间，
惰性删除策略的缺点是，它对内存是最不友好的。
定期删除策略每隔一段时间执行一次删除过期键操作，并通过限制删除操作执行的时长和频率来减少删除操作对CPU时间的影响。  
除此之外，通过定期删除过期键，定期删除策略有效地减少了因为过期键而带来的内存浪费。
定期删除策略的难点是确定删除操作执行的时长和频率。

##### 9.6 redis过期key删除策略
Redis服务器实际使用的是惰性删除和定期删除两种策略。
定期删除：redis.c/serverCron函数执行时，activeExpire-Cycle函数就会被调用。
执行SAVE命令或者BGSAVE命令所产生的新RDB文件不会包含已经过期的键。
执行BGREWRITEAOF命令所产生的重写AOF文件不会包含已经过期的键。   
当一个过期键被删除之后，服务器会追加一条DEL命令到现有AOF文件的末尾，显式地删除过期键。
从服务器即使发现过期键也不会自作主张地删除它，而是等待主节点发来DEL命令，这种统一、中心化的过期键删除策略可以保证主从服务器数据的一致性。

##### 9.7 aof和rdb和复制功能对过期键处理

##### 9.9 重点回顾
数据库主要由dict和expires两个字典构成，其中dict字典负责保存键值对，而expires字典则负责保存键的过期时间。
expires字典的键指向数据库中的某个键，而值则记录了数据库键的过期时间，过期时间是一个以毫秒为单位的UNIX时间戳

### 10. rdb
saveparam/dirty/lastsave, serverCron()/per100ms , save(block)/bgsave(fork子进程)
载入数据，优先aof，其次rdb。

### 11. aof持久化
业务收益： 数据安全性评估（可能会丢2s数据）、堵塞场景(若)、内存暴涨的避免及内存评估（aof_rewrite_buf_blocks、fork子进程的数据）。
aof_buf/ aof_rewrite_buf_blocks/ aof_fd 
疑问：rewrite过程中， 新的aof文件是整库数据库读出来的，并不是基于正在跑的aof文件汇总的。  我有个问题，若目前的库有10G， 这10G是怎么不堵塞的进行快照的， 如何实现很快的从主进程拷到子进程，这时候整库的内存占用会达到20G吗， 好奇reids又没有mysql这种mvcc机制，是如何产生的快照数据呢。是通过cow和ref_count吗？ 
~~rdb落盘时确实会是复制出同样大小的内存块的， 不过业务方比如要求10g，那实际上阿里云的宿主机会预留出来20g以上。 是宿主机给你预留的，和你申请的内存没有关系。~~
~~比如我现在占用100G内存， 那bgrewriteaof fork子进程的时候拷数据过去，这时候应该是堵塞的吧， 100G内存拷贝需要几秒呢？ 就是为了防止出现你的说的情况， 咱们的单节点别说100g了，就是出现10g的也很少， 那10G内存拷贝， 大约堵塞毫秒级别。~~
持久化阻塞：fork阻塞、aof刷盘、HugePage写操作(CopyOrRewrite)
fork和cow。fork是指redis通过创建子进程来进行bgsave操作，cow指的是copy on write，子进程创建后，父子进程共享数据段，父进程继续提供读写服务，写脏的页面数据会逐渐和子进程分离开来
https://redisbook.readthedocs.io/en/latest/internal/aof.html
单机版建议 save 900 1 + aof
集群主从版， 仅靠Master-Slave Replication实现高可用性。 在Slave上可以只开启AOF, AOF重写的基础大小默认值64M改为5G. 防止误操作命令被rewrite无法回滚。
疑问：有replica还需要用aof持久化吗。可以做误操作命令的回滚，那万一赶上rewrite怎么办？

### 15.复制
业务须知：影响性能如主从节点的CPU、IO、带宽、线程堵塞, 影响数据安全如低版本的主从复制数据丢失
cli->slave: slaveof masterip port, M发sync通知S进行bgsave, S发rdb, S发缓冲区数据。
psync：offset(MS), runID(M), backlog(M); S发psync <runID> <offset>或PSYNC ? -1, M发+continue或+FULLRESYNC <runid><offset>, M找backlog队列里offset后的数据发给S
repl-backlog-size=2*second*write_size_per_second
详细步骤：SLAVEOF 127.0.0.1 6379; socket; PING->PONG|ERR|TIMEOUT; AUTH; S发REPLCONF listening-port <port-number>; PSYNC; M切为client写命令到S
心跳检测：每秒一次REPLCONF ACK <replication_offset>。 INFO replication查lag;min-slaves-[to-write|max-lag];补发缺失数据

### 16.Sentinel
仅用于主从版。qodis未用。 大多用脚本实现主从切换，无需用sentinel
min-slaves-to-write 1
min-slaves-max-lag 10
在脑裂场景下，最多就丢失10秒的数据.
client需要做降级写队列。
键分布模型16384 槽（slot） HASH_SLOT = CRC16(key) mod 16384
键哈希标签（Keys hash tags）为了在集群稳定的情况下（没有在做碎片重组操作）允许某些多键操作
CLUSTER NODES 命令
Redis 集群是一个网状结构，每个节点都通过 TCP 连接跟其他每个节点连接。

### 17.集群
一致性hash实现、slot实现
slot实现的重点：一个node会有多个区间的slot， 所以类似一致性hash避免了太多数据的转移。
clusterState.slots_to_keys(zset. score:slot;member:key) 保存了所有key和slot的关系。用于key的迁移过程。
clusterNode.slots[16384/8]是二进制数组记录了本节点负责的slot，用于一次性把节点负责的slot查出来&指派广播;
clusterState.slots[16384]记录每个slot的*clusterNode，用==clusterState.mysql判断slot是否当前节点负责;
强一致性. 异步复制 和 网络分区
在网络分裂出现期间， 客户端 Z1 可以向主节点 B 发送写命令的最大时间是有限制的， 这一时间限制称为节点超时时间（node timeout）
gossip协议， 所有的集群节点都通过TCP连接（TCP bus？）和一个二进制协议（集群连接，cluster bus）建立通信
客户端缓存键值和节点的映射的必要性, 客户端在接收到重定向错误（redirections errors） -MOVED 和 -ASK 的时候， 将命令重定向到其他节点。
CLUSTER REPLICATE <node_id>
clusterState.myself { slaveof = clusterState.nodes[nodeid]; flags = REDIS_NODE_SLAVE }
M.clusterNode{ numslaves; **slaves; }
故障检测：每秒发PING 未PONG标PFAIL（probable fail）, clusterState.nodes[nodeid].flags=REDIS_NODE_PFAIL; 节点互通消息后: clusterNode(PFAIL的){ *fail_reports{*node(谁报告的); time;}; }; 若超过半数，flags=REDIS_NODE_FAIL并gossip
故障转移：若S发现M下线，选M; S执行SLAVEOF no one; SLOT指派给自己; gossip PONG; run;  
选主：S发现M下线gossip CLUSTERMSG_TYPE_FAILOVER_AUTH_REQUEST;  所有M回AUTH_ACK; 若过半票则当主。基于raft实现。 
消息：MEET/PING（每秒随5选1节点或>cluster-node-timeout=15s/2）/PONG/FAIL/PUBLISH;  
clusterMsg{tolen;type;count;epoch;sender[REDIS_CLUSTER_NAMELEN];myslots[REDIS_CLUSTER_SLOTS/8];port;flag;state;data}  
myslots[REDIS_CLUSTER_SLOTS/8]是为了gossip，slots是为了找对应的节点； 

http://www.redis.cn/topics/cluster-tutorial.html
http://www.redis.cn/topics/cluster-spec.html
https://www.cnblogs.com/mengchunchen/p/10059436.html

### 18.发布与订阅
```
struct redisServer { dict *pubsub_channels;  list *pubsub_patterns;};
```
* 服务器状态在pubsub_channels字典保存了所有频道的订阅关系：SUBSCRIBE 命令负责将客户端和被订阅的频道关联到这个字典里面，而UNSUBSCRIBE命令则负责解除客户端和被退订频道之间的关联。
* 服务器状态在pubsub_patterns链表保存了所有模式的订阅关系：PSUBSCRIBE 命令负责将客户端和被订阅的模式记录到这个链表中，而PUNSUBSCRIBE命令则负责移除客户端和被退订模式在链表中的记录。
* PUBLISH命令通过访问pubsub_channels字典来向频道的所有订阅者发送消息，通过访问pubsub_patterns链表来向所有匹配频道的模式的订阅者发送消息。
* PUBSUB命令的三个子命令都是通过读取pubsub_channels字典和pubsub_patterns链表中的信息来实现的。

### 19.事务
```
typedef struct redisClient { multiState mstate;    /* MULTI/EXEC state */ }redisClient
typedef struct multiState {  // 事务队列，FIFO顺序  multiCmd *commands;  // 已入队命令计数  int count;} multiState;
typedef struct redisDb {dict *watched_keys;}
```
* 在事务执行期间，服务器不会中断事务而改去执行其他客户端的命令请求
* client.flags |= REDIS_MULTI
* 当一个客户端切换到事务状态之后, 如果客户端发送的命令为EXEC、DISCARD、WATCH、MULTI四个命令的其中一个，那么服务器立即执行这个命令
* WATCH命令是一个乐观锁（optimistic locking）
* 19.2.2　监视机制的触发: 所有对数据库进行修改的命令，比如SET、LPUSH、SADD、ZREM、DEL、FLUSHDB等等，在执行之后都会调用multi.c/touchWatchKey函数对watched_keys字典进行检查
* 19.2.3　判断事务是否安全: 当服务器接收到一个客户端发来的EXEC命令时，服务器会根据这个客户端是否打开了REDIS_DIRTY_CAS标识来决定是否执行事务
* ACID, 满足A,不支持回滚。 满足CI,不满足D

### 24. 监视器
* 客户端可以通过执行MONITOR命令，将客户端转换成监视器，接收并打印服务器处理的每个命令请求的相关信息。
* 当一个客户端从普通客户端变为监视器时，该客户端的REDIS_MONITOR标识会被打开。
* 服务器将所有监视器都记录在monitors链表中。
* 每次处理命令请求时，服务器都会遍历monitors链表，将相关信息发送给监视器。

### 其他
##### pipeline 与 multi 与 redis-lua 区别

##### bloom实现

##### hyperLogLog
统计1亿个数据的基数值，大约需要内存100000000/8/1024/1024 ≈ 12M; 统计一个对象的基数值需要12M，如果统计10000个对象，就需要将近120G;
https://www.yuque.com/abser/aboutme/nfx0a4

##### reference
https://yq.aliyun.com/articles/531067
https://www.zhihu.com/question/20202931
https://github.com/0voice/backend_video#nav_1_high_performance_network_025
https://news.ycombinator.com/item?id=1171423
