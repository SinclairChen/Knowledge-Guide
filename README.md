
## Java

### 基础

**知识点/面试题:**(必看:+1: )



### 并发

1. **[多线程的基本原理](docs/java/并发/多线程的基本原理.md)** 
2. **[原子性问题、synchronized及CAS](docs/java/并发/原子性问题、synchronized及CAS.md)**
3. **[可见性、有序性问题](docs/java/并发/可见性、有序性问题.md)**
4. **[JMM与happens-before原则](docs/java/并发/JMM与hapens-before原则.md)**
5. **[AQS源码分析](docs/java/并发/AQS源码分析.md)**
6. **[Lock与ReentrantLock源码分析](docs/java/并发/Lock与ReentrantLock源码分析.md)**
7. **[Condition源码分析](docs/java/并发/Condition源码分析.md)**

**其他**

- [AbstractQueuedSynchronizer中的tryAcquire为什么没有设置为抽象方法](docs/java/并发/AbstractQueuedSynchronizer中的tryAcquire为什么没有设置为抽象方法.md)



### 框架

#### Spring



#### Mybatis



#### SpringBoot



#### SpringCloud



### JVM

1. **[类的加载](docs/java/jvm/类的加载.md)**
2. **[运行时数据区](docs/java/jvm/运行时数据区.md)**
3. **[JVM内存模型](docs/java/jvm/jvm内存模型.md)**
4. **[垃圾回收](docs/java/jvm/垃圾回收.md)**
5. **[jvm调优——常用参数篇](docs/java/jvm/jvm调优-常用参数篇.md)**
6. **[jvm调优——常用命令篇](docs/java/jvm/jvm调优-常用命令篇.md)**
7. **[jvm调优——常用工具篇](docs/java/jvm/jvm调优-常用工具篇.md)**
8. **[GC调优分析](docs/java/jvm/GC调优分析.md)**




## 数据库

### Redis

**安装、配置与使用**

1. **[CentOS7安装Redis 6.0.9 单实例](docs/数据库/redis/CentOS7安装Redis%206.0.9%20单实例.md)**
2. **[阿里云CentOS7 Docker安装Redis](docs/数据库/redis/阿里云CentOS7%20Docker安装Redis.md)**
3. **[CentOS7 单机安装Redis Cluster (3主3从)](docs/数据库/redis/CentOS%207%20单机安装Redis%20Cluster（3主3从）.md)**
4. **[Redis6.0.9一主二从Sentinel监控配置](docs/数据库/redis/Redis6.0.9一主二从Sentinel监控配置.md)**

**数据类型：** **[string](docs/数据库/redis/String字符串.md)**

**原理**

1. **[redis过期与淘汰策略](docs/数据库/redis/redis过期与淘汰策略.md)**
2. **[redis持久化机制RDB和AOF](docs/数据库/redis/redis持久化机制RDB和AOF.md)**
3. **[redis为什么这么快？](docs/数据库/redis/redis为什么这么快？.md)**
4. **[redis的事务](docs/数据库/redis/redis的事务.md)**
5. **[如何在Redis使用Lua脚本](docs/数据库/redis/如何在Redis使用Lua脚本.md)**
6. **[redis的主从复制与sentinel](docs/数据库/redis/redis的主从复制与sentinel.md)**
7. **[redis cluster](docs/数据库/redis/redis%20cluster.md)**
8. **[缓存一致性问题](docs/数据库/redis/缓存一致性问题.md)**
9. **[缓存雪崩与缓存穿透](docs/数据库/redis/缓存雪崩与缓存穿透.md)**



### MySQL

**安装、配置与使用**

1. **[CentOS7 yum方式安装MySQL 5.7](docs/数据库/mysql/CentOS7%20yum方式安装MySQL%205.7.md)**
2. **[阿里云CentOS 7 Docker安装MySQL 5.7](docs/数据库/mysql/阿里云CentOS%207%20Docker安装MySQL%205.7.md)**
3. **[MySQL主从复制配置](docs/数据库/mysql/MySQL主从复制配置.md)**


**原理**

1. **[一条查询语句在MySQL中到底是如何执行的？](docs/数据库/mysql/一条查询语句在MySQL中到底是如何执行的？.md)**
2. **[一条更新语句在MySQL是如何执行的？](docs/数据库/mysql/一条更新语句在MySQL是如何执行的？.md)**
3. **[数据库的事务及MVCC详解](docs/数据库/mysql/数据库的事务及MVCC详解.md)**
4. **[MySQL中的锁](docs/数据库/mysql/MySQL中的锁.md)**


### ElasticSearch



## 消息中间件

- **[为什么要使用MQ？](docs/消息中间件/为什么要使用MQ？.md)**

### RabbitMQ

- **[认识RabbitMQ](docs/消息中间件/rabbitmq/认识RabbitMQ.md)**

**安装**

1. **[CentOS7安装RabbitMQ单机版(3.8.4)](docs/消息中间件/rabbitmq/CentOS7安装RabbitMQ单机版(3.8.4).md)**
2. **[Docker安装RabbitMQ集群.md](docs/消息中间件/rabbitmq/Docker安装RabbitMQ集群.md)**
3. **[HAProxy+Keepalived搭建RabbitMQ高可用集群](docs/消息中间件/rabbitmq/HAProxy+Keepalived搭建RabbitMQ高可用集群.md)**
4. **[RabbitMQ安装问题汇总](docs/消息中间件/rabbitmq/RabbitMQ安装问题汇总.md)**

**使用&配置**

1. **[Rabbitmq的使用](docs/消息中间件/rabbitmq/rabbitmq的使用.md)**
2. **[RabbitMQ可靠性消息投递](docs/消息中间件/rabbitmq/RabbitMQ可靠性消息投递.md)**
3. **[RabbitMQ集群与高可用](docs/消息中间件/rabbitmq/RabbitM集群与高可用.md)**



### RocketMQ



### Kafka



## 设计模式

**[设计模式简介与七大设计原则](docs/设计模式/设计模式简介与七大设计原则.md)**

### 创建型模式：

 1.**[工厂模式](docs/设计模式/工厂方法模式.md)**  2. **[单例模式](docs/设计模式/单例模式.md)**  3.  **[原型模式](docs/设计模式/原型模式.md)**  4.  **[建造者模式](docs/设计模式/建造者模式.md)**

### 结构型模式：

 1. **[代理模式](docs/设计模式/代理模式.md)** 2. **[适配器模式](docs/设计模式/适配器模式.md)** 3. **[桥接模式](docs/设计模式/桥接模式.md)** 4. **[享元模式](docs/设计模式/享元模式.md)** 5. **[组合模式](docs/设计模式/组合模式.md)** 6. **[装饰器模式](docs/设计模式/装饰器模式.md)** 7. **[门面模式](docs/设计模式/门面模式.md)**

### 行为型模式：

1. **[委派模式](docs/设计模式/委派模式.md)** 2. **[模板方法模式](docs/设计模式/模板方法模式.md)** 3. **[策略模式](docs/设计模式/策略模式.md)** 4. **[责任链模式](docs/设计模式/责任链模式.md)**



## 分布式

### zookeeper

**安装、配置与使用**

1. **[CentOS7安装Zookeeper](docs/分布式/zookeeper/CentOS7安装Zookeeper.md)**
2. **[zookeeper 集群搭建](docs/分布式/zookeeper/zookeeper%20集群搭建.md)**



### dubbo



## 服务器


### Nginx


**安装、配置与使用**

1. **[CentOS7源码方式安装nginx-1.18.0](docs/服务器/nginx/CentOS7源码方式安装nginx-1.18.0.md)**
2. **[Nginx 1.18.0 安装第三方负载均衡模块 fair](docs/服务器/nginx/Nginx%201.18.0%20安装第三方负载均衡模块%20fair.md)**



### Tomcat



## 计算机基础

### 操作系统

#### Linux

**安装、配置与使用**

1. **[VMware+Centos7 静态IP设置方法.md](docs/计算机基础/操作系统/linux/VMware+Centos7%20静态IP设置方法.md)**



### 数据结构与算法



### 计算机网络



## 工具

### maven

1. **[解决maven依赖变红，jar包无法下载Could not transfer artifact](docs/工具/maven/解决maven依赖变红，jar包无法下载Could%20not%20transfer%20artifact.md)**



## 云原生

### Docker



### Kubernetes

