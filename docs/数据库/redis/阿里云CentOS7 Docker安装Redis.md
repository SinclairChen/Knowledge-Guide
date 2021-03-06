获取最新镜像
docker pull redis

查看已下载的镜像
docker images

因为Docker安装的Redis默认没有配置文件，所以需要挂载主机的配置文件
在主机环境中创建映射的配置和数据目录

```
mkdir -p /usr/local/soft/redis/conf/
mkdir -p /usr/local/soft/redis/data/
```

复制 `redis.conf` 文件到主机/usr/local/soft/redis/conf/目录下（[redis.conf](https://raw.githubusercontent.com/antirez/redis/4.0/redis.conf)）。

**注意有两行配置要修改：**
1. daemonize yes 这一行必须注释，否则无法启动
2. bind 127.0.0.1 改成 bind 0.0.0.0 ，否则只能在本机访问

运行Redis服务端：

```
docker run -p 6379:6379 --name redis5 -v /usr/local/soft/redis/conf/redis.conf:/etc/redis/redis.conf -v /usr/local/soft/redis/data/:/data -d redis:latest redis-server /etc/redis/redis.conf --appendonly yes
```

**注意：**
如果需要安装多个redis，可以修改映射的端口，如：

- 6391:6379
- 6392:6379
- 6393:6379

外网环境记得在末尾加上以下参数，以免被攻击

```
--requirepass "youpassword"
```

参数说明：

- -d 后台运行
- -p 6379:6379 端口映射（本机6379端口映射容器6379端口）
- –name redis5 容器别名
- -v /etc/app/redis/conf:/conf 目录映射（本机redis配置文件目录）
- -v /etc/app/redis/data:/data 目录映射（本机redis数据目录）
- redis-server /conf/redis.conf --appendonly yes 在容器运行命令，并打开数据持久化

连接Redis客户端：

```
docker exec -it redis5 redis-cli
```

如果需要远程访问6379端口，需要配置安全组策略
![20190918_180916.png](image/8c8daf771ac14dc8b1212639269af14a.png)

![20190918_180951.png](image/2bcd3e9207fc4995970fde8158db806e.png)