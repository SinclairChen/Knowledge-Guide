​	在redis中所有的数据都是存储在内存中的，在某些情况下需要对占用的内存空间进行回收。主要有两类：

1. key过期
2. 内存使用达到上限



## 1. 过期策略

### 1.1 定时过期

​	每个设置过期时间key都需要创建一个定时器，到过期时间就会立即清除。

​	好处：立即清除过期的数据，对内存友好

​	坏处：占用大量cpu资源处理过期数据，从而影响影响时间和吞吐量



### 1.2 惰性过期	

​	只有当访问key的时候，才会判断这个key是否已经过期，如果过期了就清除。

​	好处：节省了cpu资源

​	坏处：对内存不友好



### 1.3 定期过期

​	每隔一段时间，会扫描数据库中expires字典中一定数量的key，并清除其中已经过期的key。这个策略是上上面两种的一个折中的方案。通过调整定时扫描的时间间隔和每次扫描的限定耗时，可以在不同的情况下是的cpu和内存资源达到最优的平衡效果



Redis中同时使用惰性过期和定期过期两种过期策略。



## 2. 内存淘汰

​	如果都不过期，Redis内存满了怎么办？

​	redis中的内存淘汰策略，当内存使用达到最大内存极限时，需要使用内存淘汰算法来决定清理掉哪些数据，保证新的数据的存入。



### 2.1 最大内存设置

1. redis.conf:

```
#maxmemory <bytes>
```

​	如果不设置maxmemory或者设置为0，64位系统不限制内存，32位系统最多使用3GB。



2. 动态修改

```
config set maxmemory 2GB
```



### 2.2 内存淘汰策略

1. **配置**

redis.conf

```
#maxmemory-policy noeviction
```



2. **分类：**

| 策略            | 含义                                                         |
| :-------------- | ------------------------------------------------------------ |
| volatile-lru    | 根据lru算法删除设置了超时属性的键值，直到腾出足够内存位置。如果没有可删除的键对象，回退到noeviction策略 |
| allkeys-lru     | 根据lru算法删除键，不过数据有没有设置超时属性，直到腾出足够内存位置 |
| volatile-lfu    | 在带有过期时间中的键中选择不常用的                           |
| allkeys-lfu     | 在所有的键中选择最不常用的，不管数据有没有设置超时时间       |
| volatile-random | 随机删除带有过期时间的键                                     |
| allkeys-random  | 随机删除所有的键                                             |
| volatile-ttl    | 根据键值对象的ttl属性，删除最近将要过期的数据，如果没有退回到noeviction策略。 |
| noeviction      | 默认策略。不会删除任何数据，拒绝所有写入请求并返回客户端错误信息（err）OOM。此时redis只能响应读操作。 |



#### 2.2.1 LRU淘汰原理

 	Redis LRU对传统的LRU算法进行了改良，通过随机采用来调整算法的精度。

​	如果淘汰策略是LRU，根据配置的采样值maxmemory_samples(默认是5个)，随机中数据库中选择m个key，淘汰其中热度最低的key对饮的缓存数据。所以采样数m配置的数值越大，就越能精确的查找到带淘汰的缓存数据，但是也会消耗更多的cpu计算，执行效率降低

​	

​	**如何找出热度最低的数据？**

​	reids中所有的对象结构都有一个lru字段，且使用了unsigned的低24位，这个字段用来记录对象的热度。对象被创建的时候会记录lru值。在被访问的时候会更新lru的值，但是不是获取系统当前的时间戳，二是设置为全局变量`server.lruclock`的值

​	源码：server.h

```
typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:LRU_BITS; 	/* LRU time  (relative  to global lru_clock) or
    						* LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time). */
    int refcount; void *ptr;
} robj;
```



​	**server.lruclock的值是怎么来的？**

​	redis中有个定时处理的函数serverCron，默认每100ms调用函数updateCachedTime更新一次全局变量server.lruclock的值，它记录的是当期那unix时间戳

​	源码：server.c

```
void updateCachedTime(void) {
	time_t unixtime  = time(NULL);
    atomicSet(server.unixtime,unixtime);
    server.mstime  = mstime();
    
	struct tm tm;
	localtime_r(&server.unixtime,&tm);
	server.daylight_active  = tm.tm_isdst;
}
```



​	**为什么不获取精确的时间而是放在全局变量中？不会有延迟的问题吗？**

​	这样函数lockupkey中更新数据lru热度值时，就不用每次调用系统函数time，可以提高执行效率。



​	**如何评估对象的热度？**

​	函数estimateObjectIdleTime评估指定对象的lru热度，思想就是对象来的lru值和全局的server.lruclock的差值越大，该对象热度越低。

​	源码：evict.c

```
unsigned long long estimateObjectIdleTime(robj *o)  { 
	unsigned long long lruclock = LRU_CLOCK();
    if (lruclock >=  o->lru) {
		return (lruclock - o->lru) * LRU_CLOCK_RESOLUTION;
    } else {
		return (lruclock + (LRU_CLOCK_MAX -  o->lru)) * LRU_CLOCK_RESOLUTION;
	}
}
```

​	server.lruclock只有24位，按秒位单位来表示才能存储194天。当超过24bit能表示的最大时间的时候，他会从头开始计算。

​	源码：server.h

```
#define LRU_CLOCK_MAX ((1<<LRU_BITS)-1)  /* Max value of obj->lru */
```

​	在这种情况下，可能会出现对象的lru大于server.lruclock的情况，那么就是两个相加而不是两个相减来求最久的key



​	**为什么不适用传统的LRU算法？**

1. 需要使用额外的内存结构消耗内存
2. 假设A在10秒内被访问的5次，B在10秒内被访问3次。但是B的访问时间要比A的访问时间晚，这种情况反而会导致A被回收。



#### 2.2.2 LFU

```
typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:LRU_BITS; 		/* LRU time  (relative  to global lru_clock) or
    							* LFU data (least significant 8 bits frequency 
    							* and most significant 16 bits access time). */
    int refcount;
    void *ptr; 
} robj; 
```

当着24bits用作LFU时，会被分成两个部分：

- 高16位用来记录访问的时间（单位位分钟，ldt， last decrement time）
- 低8位用来记录访问的频率，简称counter（logc，logistic counter）

counter是用基于对数计数器实现的，8位可以表示百万次的访问频率



​	对象读写的时候，lfu的值会被更新。

​	源码：db.c——lookupke

```
void updateLFU(robj *val)  {
	unsigned long counter =  LFUDecrAndReturn(val);
    counter =  LFULogIncr(counter);
	val->lru = (LFUGetTimeInMinutes()<<8) | counter; 
}
```



​	增长速率由，`lfu-log-factor`来决定，值越大，counter增长的越慢

​	redis.conf配置文件：

```
# lfu-log-factor 10
```



​	**计数器如何减少？**

​	通过衰减因子`lfu-decay-time`来控制，如果值是1的话，N分钟没有访问就要减少N

​	redis.conf配置文件

```
# lfu-decay-time 1
```



​	