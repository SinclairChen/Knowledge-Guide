## 前言
在数据库里面，我们说的update操作其实是包括了更新、插入和删除。如果我们查看过MyBatis中的源码，我们会发现Executor中只有doQuery和doUpdate方法啊，没有doDelete和doInsert方法。

**更新流程和查询流程有什么不同呢？**
基本流程是一致的，它也是要经过分析器，优化器，最后交给执行器处理。区别在于拿到符合条件数据之后的操作。

啥也不说，先上图
**Innodb内存结构和磁盘结构图：**
![Innodb内存结构和磁盘结构](https://img-blog.csdnimg.cn/20201221162318506.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDk3NTI5MQ==,size_16,color_FFFFFF,t_70#pic_center)
**更新流程图：**
![update流程图](https://img-blog.csdnimg.cn/20201221163022842.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDk3NTI5MQ==,size_16,color_FFFFFF,t_70)

## 1. Buffer Pool（缓冲池）
首先，Innodb的数据都是放在磁盘上的，如果直接操作磁盘的话，那速度太慢了。所以Innodb提供了一个缓冲池的技术，就把数据通过缓冲池进行交互。当我们进行增删改的时候需要先看下缓冲池中有没有，如果有直接修改，如果没有还需要将数据从磁盘中读到缓冲池中。然后在这个缓冲池中进行修改。
Buffer Pool主要分为3个部分：buffer pool、change buffer、adaptive hash index，另外还有一个redo log buffer

#### 1.1 Buffer Pool
现在我们知道我们修改数据是要经过buffer pool的，那么数据是以什么方式加载到这个缓冲池中的呢？

是以**数据页**的形式加载到缓冲区，加载到缓冲区的为**缓存页**。那什么是数据页什么是缓存页呢?
- 数据页：就是MySQL中抽象出来的数据单位，他是把很多行数据放在了一个数据页里面，也就是说我们的磁盘文件中会有很多的数据页，每一页都放了很多行的数据。Innodb数据页的大小为16KB。
- 缓存页：buffer pool中存放的数据页，我们叫做缓存页。每一个缓存也都有一个自己的描述信息，包含这个数据也对应的表空间，数据页的编号，这个缓存也在buffer pool中的地址等等。这个描述信息本身也是一块数据。

具体大概长这个样子：
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020122418281433.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDk3NTI5MQ==,size_16,color_FFFFFF,t_70)
**查看buffer pool的相关信息** 
```sql
show status like '%innodb_buffer_pool%'
```
具体含义可以查看[官方文档](https://dev.mysql.com/doc/refman/5.7/en/server-status-variables.html)
同时我也知道Buffer Pool默认大小128M，这个是可以调整的。

**查看buffer_pool系统参数**
```sql
show variables like '%innodb_buffer_pool%'
```
具体含义也可以查看[官方文档](https://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html)

### 1.1.1 内存淘汰策略
上述我们知道Buffer Pool的默认大小是128M，那么如果缓冲池写满了怎么办？

Innodb是通过LRU算法来管理缓冲池的。

**那么普通的LRU算法能不能直接使用呢？会带来什么样的问题？**
1. 第一种情况：在MySQL中有一个预读机制，当你从磁盘加载一个数据页的时候，他可能会将这个数据也的相邻的其他数据页都加载到缓存中。
如图所示：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201226231103862.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDk3NTI5MQ==,size_16,color_FFFFFF,t_70#pic_center)
当我们进行淘汰的时候尾部的，被人访问的数据缓存也被淘汰了。而没有被人访问的却还是链表中。

2. 第二种情况：全表扫描。比如说：
select * from table1;
这样的语句会将表中所有的数据都加载到缓存中，后续却不在访问这些数据。而在之前经常会访问的数据会被推到LRU链表的尾部，当需要刷盘的时候却将被访问被访问的数据刷到了磁盘，不被访问的数据却还在LRU链表中。

所以MySQL对LRU进行了优化，将其拆分为两部分：一部分是热数据，一部分是冷数据。比例由innodb_old_blocks_pct这个参数控制，默认为37，也就是表示冷数据占比37%
如图所示：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201226232247718.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDk3NTI5MQ==,size_16,color_FFFFFF,t_70#pic_center)
1.当我们加载数据的时候会加载到冷数据区的头部
2. 如果经常被访问的数据会往热数据区移动
3. 不经常访问的，慢慢的往冷数据区移动
4. 当刷盘的时候，直接将冷数据区尾部的数据通过一个后台线程定时的刷到磁盘中

### 1.2 change buffer 写缓冲
如果这个数据也不是唯一索引，不存在数据重复的情况，也就不要从磁盘加载索引页判断数据是不是重复。这种情况下可以先把修改记录在内存的缓冲池中，从而提升更新语句的执行速度。这个区域就是Change Buffer。最后再把Change Buffer记录到数据页，这个操作叫做merge。

那什么时候发生merge？
1. 在访问这个数据页的时候
2. 通过后台线程时候
3. 数据库shut down的时候
4. redo log写满的时候

如果数据库大部分索引都是非唯一索引，并且业务是写多读少，不会在写数据后立刻读取。就可以使用change buffer。同时可以调大它的大小：
```sql
show variables like 'innodb_change_buffer_max_size';
```
这个值代表的是change buffer占buffer pool的比例，默认25%。

### 1.3Adaptive Hash Index（自适应哈希索引）
Adaptive Hash Index是针对B+树Search Path的优化。我们知道hash是一种非常快的查找方式，查找的时间复杂度为O（1）。所以Innodb提出了一种实现方式。

Innodb引擎会监控所有对表上的查找，如果观察到可以建立哈希索引能提高速度，就会建立hash索引。自适应hash索引是在B+树的基础上构建的，而且不会将整张表都建立成自适应hash索引，是根据访问的频率和模式来为某些页建立的。

根据官方文档显示，启用自适应hash索引后，读取和写入的速度可以提高2倍，

### 1.4 redo log buffer
redo log日志（详情请看下面）不是每次都是直接写入磁盘的，在内存中有一块区域专门用来保存即将要写入日志文件的数据，默认为16M，它的作用是为了节省磁盘I/O。

**查看大小**
```sql
show variables like 'innodb_log_buffer_size';
```

**那么buffer中的数据什么时候刷入磁盘文件呢？**
log buffer写入磁盘的时机，由一个参数控制，默认是1.
```sql
show variables like 'innodb_flush_log_at_trx_commit';
```
有3个值可供修改：
| 值                  | 含义                                                         |
| :------------------ | :----------------------------------------------------------- |
| 0（延迟写）         | log buffer 将每秒一次地写入log file中，并且log file的flush操作同时进行。该模式下，在事务提交的时候，不会主动触发写入磁盘操作 |
| 1（实时写，实时刷） | 每次事务提交MySQL都会吧log buffer的数据写入log file，并且刷到磁盘中去 |
| 2（实时写，延迟刷） | 每次事务提交时MySQL都会把log buffer的数据写入log file，但是flush操作并不会同时进行。该模式下，MySQL会每秒执行一次flush操作。 |

## 2. 磁盘结构
### 2.1 系统表空间 system tablespace
在默认情况下InnoDB存储引擎有一个共享表空间(对应文件/var/lib/msql/ibdata1)，叫系统表空间。其中包含Innodb数据字典、双写缓冲区、change buffer 和undo log，如果没有指定file-per-table，也包含用户创建的表和索引数据。

#### 2.1.1 数据字典
由内部系统表组成，存储表和索引的元数据（定义信息）。

#### 2.1.2 doublewrite buffer（双写缓冲）
我们知道innodb数据页的大小是16kb，而操作系统页的大小为4kb，当写数据页的时候是4k，4k的写。如图所示：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201226204740827.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDk3NTI5MQ==,size_16,color_FFFFFF,t_70#pic_center)
如果在写到一半的时候，比如写了8k，数据服务宕机了。数据页只写了一部分，这个时候数据是不完整的。这种情况叫做部分写失效（parital page write）。为了解决这个问题，innodb设计了一个双写缓冲区来解决这个问题。

当进行刷脏的时候，会写两遍到磁盘上，第一遍是写到doublewrite buffer，第二遍是从doublewrite buffer写到真正的数据文件中。如果发生了宕机等问题的时候，InnoDB再次启动后，发现了其中一个数据页已经损坏或者不完整，那么此时就可以从doublewrite buffer中进行数据恢复了。

#### 2.1.3 undo logs（回滚日志）
  undo log（撤销日志或回滚日志）记录了事务发生之前的数据状态（不包括select）。如果修改数据是出现异常，可以用undo log来实现回滚操作，来保证原子性。数据默认存在系统表空间ibdata1文件中。

  在执行undo log的时候，仅仅是将数据从逻辑上恢复至事务之前的状态，而不是从物理页面上操作实现，适于逻辑格式的日志。

redo log 和 undo log 统称为事务日志。

### 2.2 独占表空间 file-per-tablespaces
可以让每张表独占一个表空间，这个设置默认是开启的。
```sql
show variables like 'innodb_file_per_table';
```
每张表都会开辟一个表空间，这个文件就是数据目录下的ibd文件，用来存放表的索引和数据。

### 2.3 通用表空间 general tablespaces
通用表空间也是一种共享的表空间，和ibdata1类似。

可以创建一个通用的表空间，用来存储不同数据库的表，数据路径和文件都可以自定义。
```sql
create tablespace gts2673 add datafile '/var/lib/mysql/gts2673.ibd' file_block_size=16k engine=innodb;
```
然后当我们创建表的时候可以指定表空间
```sql
create table table1 tablespace gts2673;
```

删除表空间，需要先删除里面所有的表：
```sql
drop table table1;
drop tablespace gts2673;
```

### 2.4 临时表空间 temporary tablespaces
用来存储临时表的数据，包括用户创建的临时表和磁盘内部临时表，对应数据目录下的ibtmp1文件。

当数据服务器正常关闭时，该表空间被删除，下次重新产生。

### 2.5 redo log（重做日志）
1. 如果buffer pool里面的脏页还没有刷入磁盘的时候，数据库宕机后者重启，这些数据就会丢失。如果写操作写到一半，甚至可能会破坏数据文件导致数据不可用。为了避免这个问题，Innodb把所有对页的修改操作专门写入一个日志文件，并且在数据库启动的时候从这个文件进行恢复操作（crash-safe）来保证事务的持久性。
2. 刷盘是随机I/O，而记录日志是顺序I/O，顺序I/O的效率会比随机I/O的效率要高，因此先把数据写到日志中，在延迟刷盘的时机，可以提高吞吐量。

这种日志和磁盘配合的过程，就是MySQL中的WAL技术（Write-Ahead Logging），它的关键点在于先写日志，再写磁盘。

**查看redo log参数**
```sql
show variables like 'innodb_log%';
```
![redo log参数](https://img-blog.csdnimg.cn/20201226193835391.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDk3NTI5MQ==,size_16,color_FFFFFF,t_70#pic_center)
| 值                        | 含义                                                  |
| :------------------------ | :---------------------------------------------------- |
| innodb_log_file_size      | 指定每个文件的大小，默认48M                           |
| innodb_log_files_in_group | 指定文件的数量，默认为2                               |
| innodb_log_group_home_dir | 指定文件所在路径，相对或绝对。如果不指定，则为datadir |

Innodb的redo log是固定大小的，比如可以配置一组4个文件，每个文件的大小是1Gb，公共就是可以记录4GB的大小，如果写完了那么就会从头开始重新写。具体可以如图所示（来自极客时间-MySQL实战45讲）：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201226202404907.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDk3NTI5MQ==,size_16,color_FFFFFF,t_70)
- write pos是当前记录的位置，一边写一边往后移动，写到第3个文件的末尾就会回到0号文件的开头。
- check point是当前要擦除的位置，也是往后移并且循环，擦除记录前要把记录更新到数据文件中。
- write pos和check point之间的空着的部分，可以用来记录新的操作。如果write pos追上了check point，这个时候就不能在执行新的更新，得停下来先擦掉一些记录，把check point往前推进后才能继续。

### 2.6 undo log tablespace
My SQL5.6以后可以使用独立的undo表空间，将其从系统表空间中提出，使得不会因为大事务导致系统表空间不断的增大。

## 3. bin log（归档日志）
binlog以事件的形式记录了所有的DDL和DML语句，可以用来做主从复制和数据恢复

和redo log 不一样，他的文件内容是可以追加的，没有固定大小的限制。

在开启了binlog功能的情况下，我们可以把binlog导出成SQL语句，把所有的操作重放一遍，来实现数据恢复。

binlog的另一个功能就是用来实现主从复制，原理如图：
![主从复制原理](https://img-blog.csdnimg.cn/20201225083656600.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDk3NTI5MQ==,size_16,color_FFFFFF,t_70)

**注意：**
- 从库读取binlog日志，写relay日志，再将日志读取到本地，这个操作是串行化的。
所以从库上的数据要比主库上的数据慢一点。
- 延迟的主要因素还是取决于主库的并发量，主库的并发越高，从库的延迟就越高。
  * 从库可以开启并行复制来缓解延迟的问题，就是开启多个线程读取relay日志中不同库的数据，再并行的写到不同的库中
 - 还有就时数据丢失的问题。比如主库写了一条日志到binlog日志中了。还没有同步呢，但是这个时候主库挂了，那么从库就会变成主库。这条数据就丢失了。
   * 开启半同步复制可以解决这个问题，当主库写入一条binlog日志的时候强制将这条日志同步到从库的relay日志中，然后从库会返回一个ack，主库接收到这个ack才会认为这条数据写完了。

## 4. 后台IO线程
后台线程的主要作用是负责将buffer pool中的数据刷新到磁盘。
主要分为：
- master thread：负载刷新缓存数据到磁盘并协调调度其他后台线程。
- IO thread： 分为insert buffer、log、read、write进程，分别处理insert buffer、重做日志、读写请求的IO回调。
- purge thread：用来回收undo页
- page cleaner thread： 用来刷新脏页。