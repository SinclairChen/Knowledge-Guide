## 1. Lua脚本

​	Lua是一种轻量级脚本语言，它是用C语言编写的，跟数据的存储过程有点类似。



​	使用Lua脚本来执行Redis命令的好处：

1. 一次发送多个命令，可以减少网络开销
2. Redis会将整个脚本作为一个整体执行，不会被其他请求打断，保证了原子性
3. 对于复杂的组合命令，我们可以放在文件中，可以实现复用



## 2. 在Redis中调用Lua脚本

​	使用`eval`方法，语法格式：

```
redis>eval lua-script key-num [key1 key2 key3 ....] [value1 value2 value3 ....]
```

- eval：代表执行lua语言的命令
- lua-script：代表Lua脚本内容
- key-num：表示参数中有多少个key。**注意：Redis中的key是从1开始的，没有可以写0**
- [key1 key2 key3 ....]：表示key作为参数传给Lua脚本，也可以不填。需要和key-num对应
- [value1 value2 value3 ....]：表示将值参数传给lua脚本，也可以不填



示例：返回一个字符串，0个参数

```
redis>eval "return 'hello world'" 0
```



## 3. 在Lua脚本中调用Redis命令

​	使用`redis.call(command, key [param1, param2 ...])`进行操作，具体格式：

```
redis> eval "redis.call('set',KEYS[1],ARGV[1])" 1 lua-key lua-value
```

- command：命令，包括set、get、del等
- key：被操作的键
- param1、param2：key的值



### 3.1 设置键值对

​	在redis中调用Lua脚本执行Redis命令

```
redis>eval "return redis.call('set', key[1], argv[1])" 1 k1 666
redis>get k1
```

​	以上命令，等价于`set k1 666`



### 3.2 在redis中调用Lua脚本文件

​	1、创建Lua脚本文件

```
cd /usr/local/redis/src
vim demo.lua
```

​	

​	2、编写脚本内容

```lua
redis.call('set', 'k1', '666')
return redis.caluu('get', 'k1')
```



​	3、在redis客户端调用lua脚本

```
cd /usr/local/redis/src
redis-ci --eval demo.lua 0
```



#### 3.2.1 案例：对IP进行限流

​	需求：在x秒内只能访问y次

​	设计思路：用ip作为key，用次数作为value

​	拿到ip以后，如果是第一次访问，对key设置过期时间，否者判断次数。超过限定次数，返回0。没有超过次数，返回1。

```lua
--ip_limit.lua
-- ip限流，对某个ip访问评率进行限制，6秒钟访问10次
-- keys[1]：ip	argv[1]：过期时间	argv[2]：访问次数
local num=redis.call('incr',KEYS[1])
if tonumber(num)==1 then
    redis.call('expire', KEYS[1],ARGV[1])
    return 1
    else if tonumber(num)>tonumber(ARGV[2]) then
        return 0
    else
        return 1
end
```



​	调用测试：

```
./redis-cli --eval "ip_limit.lua" app:ip:limit:192.168.1.111 , 6 10
```

- app:ip:limit:192.168.1.111：key，后面是参数值，中间需要加上一个空格，一个逗号，一个空格
- 多个参数之间用空格分割



## 4. 缓存Lua脚本

### 4.1 为什么要缓存脚本

​	因为每次调用脚本都需要将整个脚本传给redis服务端，这会产生较大的网络开销。为了解决这个问题，redis提供了`evalsha`命令，允许开发者通过脚本内容sha1摘要来执行脚本



### 4.2 如何缓存

​	Reids在执行script load命令时会计算脚本sha1摘要并记录在加班呢缓存中，执行evalsha命令时Redis会根据摘要从脚本缓存中查找对应的脚本内容，如果找到了则执行脚本，否者会返回错误：”NOSCRIPT No matching script. Please use EVAL.“

```
127.0.0.1:6379>script load "return 'hello world'"
"470877a599ac74fbfda41caa908de682c5fc7d4b"
127.0.0.1:6379>evalsha "470877a599ac74fbfda41caa908de682c5fc7d4b" 0
"hello world"
```



#### 4.2.1 案例：自乘

​	Redis中有`incrby`这样的自增命令，但是没有自乘，比如乘以3，或者乘以5

​	我们可以写一个自乘的运算，让它乘以后面的参数

```lua
local curVal=redis.call("get", KEYS[1])
if curVal == false then
	curVal = 0
else
    curVal = tonumber(curVal)
end
curVal = curVal * tonumber(ARGV[1])
redis.call('set',KEYS[1], curVal)
return curVal
```



​	把上面这个脚本变成单行，语句之间使用`；`隔开

```
local curVal = redis.call("get", KEYS[1]); if curVal == false then curVal = 0 else curVal = tonumber(curVal) end; curVal = curVal * tonumber(ARGV[1]); redis.call("set", KEYS[1], curVal); return curVal
```



​	然后使用`script load`上面的命令

```
127.0.0.1:6379>script load 'local curVal = redis.call("get", KEYS[1]); if curVal == false then curVal = 0 else curVal = tonumber(curVal) end; curVal = curVal * tonumber(ARGV[1]); redis.call("set", KEYS[1], curVal); return curVal'
"be4f93d8a5379e5e5b768a74e77c8a4eb0434441"
```



​	测试调用：

```
127.0.0.1:6379>set num 2
OK
127.0.0.1:6379>evalsha “be4f93d8a5379e5e5b768a74e77c8a4eb0434441" 1 num 6
(integer)12
```



## 5. 脚本超时

​	Redis的指令是以单线程来执行的，这个线程还要执行lua脚本呢，如果lua脚本执行超时或者陷入死循环，是不是没有办法给客户端提供服务了？

​	为了防止某个脚本执行时间过长导致Redis无法提供服务，Redis提供了`lua-time-limit`这个参数来限制脚本的最长运行时间，默认为5秒：

​	redis.conf:

```
lua-time-limit 5
```



​	当脚本的运行超过这个限制时间，Redis开始接受其他命令但不会执行（为了保证脚本的原子性，脚本其实没有没终止），而是返回一个"BUSY"的错误。

​	Redis提供了一个`script kill`命令来终止脚本的执行。新开一个客户端：

```
script kill
```

​	

​	如果当前执行的Lua脚本对Redis的数据进行了修改（set， del等），那么`script kill`命令是不能终止脚本运行的

```
127.0.0.1:6379>eval redis.call('set', 'k1', '666') while true do end" 0
127.0.0.1:6379>script kill
(error) UNKILLABLE Sorry the script already executed write commands against the dataset. You can either wait the script termination or kill the server in a hard way using the SHUTDOWN NOSAVE command.
```

​	因为要保证脚本运行的原子性，如果脚本执行了一部分终止了，那就违背了脚本原子性的要求。



​	遇到这种情况，只能通过`shutdown nosave`命令来强制终止Redis

​	

​	**shutdown nosave和shutdown的区别：**

- shutdown nosave：不会进行持久化操作，意味着发生在上一次快照后的数据都会丢失