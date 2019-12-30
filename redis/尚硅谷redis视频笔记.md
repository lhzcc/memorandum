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

   ### Redis常用命令

   ------

   > Redis五种数据类型：string、hash、list、set、zset

##### 公用命令

```shell
DEL key
DUMP key：序列化给定key，返回被序列化的值
EXISTS key：检查key是否存在
EXPIRE key second：为key设定过期时间
TTL key：返回key剩余时间
PERSIST key：移除key的过期时间，key将持久保存
KEY pattern：查询所有符号给定模式的key
RANDOM key：随机返回一个key
RANAME key newkey：修改key的名称
MOVE key db：移动key至指定数据库中
TYPE key：返回key所储存的值的类型
```

```text
EXPIRE key second的使用场景：
1、限时的优惠活动
2、网站数据缓存
3、手机验证码
4、限制网站访客频率
```

##### key的命名建议

1. key不要太长，尽量不要超过1024字节。不仅消耗内存，也会降低查找的效率
2. key不要太短，太短可读性会降低
3. 在一个项目中，key最好使用统一的命名模式，如user:123:password
4. key**区分大小写**

#### string

> string类型是二进制安全的，redis的string可以包含任何数据，如图像、序列化对象。一个键最多能存储512MB。==二进制安全是指，在传输数据的时候，能保证二进制数据的信息安全，也就是不会被篡改、破译；如果被攻击，能够及时检测出来 ==

```shell
setkey_name value：命令不区分大小写，但是key_name区分大小写
SETNX key value：当key不存在时设置key的值。（SET if Not eXists）
get key_name
GETRANGE key start end：获取key中字符串的子字符串，从start开始，end结束
MGET key1 [key2 …]：获取多个key
GETSET KEY_NAME VALUE：设定key的值，并返回key的旧值。当key不存在，返回nil
STRLEN key：返回key所存储的字符串的长度
INCR KEY_NAME ：INCR命令key中存储的值+1,如果不存在key，则key中的值话先被初始化为0再加1
INCRBY KEY_NAME 增量
DECR KEY_NAME：key中的值自减一
DECRBY KEY_NAME
append key_name value：字符串拼接，追加至末尾，如果不存在，为其赋值
```

String应用场景：

```tex
1、String通常用于保存单个字符串或JSON字符串数据
2、因为String是二进制安全的，所以可以把保密要求高的图片文件内容作为字符串来存储
3、计数器：常规Key-Value缓存应用，如微博数、粉丝数。INCR本身就具有原子性特性，所以不会有线程安全问题
```

#### hash

Redis hash是一个string类型的field和value的映射表，**hash特别适用于存储对象**。每个hash可以存储232-1键值对。可以看成KEY和VALUE的MAP容器。相比于JSON，hash占用很少的内存空间。

***常用命令***

```shell
HSET key_name field value：为指定的key设定field和value
hmset key field value[field1,value1]
hget key field
hmget key field[field1]
hgetall key：返回hash表中所有字段和值
hkeys key：获取hash表所有字段
hlen key：获取hash表中的字段数量
-hdel key field [field1]：删除一个或多个hash表的字段
```

应用场景

```tex
Hash的应用场景，通常用来存储一个用户信息的对象数据。
1、相比于存储对象的string类型的json串，json串修改单个属性需要将整个值取出来。而hash不需要。
2、相比于多个key-value存储对象，hash节省了很多内存空间
3、如果hash的属性值被删除完，那么hash的key也会被redis删除
```

#### list

类似于Java中的LinkedList。

***常用命令***

```shell
lpush key value1 [value2]
rpush key value1 [value2]
lpushx key value：从左侧插入值，如果list不存在，则不操作
rpushx key value：从右侧插入值，如果list不存在，则不操作
llen key：获取列表长度
lindex key index：获取指定索引的元素
lrange key start stop：获取列表指定范围的元素
lpop key ：从左侧移除第一个元素
prop key：移除列表最后一个元素
blpop key [key1] timeout：移除并获取列表第一个元素，如果列表没有元素会阻塞列表到等待超时或发现可弹出元素为止
brpop key [key1] timeout：移除并获取列表最后一个元素，如果列表没有元素会阻塞列表到等待超时或发现可弹出元素为止
ltrim key start stop ：对列表进行修改，让列表只保留指定区间的元素，不在指定区间的元素就会被删除
lset key index value ：指定索引的值
linsert key before|after world value：在列表元素前或则后插入元素
```

应用场景

```tex
1、对数据大的集合数据删减
		列表显示、关注列表、粉丝列表、留言评价...分页、热点新闻等
2、任务队列
		list通常用来实现一个消息队列，而且可以确保先后顺序，不必像MySQL那样通过order by来排序
```

> 补充：
>
> rpoplpush list1 list2 移除list1最后一个元素，并将该元素添加到list2并返回此元素
> 用此命令可以实现订单下单流程、用户系统登录注册短信等。

#### set

唯一、无序

```shell
sadd key value1[value2]：向集合添加成员
scard key：返回集合成员数
smembers key：返回集合中所有成员
sismember key member：判断memeber元素是否是集合key成员的成员
srandmember key [count]：返回集合中一个或多个随机数
srem key member1 [member2]：移除集合中一个或多个成员
spop key：移除并返回集合中的一个随机元素
smove source destination member：将member元素从source集合移动到destination集合
sdiff key1 [key2]：返回所有集合的差集
sdiffstore destination key1[key2]：返回给定所有集合的差集并存储在destination中
```

```tex
对两个集合间的数据[计算]进行交集、并集、差集运算
1、以非常方便的实现如共同关注、共同喜好、二度好友等功能。对上面的所有集合操作，你还可以使用不同的命令选择将结果返回给客户端还是存储到一个新的集合中。
2、利用唯一性，可以统计访问网站的所有独立 IP
```

#### zset

有序且不重复。每个元素都会关联一个double类型的分数，Redis通过分数进行从小到大的排序。分数可以重复

```shell
ZADD key score1 memeber1
ZCARD key ：获取集合中的元素数量
ZCOUNT key min max 计算在有序集合中指定区间分数的成员数
ZCOUNT key min max 计算在有序集合中指定区间分数的成员数
ZRANK key member：返回有序集合指定成员的索引
ZREVRANGE key start stop ：返回有序集中指定区间内的成员，通过索引，分数从高到底
ZREM key member [member …] 移除有序集合中的一个或多个成员
ZREMRANGEBYRANK key start stop 移除有序集合中给定的排名区间的所有成员(第一名是0)(低到高排序）
ZREMRANGEBYSCORE key min max 移除有序集合中给定的分数区间的所有成员
```

```tex
常用于排行榜：
1、如推特可以以发表时间作为score来存储
2、存储成绩
3、还可以用zset来做带权重的队列，让重要的任务先执行
```

### Jedis和Spring-data-redis

------

1. 连接**redis**
2. JedisPool
3. Spring-data

### Redis功能特性

------

#### 发布订阅

Redis 发布订阅(pub/sub)是一种消息通信模式：发送者(pub)发送消息，订阅者(sub)接收消息。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190925161943181.png)

Redis 客户端可以订阅任意数量的频道。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190925161952995.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMzNDIzNDE4,size_16,color_FFFFFF,t_70)

下图展示了频道 channel1 ， 以及订阅这个频道的三个客户端 —— client2 、 client5 和 client1 之间的关系：

当有新消息通过 PUBLISH 命令发送给频道 channel1 时， 这个消息就会被发送给订阅它的三个客户端：

命令

```shell
subscribe channel [channel…]：订阅一个或多个频道的信息
psubscribe pattern [pattern…]：订阅一个或多个符合规定模式的频道
publish channel message ：将信息发送到指定频道
unsubscribe [channel[channel…]]：退订频道
punsubscribe [pattern[pattern…]]：退订所有给定模式的频道
```

***应用场景***

```
构建实时的消息系统，比如普通聊天、群聊等功能。
1、博客网站订阅，当作者发布就可以推送给粉丝
2、微信公众号模式
```

#### Redis多数据库

```shell
select db
move key db
flushdb
flushall
```

#### Redis 事务

Redis事务可以一次执行多个命令，（按顺序地串行化执行，执行过程中不允许其他命令插入执行序列中）。
1、Redis会将一个事务中的所有命令序列化，然后按顺序执行
2、执行中不会被其他命令插入，不允许加塞行为

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190925163101930.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMzNDIzNDE4,size_16,color_FFFFFF,t_70)

> 1输入Multi命令开始，输入的命令都会依次进入命令队列中，但不会执行
> 2直到输入exec后，Redis会将之前队列中的命令依次执行
>
> ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190925163359733.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMzNDIzNDE4,size_16,color_FFFFFF,t_70)
>
> 3discard放弃队列执行
>
> ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190925163647259.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMzNDIzNDE4,size_16,color_FFFFFF,t_70)
>
> 4如果某个命令报出错，则只有报错的命令不会被执行，而其他的命令都会执行，不会回滚。
>
> ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190925163900602.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMzNDIzNDE4,size_16,color_FFFFFF,t_70)
>
> 5如果队列中某个命令出现报告错误（语法错误），执行时整个队列都会被取消

主从复制
[Redis主从复制配置](https://blog.csdn.net/qq_33423418/article/details/101442873)

Redis集群
Redis集群搭建：
[Redis documentation](https://redis.io/documentation)
[Redis cluster tutorial](https://redis.io/topics/cluster-tutorial)
[Redis-5.0.5集群搭建](https://blog.csdn.net/qq_33423418/article/details/101516948)
[单机Redis和集群Redis的区别](https://blog.csdn.net/yuhaibao324/article/details/92008507)
[Redis-4.0集群搭建](https://blog.csdn.net/lx1309244704/article/details/80681292)

Redis持久化
	1.RDB

> RDB是Redis默认持久化机制。RDB相当于快照，保存的是一种状态

​	优点：
​		保存速度、还原速度极快,适用于灾难备份
​	缺点：
​		小内存的机器不符合使用。RDB机制符合要求就会快照。

​	2.AOF

如果Redis意外down掉，RDB方式会丢失最后一次快照后的所有修改。如果要求应用不能丢失任何修改，可以采用AOF持久化方式。

> AOF：Append-Only File：Redis会将没一个收到写命令都追加到文件中（默认是appendonly.aof）。当Redis重启时会通过重新执行文件中的写命令重建整个数据库的内容。

产生的问题：
有些命令是多余的。

转自https://blog.csdn.net/qq_33423418/article/details/101351944