# RedisMemoryTree
Redis底层数据存储技术研究


###jemalloc size class categories,左边是用户申请内存范围,右边是实际申请的内存大小.

![](https://i.imgur.com/gQ9EiSe.png)

###String
####一个set hello world命令最终(中间会malloc,free的我们不考虑)会产生4个对象,一个dictEntry(12字节),一个sds用于存储key,还有一个redisObject(12字节),还有一个存储string的sds.sds对象除了包含字符串本生之外,还有一个sds header和额外的一个字节作为字符串结尾共9个字节

根据内存申请对照表，一共申请64字节。

![](https://i.imgur.com/gkWITqc.png)

###Number类型

####sds被释放了,数字被存储在指针位上,所以对于set hello 1111111就只需要48字节的内存

###调整REDIS_SHARED_INTEGERS
####如果value数字小于宏REDIS_SHARED_INTEGERS(默认10000),则这个redisObject也都节省了,使用share Object，Redis程序启动时直接创建10000个RedisObject,代表 1-10000的整形。

####这样一个set hello 111就只需要32字节,连redisObject也省了.所以对于value都是小数字的应用,适当调大REDIS_SHARED_INTEGERS这个宏可以很好的节约内存

<pre>
Redis的过期机制
      Redis为了不影响正常工作，一般只会在必要或CPU空闲的时候做过期清理的动作。
      必要：
          访问Key的时候
      CPU空闲：
          系统空闲时做后台定期任务，时间限制为25%的CPU时间。
          若达到了时间限制，退出清理。

      淘汰策略：
          volatile-lru：从已设置过期时间的数据中挑选最近最少使用的数据淘汰。
          volatile-ttl：从已设置过期时间的数据中挑选将要过期的数据淘汰
          volatile-random:从已设置过期时间的数据集中任意选择数据淘汰
          allkeys-lru:从数据集中挑选最近最少使用的数据淘汰
          allkeys-random：随机淘汰数据
          no-eviction:禁止淘汰数据
</pre>

<pre>
Redis中的BitMap
</pre>

<pre>
String 
     K/V结构
     底层：
         实现方式：
                String在Redis内部存储默认就是一个字符串，被RedisObject所引用，当遇到
         incr, decr等操作时会转成数值型进行计算，此时RedisObject的encoding字段为int
</pre>

<pre>
Hash 
     Redis 的 Hash 实际是内部存储的 Value 为一个 HashMap

     Redis Hash 对应 Value 内部实际就是一个 HashMap，实际这里会有2种不同实现，这个 
     Hash 的成员比较少时 Redis 为了节省内存会采用类似一维数组的方式来紧凑存储，而不会采
     用真正的 HashMap 结构，对应的 value redisObject 的 encoding 为 zipmap，当成员
     数量增大时会自动转成真正的 HashMap，此时 encoding 为 ht
</pre>

<pre>
List
     实现方式：
     Redis list 的实现为一个双向链表，即可以支持反向查找和遍历，更方便操作，不过带来了部分额外的内存开销，Redis 内部的很多实现，包括发送缓冲队列等也都是用的这个数据结构
</pre>

<pre>
Set
     Redis set 对外提供的功能与 list 类似是一个列表的功能，特殊之处在于 set 是可以自动
     排重的，当你需要存储一个列表数据，又不希望出现重复数据时，set 是一个很好的选择，并
     且 set 提供了判断某个成员是否在一个 set 集合内的重要接口，这个也是 list 所不能提供的

     实现方式：
     set 的内部实现是一个 value 永远为 null 的 HashMap，实际就是通过计算 hash 的方式
     来快速排重的，这也是 set 能提供判断一个成员是否在集合内的原因
</pre>

<pre>
Sorted set
     实现方式：
     Redis sorted set 的内部使用 HashMap 和跳跃表（SkipList）来保证数据的存储和有
     序，HashMap 里放的是成员到 score 的映射，而跳跃表里存放的是所有的成员，排序依据
     是 HashMap 里存的 score，使用跳跃表的结构可以获得比较高的查找效率，并且在实现上比较简单
</pre>

<pre>
Redis的持久化机制
     1）定时快照方式 (snapshot)
     2）基于语句追加文件的方式 (aof)
     3) 虚拟内存
     5）Diskstore方式

     在设计思路上。前两种是基于全部数据都在内存中，即小数据量下提供磁盘落地功能，而后两种
     方式则是作者在尝试存储数据超过物理内存时，即大数据量的数据存储，当前后两种方式仍在
     实验阶段，并且VM已经被作者放弃。换句话说，Redis目前还只能作为小数据量存储（全部数据
     加载在内存中），海量数据存储方面并不是Redis锁擅长的领域。

     定时快照方案：
         该持久化方式实际上是在Redis内部一个定时器事件，每隔固定时间去检查当前数据发生的
     改变次数与时间是否满足配置的持久化触发的条件，如果满足则通过操作系统fork调用来创建
     一个子进程，这个子进程默认会与父进程共享相同的地址空间，这时就可以通过子进程来遍历整个
     内存来进行存储操作，而主进程则任然可以提供服务，当有写入时由操作系统按照内存页为单位
     进行copy_on_write保证父子进程不会相互影响。
     
         该持久化的主要缺点是定时快照只是代表一段时间内的内存映像，所以系统重启会丢失上次快
     照与重启之间所有的数据。


     基于语句追加方式
         aof方式实际类似于mysql基于语句的binlog方式，即每条会使Redis内存数据发生改变的命
     令都会追加到一个log文件中，也就是说这个log文件就是Redis的持久化数据。

         aof的主要缺点是追加log文件可能导致体积过大，当系统重启恢复数据时,加载数据会非常慢
     慢，几十G的数据可能需要几小时才能加载完成，当然这个耗时并不是因为磁盘文件读取速度慢，
     而是由于读取的所有命令都要在内存中执行一遍。另外由于每条命令都要写log，所以使用aof的
     方式，Redis的读写性能也会有所下降。

  
     Disstore方式
         作者放弃虚拟内存方式之后的一种新的实现方式，也就是传统的B-树方式，目前仍在实验阶段。
</pre>

<pre>
Redis持久化磁盘IO方式及其带来的问题
 
     
</pre>

