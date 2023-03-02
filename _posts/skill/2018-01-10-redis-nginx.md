---
layout: post
title: redis 以及nginx
category: 技术
tags:  redis nginx
keywords: redis nginx
description: redis缓存,nginx反向代理,动静分离
---


* content
{:toc}
redis缓存,nginx反向代理,动静分离



![startnginx](_posts/skill/startnginx.png)



## redis命令
	启动:	redis-server redis.conf
	关闭：redis-cli  shutdown 有密码先登录auth登录直接敲  shutdown
	登录(如有密码): auth 密码 
	
	连接：
		本地： redis-cli
		远程： redis-cli -h 192.168.86.20 -p 6379
	
	切换数据库(默认16个数据库)：select 0
	
	查看里面的键值： keys * 
	*代表所有,也可以输入表达式
	
	删除已有的键： del 键名
	
### 设置
	requirepass 设置密码
	db



### 通用操作 

> [参考](https://www.runoob.com/redis/redis-server.html)

``` bash
# 下面命令可直接运行
flushdb: 
flushall:

BGSAVE: 在后台异步保存当前数据库的数据到磁盘
CLIENT LIST: 获取连接到服务器的客户端连接列表

COMMAND: 获取 Redis 命令详情数组
COMMAND COUNT:获取 Redis 命令总数
COMMAND GETKEYS:获取给定命令的所有键
CONFIG GET parameter :获取指定配置参数的值 eg: config get *max-*-entries*
CONFIG SET parameter value: 修改 redis 配置参数，无需重启 eg: CONFIG SET slowlog-max-len 10086
DBSIZE :返回当前数据库的 key 的数量
ROLE   :返回主从实例所属的角色
SAVE	:同步保存数据到硬盘
BGSAVE	:在后台异步保存当前数据库的数据到磁盘
SHUTDOWN [NOSAVE] [SAVE]	:异步保存数据到硬盘，并关闭服务器
DEBUG SEGFAULT	:让 Redis 服务崩溃
SLAVEOF host port: 将当前服务器转变为指定服务器的从属服务器(slave server)	eg: SLAVEOF 127.0.0.1 6379(打开) ,   SLAVEOF NO ONE(关闭)
```





### 数据类型

> Redis支持五种数据类型：string（字符串），hash（哈希），list（列表），set（集合）及zset(sorted set：有序集合)。GEO 地理位置(3.2+), Stream结构（5.0+）



##### String (字符串)



```bash
192.168.86.20:6379> set tom tomcat
OK
192.168.86.20:6379> get tom
"tomcat"
```

##### hash(哈希)

> Redis hash 是一个键值(key=>value)对集合。
>
> Redis hash 是一个 string 类型的 field 和 value 的映射表，hash 特别适合用于存储对象。



``` bash

192.168.86.20:6379> hmset bob field1 "bb1" field2 "bb2"
OK
192.168.86.20:6379> hmget bob field1
1) "bb1"

```



##### list（列表）

> Redis 列表是简单的字符串列表，按照插入顺序排序。你可以添加一个元素到列表的头部（左边）或者尾部（右边）

``` bash
192.168.86.20:6379> lpush june barry1
(integer) 1
192.168.86.20:6379> lpush june barry2
(integer) 2
192.168.86.20:6379> lrange june 0 2
1) "barry2"
2) "barry1

```

##### set(集合)

> Redis 的 Set 是 string 类型的无序集合。
>
> 集合是通过哈希表实现的，所以添加，删除，查找的复杂度都是 O(1)。

``` bash

192.168.86.20:6379> sadd jike jike1
(integer) 1
192.168.86.20:6379> sadd jike jike2
(integer) 1
192.168.86.20:6379> sadd jike jike2
(integer) 0
192.168.86.20:6379>
192.168.86.20:6379> smembers jike
1) "jike1"
2) "jike2"
192.168.86.20:6379>



```





##### zset(sorted set：有序集合)

> Redis zset 和 set 一样也是string类型元素的集合,且不允许重复的成员。
>
> 
>
> 不同的是每个元素都会关联一个double类型的分数。redis正是通过分数来为集合中的成员进行从小到大的排序。
>
> zset的成员是唯一的,但分数(score)却可以重复。



``` bash

192.168.86.20:6379> zadd rose 0 rose1
(integer) 1
192.168.86.20:6379> zadd rose 1 rose1
(integer) 0
192.168.86.20:6379> zadd rose 2 rose2
(integer) 1
192.168.86.20:6379> zadd rose 3 rose3
(integer) 1
192.168.86.20:6379> zrangebyscore 0 2
(error) ERR wrong number of arguments for 'zrangebyscore' command
192.168.86.20:6379> zrangebyscore rose 0 2
1) "rose1"
2) "rose2"
192.168.86.20:6379>

```









### 订阅发布

> Redis 发布订阅 (pub/sub) 是一种消息通信模式：发送者 (pub) 发送消息，订阅者 (sub) 接收消息。
>
> Redis 客户端可以订阅任意数量的频道。

订阅之后控制台挂起,等有发布消息在打印出来

```bash
# 订阅

192.168.86.20:6379> subscribe myChat
Reading messages... (press Ctrl-C to quit)
1) "subscribe"
2) "myChat"
3) (integer) 1
1) "message"
2) "myChat"
3) "hello I'm bob"


```



``` bash
# 发布
192.168.86.20:6379> publish myChat "hello I'm bob"
(integer) 1
192.168.86.20:6379> publish myChat "hello I'm bob"

```



### 事务

> Redis 事务可以一次执行多个命令， 并且带有以下三个重要的保证：
>
> - 批量操作在发送 EXEC 命令前被放入队列缓存。
> - 收到 EXEC 命令后进入事务执行，事务中任意命令执行失败，其余的命令依然被执行。
> - 在事务执行过程，其他客户端提交的命令请求不会插入到事务执行命令序列中。
>
> 一个事务从开始到执行会经历以下三个阶段：
>
> - 开始事务。
> - 命令入队。
> - 执行事务。



* 开始事务命令: MULTI 
* 执行提交: EXEC
* 取消事务,回滚: DISCARD

``` bash
# 事务提交
192.168.86.20:6379> multi
OK
192.168.86.20:6379> set aaa aaa1
QUEUED
192.168.86.20:6379> set aaa1 aaa1
QUEUED
192.168.86.20:6379> set aaa1 aaa1
QUEUED
192.168.86.20:6379> exec
1) OK
2) OK
3) OK

# 事务回滚
192.168.86.20:6379> multi
OK
192.168.86.20:6379> set aaa2 aaa2
QUEUED
192.168.86.20:6379> discard
OK
```

























​	

## nginx 

### nginx -s reload  ：修改配置后重新加载生效
nginx -s reopen  ：重新打开日志文件
nginx -t -c /path/to/nginx.conf 测试nginx配置文件是否正确

### 关闭nginx：
nginx -s stop  :快速停止nginx
         quit  ：完整有序的停止nginx

### 其他的停止nginx 方式：

ps -ef | grep nginx

kill -QUIT 主进程号     ：从容停止Nginx
kill -TERM 主进程号     ：快速停止Nginx
pkill -9 nginx          ：强制停止Nginx



# 启动nginx:
nginx -c /path/to/nginx.conf

# 平滑重启nginx：
kill -HUP 主进程号