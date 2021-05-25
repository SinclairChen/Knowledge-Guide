## 1. 为什么要用事务

​	我们知道Redis的的那个命令是原子性的，如果涉及到多个命令的时候，需要把多个命令作为一个不可分割的来处理，就需要用到事务。

​	Redis的事务有两个特点：

	1. 按进入队列的顺序执行
 	2. 不会受到其他客户端请求的影响



​	Redis事务的四个命令：具体查看[官方文档](https://redis.io/topics/transactions/)和[Redis命令参考文档](http://redisdoc.com/topic/transaction.html)

	- multi：开启事务
	- exec：执行事务
	- discard：取消事务
	- watch：监控



## 2. 事务的用法

案例：tom和mic各有1000元，tom向mic转账100元，tom账户余额减少100元，mic账户余额增加100元

```
127.0.0.1:6379> set tom 1000
OK
127.0.0.1:6379> set mic 1000
OK
127.0.0.1:6379> multi
OK
127.0.0.1:6379> decrby tom 100 QUEUED
127.0.0.1:6379> incrby mic  100 QUEUED
127.0.0.1:6379> exec
1) (integer)  900
2) (integer)  1100
127.0.0.1:6379> get tom
"900"
127.0.0.1:6379> get mic
"1100"
```



​	**流程：**

1. 使用multi开启事务。**注意：事务不能嵌套，多个multi命令效果一样**
2. 开启multi后，客户端可以向服务器发送任意多条命令，这些命令不会立即执行，而是会被放到一个队列中
3. 执行exec命令，队列中所有的命令被执行，如果没有执行exec，所有命令都不会被执行



​	**如果中途不想执行事务了，可以通过discard清空队列，取消事务**

```
multi
set k1 1
set k2 2
set k3 3
discard
```



## 3. watch命令

​	watch命令可以为Redis事务提供CAS乐观锁的行为，也就是多个线程更新变量的时候，会和原来的值进行比较，只有它没有被其他线程修改的情况下，才会更新成功。

​	我们可以通过watch监视一个或者多个key。**注意：如果开启事务之后，至少有一个被监视key键在exec执行之前被修改了，那么整个事务都会被取消。（key提前过期除外）**

​	

| client1                                                      | client2                                                |
| ------------------------------------------------------------ | ------------------------------------------------------ |
| 127.0.0.1:6379> set balance 1000<br/>OK<br/>127.0.0.1:6379> watch balance<br/>OK<br/>127.0.0.1:6379> multi<br/>OK<br/>127.0.0.1:6379> incrby balance 100<br/>QUEUED |                                                        |
|                                                              | 127.0.0.1:6379> decrby balance 100<br /> (integer) 900 |
| 127.0.0.1:6379> exec<br /> (nil)<br/>127.0.0.1:6379> get balance<br/>"900" |                                                        |



## 4. 使用事务可能会遇到的问题

​	Redis事务执行遇到的问题分为两种：

1. 在执行exec之前发生错误
2. 在执行exec之后发生的错误

### 4.1 在执行exec之前发生错误

​	比如：入队的命令存在语法错误，包括参数数量，参数名等等（编译器错误）

```
127.0.0.1:6379> multi
OK
127.0.0.1:6379> set k1 666
QUEUED
127.0.0.1:6379> hset k2 2673
(error) ERR wrong number  of  arguments for 'hset' command
127.0.0.1:6379> exec
(error) EXECABORT Transaction discarded because of previous errors.
```



### 4.2 在执行exec之后发生的错误

​	比如：类型错误（对string类型使用hash的命令），这是一种运行时的错误

```
127.0.0.1:6379> flushall
OK
127.0.0.1:6379> multi
OK
127.0.0.1:6379> set k1 1 QUEUED
127.0.0.1:6379> hset k1 a  b QUEUED
127.0.0.1:6379> exec
1) OK
2) (error) WRONGTYPE Operation against a key holding the wrong kind of value
127.0.0.1:6379> get  k1
"1"
```

​	我们发现这里set命令执行成功了，但是hset命令没有执行成功。也就是只有错误的命令没有被执行，但是其他命令没有受到影响。

​	这就不符合原子性了，也就是我们没有办法使用Redis的这种事务机制来保证数据的一致性。

