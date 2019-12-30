### Redis简介

```shell
1 | Redis：开源、免费、高性能、K-V数据库、内存数据库、非关系型数据库，支持持久化、集群和事务
```

### Redis安装及配置

------

1. 用docker运行Redis

   ```shell
   docker pull redis
   docker run -d --name redis -p 6379:6379 redis
   docker exec -it redis redis-cli
   ```

2. Linux安装

   ```shell
   #确保Linux已经安装gcc
   #下载Redis
   wget http://download.redis.io/releases/redis-4.0.1.tar.gz
   #解压
   tar -zxvf redis-4.0.1.tar.gz
   #进入目录后编译
   cd redis-4.0.1
   make MALLOC=libc
   #安装
   make PREFIX=/usr/local/redis install #指定安装目录为/usr/local/redis
   #启动
   /usr/local/redis/bin/redis-server
   ```

3. Redis配置

   ```shell
   #进入解压的Redis目录，将redis.conf复制到安装文件的目录下
   cp redis.conf /usr/local/redis
   #启动自定义配置的Redis
   /usr/local/redis/bin/redis-server /usr/local/redis/redis.conf
   ```

4. 配置详解

   ```shell
   aemonize ： 默认为no，修改为yes启用守护线程
   port ：设定端口号，默认为6379
   bind ：绑定IP地址
   databases ：数据库数量，默认16
   save <second> <changes> ：指定多少时间、有多少次更新操作，就将数据同步到数据文件
   #redis默认配置有三个条件，满足一个即进行持久化
   save 900 1 #900s有1个更改
   save 300 10 #300s有10个更改
   save 60 10000 #60s有10000更改
   dbfilename ：指定本地数据库的文件名，默认为dump.rdb
   dir ：指定本地数据库的存放目录，默认为./当前文件夹
   requirepass ：设置密码，默认关闭
   redis -cli -h host -p port -a password
   ```

5. Redis关闭

   ```shell
   #使用kill命令 (非正常关闭，数据易丢失)
   ps -ef|grep -i redis
   kill -9 PID
   #正常关闭
   redis-cli shutdown
   ```

   

