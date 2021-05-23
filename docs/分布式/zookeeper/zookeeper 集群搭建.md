三台服务器分别是
192.168.2.158
192.168.2.152
192.168.2.150
然后在三台服务器分别安装zookeeper

下载

```
wget http://mirrors.hust.edu.cn/apache/zookeeper/stable/apache-zookeeper-3.5.7-bin.tar.gz
```

解压到user/local下

```tar
mv /usr/local/apache-zookeeper-3.5.7 /usr/local/zookeeper
```

修改配置文件

```cp
vim /usr/local/zookeeper/conf/zoo.cfg
initLimit=10
syncLimit=5
dataDir=/usr/local/zookeeper/data
clientPort=2181
server.1=192.168.2.158:2888:3888
server.2=192.168.2.152:2888:3888
server.3=192.168.2.150:2888:3888
```

创建数据存储目录

```
mkdir -p /usr/local/zookeeper/data
```

创建myid文件， id 与 zoo.cfg 中的序号对应

```#在192.168.2.158机器上执行
echo 1 > /usr/local/zookeeper/data/myid
#在192.168.2.152机器上执行
echo 2 > /usr/local/zookeeper/data/myid
#在192.168.2.150机器上执行
echo 3 > /usr/local/zookeeper/data/myid
```

配置环境变量

```
vim /etc/profile
```

在最后加上

```
export ZK_HOME=/usr/local/zookeeper
export PATH=$ZK_HOME/bin:$PATH
```

强制生效

```
source /etc/profile
```

分别启动

```
zkServer.sh start
```