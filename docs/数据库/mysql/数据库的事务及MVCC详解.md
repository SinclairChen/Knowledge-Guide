## 1. 什么是数据库的事务？
事务是数据库管理系统（DBMS）执行过程中的一个逻辑单位，由一个有限的数据库操作序列构成。
从这条定义中我们知道，第一个就是这个最小的一个工作单元，不可分割。第二就是它包含一系列的操作，增删改查。当然，单条的SQL也是有事务的！
最常见的例子就是，转账！小明给小红转100块，转账的过程中涉及到一系列的操作，小明的账户要先进行查询，然后减100块，更新余额。然后小红的账户要加100，在更新余额。这一系列的操作是一体的。不可分割的！
## 2. 在MySQL中哪些存储引擎支持事务
查看[MySQL官方文档](https://dev.mysql.com/doc/refman/5.7/en/storage-engines.html)我们知道Innodb是支持事务的，还有一个就是NDB.
## 3. 事务的四大特性
- A：原子性，Atomicity，我们对数据库的一系列操作，要么都成功要么都失败。不能出现部分成功或者部分失败的情况。
- C：一致性，Consistency，事务执行的前后都是合法的数据状态。
- I：隔离性，Isolation，数据库中会有很多的事务同时去操作同一张表或者同一行数据，他们之间是相互隔离的，互不干扰的。
- D：持久性，Durable，对数据库任务的操作，增删改，当事务提交后，结果是永久性的。
## 4. 事务并发会带来什么样的问题？
1. 脏读：一个事务读到其他事务没有提交的数据
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201227003426343.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDk3NTI5MQ==,size_16,color_FFFFFF,t_70#pic_center)
2. 不可重复读：一个事务读到其他事务已经提交的数据
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020122700364870.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDk3NTI5MQ==,size_16,color_FFFFFF,t_70#pic_center)
3. 幻读：一个事务读到其他事务插入的数据
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201227003708286.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDk3NTI5MQ==,size_16,color_FFFFFF,t_70#pic_center)
## 6. SQL92标准中对4个隔离级别的规定
**什么是SQL92标准？**
就是很多数据库专家联合制定的一个标准，建议数据库厂商都按照这个标准，提供一定的事务隔离级别来解决事务并发的问题。
[SQL92官网](http://www.contrib.andrew.cmu.edu/~shadow/sql/sql1992.txt)
搜索iso可以看到一张表格，里面定义了四个隔离级别。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201227141932557.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDk3NTI5MQ==,size_16,color_FFFFFF,t_70#pic_center)
- p1、p2、p3：分别代表了脏读，不可重复度和幻读。
- possible：代表在这个隔离级别下这个问题有可能发生
- not possible：代表在这个隔离级别下解决了这个问题

然后我们看下这4中隔离级别分别是如何定义的：
1. Read Uncommitted（未提交读）：一个事务可以读取到其他事务没有提交的数据，会出现脏读，简称RU
2. Read Committed（已提交读）：一个事务只能读取到其他事务已经提交的数据，不能读取到其他事务没有提交的数据。他解决了脏读，但是会出现不可重复读的问题，简称RC
3. Repeatable Read（可重复度）：在同一个事务里面多次读取同样的数据结果是一样的，他解决了不可重复读的问题，但是没有解决幻读的问题，简称RR
4. Serializable（串行化）：在这个隔离级别下，所有的事务都是串行执行的，没有并发操作。所以他解决了所有的问题

注：不同的数据库厂商或者存储引擎的实现有一定的差异。

**MySQL Innodb对隔离级别的支持**
Innodb支持的四个隔离级别和SQL92定义的基本一致，隔离级别越高，事务的并发度就越低。唯一的区别在于，Innodb在RR级别就解决了幻读的问题。如图
|  事务的隔离级别  |               脏读                |            不可重复读             |                    幻读                    |
| :--------------: | :-------------------------------: | :-------------------------------: | :----------------------------------------: |
| Read Uncommitted |               可能                |               可能                |                    可能                    |
|  Read Committed  | <font color = 'red'>不可能</font> |               可能                |                    可能                    |
|  Reatable Read   | <font color = 'red'>不可能</font> | <font color = 'red'>不可能</font> | <font color = 'blue'>对Innodb不可能</font> |
|   Serializable   | <font color = 'red'>不可能</font> | <font color = 'red'>不可能</font> |     <font color = 'red'>不可能</font>      |

MySQL默认的事务隔离级别是RR。
## 8. 事务的两大实现方案
### 8.1 LBCC（基于锁的并发控制）
当我要操作数据库的时候，比如说我要查询一条数据，这个时候我就锁定我要操作的那条数据，不允许其他事务来修改他。这种方式就叫做基于锁的并发控制Lock Based Concurrency Control （LBCC）

但是如果是通过这种方式来实现事务的隔离，那么当我读取一条数据的时候，就不允许其他的读写操作，那么大大影响操作的效率。
### 8.2 MVCC（多版本并发控制）
这个时候我们还有另一种解决方案，就是当我操作数据库的时候，比如说我要查询一条数据，我就对这条数建立一个备份或者快照，然后直接读取这个快照。这种方案就叫做多版本并发控制Multi Version Concurrency Control（MVCC）。

为了更好的了解这个机制，我们看一下**Undo Log版本链**。

**什么是Undo Log版本链**
当我们插入一条数据的时候，每条数据其实都有两个隐藏的字段，一个是trx_id（事务id），一个就是roll_pointer（回滚指针）。

举例：现在我们有3个事务A、B、C，我们的事务A插入了一条数据，这个时候这条数据的事务id为50，因为之前没有这条数据，所以回滚指针为空。这个时候事务B来修改了这条数据，此事这个事务id为58，而回滚指针指向了事务A。最后事务C有来修改了这条数据，这个时候事务id为68，回滚指针指向了事务B。如图所示：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201228013950440.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDk3NTI5MQ==,size_16,color_FFFFFF,t_70#pic_center)
还有一个是通过Undo log版本链实现的**ReadView机制**

**什么是ReadView呢？**
就是在你执行一个事务的时候，给你生成一个ReadView，其中包含了4个关键的东西：
1. m_ids：表示此时还有哪些事务在MySQL中没有提交
2. min_trx_id：代表m_ids中最小的值
3. max_trx_id：代表MySQL下一个要生成的事务id，就是最大事务id
4. creator_trx_id：就是当前这个事务id

现在来举例详细说明这个ReadView是如何实现的：

1. 假设我们现在数据库中已经插入过了一条数据，事务id为32。然后现在有两个事务A，B来同时操作这条数据。事务A（id=45）来查询这条数据，事务B（id=59）来修改这条数据。如图所示：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201228160849708.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDk3NTI5MQ==,size_16,color_FFFFFF,t_70)
2. 事务A会直接开启一个ReadView，其中m_ids中包含了事务A和事务B的两个事务id，45和59，min_trx_id就是45，max_trx_id就是下一个要生成的事务id，现在最大的是59，所以他是60，最后一个就是creator_id是45，就是创建这个ReadView的事务A。

3. 事务A第一次查询这条数据的时候会先进行判断，就是判断当前这条数据的事务id是否小于min_trx_id，当发现当前的这个事务id（32）小于最小的那个事务id（45），就说明他在你之前这个事务已经提交了，你可以查到这条数据。

4. 现在事务B开始修改这条数据，然后这行数据的trx_id就设置成了事务B的事务id（59），同时roll_pointer指向了修改之前生成的一个undo log，接着这个事务提交了。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201228161209214.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDk3NTI5MQ==,size_16,color_FFFFFF,t_70)
5. 接着事务A又来查询了一次，发现此时这条数据的trx_id（59）是大于readview中最小的那个事务id的（45）,同时又小于ReadView里面的max_trx_id（60），这时候判断这条数据的事务id是否在m_ids中，如果在那么就不能查。接着会按照这条数据的roll_pointer往下找，查看之前的那条数据的trx_id是否小于ReadView中最小的min_trx_id，发现小于那么就可以查到了
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201228162019539.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDk3NTI5MQ==,size_16,color_FFFFFF,t_70)
 6. 如果说现在事务A这个时候来修改这条数据，那么就会生成一个快照。同时这条数据的事务id改成事务A的id也就是45。修改完成后，事务A再来查看这条数据，会发现这条数据的事务id和creator_id是一样的，就可以直接查到。
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020122816334234.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDk3NTI5MQ==,size_16,color_FFFFFF,t_70)
7. 最后来了一个事务C（id=76），来修改这条数据，也会生成一个快照。然后我们的事务A又来查询了，这个时候发现这条数据的事务id已经大于自己ReadView中的max_trx_id了，说明在自己这个事务之后又有一个事务来修改了这条数据，自己就不能查到，就要顺这roll_pointer往下找，找到自己修改过的那个版本。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201228164455400.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDk3NTI5MQ==,size_16,color_FFFFFF,t_70)

到这，我相信我们现在知道MySQL中的MVCC其实就是基于undo log版本链和ReadView机制来实现的。