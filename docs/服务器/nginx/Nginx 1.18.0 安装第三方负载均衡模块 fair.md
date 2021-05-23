建议先备份一下Nginx。

这里涉及到两个路径：
/usr/local/soft/nginx-1.18.0是源码路径。
/usr/local/soft/nginx/是安装编译后的路径。

## 1、下载解压

放在编译后的路径或者源码路径都可以。

```
cd /usr/local/soft/nginx/modules
wget https://files.cnblogs.com/files/ztlsir/nginx-upstream-fair-master.zip
tar -zxvf nginx-upstream-fair-master.zip
```



## 2、备份nginx启动文件

把编译后路径的脚本备份一下。

```
cd /usr/local/soft/nginx/sbin
cp nginx nginx.bak
```

## 3、在nginx原解压根目录下add module

注意，如果之前已经启用了、添加了其他的模块，需要把–add-module的参数加在最后面。
先查看之前的启动参数：

```
./nginx -V
cd /usr/local/soft/nginx-1.18.0
./configure --prefix=/usr/local/soft/nginx --add-module=/usr/local/soft/nginx/modules/nginx-upstream-fair-master
```

## 4、在nginx原解压根目录下 make，确保没有错误

```
cd /usr/local/soft/nginx-1.18.0
make
```

## 5、检查是否安装成功

```
cd /usr/local/soft/nginx-1.18.0/objs/
./nginx -V
```

## 6、复制objs目录下的nginx文件到sbin目录，覆盖原文件

```
cp -rf nginx /usr/local/soft/nginx/sbin/
```

## 7、重启Nginx

```
./nginx -c /usr/local/soft/nginx/conf/nginx-load.conf
```

Nginx配置：

```
    upstream ecif {
        fair;
        server 192.168.44.1:12673;
        server 192.168.44.1:12674;
        server 192.168.44.1:12675;
    }
```