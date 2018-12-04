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

