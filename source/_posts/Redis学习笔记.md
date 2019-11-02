---
title: Redis学习笔记
date: 2019-11-02 14:52:19
tags: ['Redis']
---

Redis！Redis！

<!--more-->

# 简介

> Redis is an open source (BSD licensed), in-memory **data structure store**, used as a database, cache and message broker. It supports data structures such as [strings](https://redis.io/topics/data-types-intro#strings), [hashes](https://redis.io/topics/data-types-intro#hashes), [lists](https://redis.io/topics/data-types-intro#lists), [sets](https://redis.io/topics/data-types-intro#sets), [sorted sets](https://redis.io/topics/data-types-intro#sorted-sets) with range queries, [bitmaps](https://redis.io/topics/data-types-intro#bitmaps), [hyperloglogs](https://redis.io/topics/data-types-intro#hyperloglogs) and [geospatial indexes](https://redis.io/commands/geoadd) with radius queries. Redis has built-in [replication](https://redis.io/topics/replication), [Lua scripting](https://redis.io/commands/eval), [LRU eviction](https://redis.io/topics/lru-cache), [transactions](https://redis.io/topics/transactions) and different levels of [on-disk persistence](https://redis.io/topics/persistence), and provides high availability via [Redis Sentinel](https://redis.io/topics/sentinel) and automatic partitioning with [Redis Cluster](https://redis.io/topics/cluster-tutorial).

Redis是一个开源（BSD许可），内存数据结构存储，用作数据库，缓存和消息代理。 它支持的数据结构有字符串，hash，列表，集合，带有范围查询的排序集，位图，超级日志和带有半径查询的地理空间索引。 Redis具有内置复制，Lua脚本，LRU驱逐，事务和不同级别的磁盘持久性，并通过Redis Sentinel提供高可用性并使用Redis Cluster自动分区。

## 特性

1. 速度快，最快可达到 `10w QPS`（基于 **内存**，`C` 语言，**单进程单线程** 架构）
2. 基于 **键值对** (`key/value`) 的数据结构服务器。全称 `Remote Dictionary Server`。包括 `string`(**字符串**)、`hash`(**哈希**)、`list`(**列表**)、`set`(**集合**)、`zset`(**有序集合**)、`bitmap`(**位图**)。同时在 **字符串** 的基础上演变出 **位图**（`BitMaps`）和 `HyperLogLog` 两种数据结构。`3.2` 版本中加入 `GEO`（**地理信息位置**）
3. 丰富的功能。例如：**键过期**（缓存），**发布订阅**（消息队列）， `Lua` 脚本（自己实现 `Redis` 命令），**事务**，**流水线**（`Pipeline`，用于减少网络开销）
4. 简单稳定。无外部库依赖，单线程模型
5. 客户端语言多
6. 持久化**（支持两种** 持久化方式 `RDB` 和 `AOF`）
7. 主从复制（分布式的基础）
8. **高可用**（`Redis Sentinel`），**分布式**（`Redis Cluster`）和 **水平扩容**

## 安装

- Mac

  ```
  brew install redis
  ```

  对应的配置文件在`/usr/local/etc/redis.conf`

## 常用命令

- 启动命令(Mac下用homebrew安装)

  ```
  redis-server /usr/local/etc/redis.conf
  ```

- 关闭指定端口的redis

  ```
  redis-cli -p {port} shutdown
  ```

- 查看redis信息

  ```
  info
  ```

# Redis

## 过期策略

1. 定时删除

   为设置了过期时间的key设置一个定时器。该策略可能造成占用大量的CPU时间

2. 定期删除

   每隔一段时间（配置文件中的hz，默认为10）删除已过期的key。是定时删除和惰性删除的解决方案

3. 惰性删除

   当Redis访问一个key时，判断这个key是否过期，如果过期了那么删除这个key。对CPU占用小，但是可能造成内存的浪费

**Redis同时使用了定期删除和惰性删除**

## 内存淘汰策略

当Redis执行每个命令的时候都会检查当前使用的内存是否超过配置中规定的maxmemory，如果超过，那么会执行对应的内存淘汰策略

以下是redis配置文件中摘录的：

```
# MAXMEMORY POLICY: how Redis will select what to remove when maxmemory
# is reached. You can select among five behaviors:
#
# volatile-lru -> remove the key with an expire set using an LRU algorithm
# allkeys-lru -> remove any key according to the LRU algorithm
# volatile-random -> remove a random key with an expire set
# allkeys-random -> remove a random key, any key
# volatile-ttl -> remove the key with the nearest expire time (minor TTL)
# noeviction -> don't expire at all, just return an error on write operations
#
# Note: with any of the above policies, Redis will return an error on write
#       operations, when there are no suitable keys for eviction.
#
#       At the date of writing these commands are: set setnx setex append
#       incr decr rpush lpush rpushx lpushx linsert lset rpoplpush sadd
#       sinter sinterstore sunion sunionstore sdiff sdiffstore zadd zincrby
#       zunionstore zinterstore hset hsetnx hmset hincrby incrby decrby
#       getset mset msetnx exec sort
#
# The default is:
#
# maxmemory-policy noeviction
```

共有六种淘汰策略

- noeviction

  对于写入操作报错，不会淘汰任何数据

- allkeys-random

  随机淘汰任意一个数据

- volatile-random

  从已设置过期的数据集中随机淘汰一个数据

- allkeys-lru

  根据最近最少使用策略淘汰任意一个数据

- volatile-lru

  根据最近最少使用过策略从已设置过期的数据集中淘汰一个数据

- volatile-ttl

  从已设置过期的数据集中淘汰即将过期的数据

**对于上述任何策略，当没有合适的可以被驱逐key时，Redis将在写入时返回错误操作**

## 持久化策略

1. RDB （默认策略）

   ```
   # Save the DB on disk:
   #
   #   save <seconds> <changes>
   #
   #   Will save the DB if both the given number of seconds and the given
   #   number of write operations against the DB occurred.
   #
   #   In the example below the behaviour will be to save:
   #   after 900 sec (15 min) if at least 1 key changed
   #   after 300 sec (5 min) if at least 10 keys changed
   #   after 60 sec if at least 10000 keys changed
   #
   #   Note: you can disable saving completely by commenting out all "save" lines.
   #
   #   It is also possible to remove all the previously configured save
   #   points by adding a save directive with a single empty string argument
   #   like in the following example:
   #
   #   save ""
   
   save 900 1    #after 900 sec (15 min) if at least 1 key changed
   save 300 10   #after 300 sec (5 min) if at least 10 keys changed
   save 60 10000 #after 60 sec if at least 10000 keys changed
   ```

   该方式会按照指定的时间策略将数据快照（dump.rdb）文件保存在硬盘上，如果发生意外，那么数据快照可能不是最新的，可能丢失几分钟的数据

1. AOF

   ```
   # By default Redis asynchronously dumps the dataset on disk. This mode is
   # good enough in many applications, but an issue with the Redis process or
   # a power outage may result into a few minutes of writes lost (depending on
   # the configured save points).
   #
   # The Append Only File is an alternative persistence mode that provides
   # much better durability. For instance using the default data fsync policy
   # (see later in the config file) Redis can lose just one second of writes in a
   # dramatic event like a server power outage, or a single write if something
   # wrong with the Redis process itself happens, but the operating system is
   # still running correctly.
   #
   # AOF and RDB persistence can be enabled at the same time without problems.
   # If the AOF is enabled on startup Redis will load the AOF, that is the file
   # with the better durability guarantees.
   ```

   Redis会将接受到的写命令写入到制定的文件中（默认为appendonly.aof），使用该方式持久化在遇到突发事件如断电时只会丢失一秒的写入数据，或者在Redis进程奔溃时但操作系统依旧正确的运行时只会丢失一次写入数据。

   Redis有三种数据fsync策略

   1. appendfsync everysec

      每秒将buffer中的写入命令写入到文件中。这是默认使用的策略，是一种性能和数据安全的折中策略

   2. appendfsync no

      依赖操作系统的写入，一般30s左右写入一次。性能最好，但是无法保证数据安全

   3. appendfsync always

      每个命令都会直接写入到文件中。数据安全，但性能低下

## Redis内存回收策略

### 引用计数法

- 在创建一个新对象时， 引用计数的值会被初始化为 `1` ；
- 当对象被一个新程序使用时， 它的引用计数值会被增一；
- 当对象不再被一个程序使用时， 它的引用计数值会被减一；
- 当对象的引用计数值变为 `0` 时， 对象所占用的内存会被释放。

## 部署方式

### 主从模式

缓解主库压力，做读写分离（从库默认情况下无法写入，只能读取）。如果主节点down，那么只能手动恢复主节点。哨兵模式解决了自动化的故障恢复

#### 本机配置

- master实例，端口6379
- slave1实例，端口20000
- slave2实例，端口20001

```
# master配置(/redis_conf/redis-master.conf)，其余默认
bind 127.0.0.1
port 6379
# slave1配置(/redis_conf/redis-slave-20000.conf)，其余默认
slaveof 127.0.0.1 6379
port 20000
# slave2配置(/redis_conf/redis-slave-20001.conf)，其余默认
slaveof 127.0.0.1 6379
port 20001
```

依次启动master、slave

```
redis-server /redis_conf/redis-master.conf
redis-server /redis_conf/redis-slave-20000.conf
redis-server /redis_conf/redis-slave-20001.conf
```

输入`redis-cli -p 6379`进入master的客户端，输入`info replication`查看从节点，显示如下

```
# Replication
role:master
connected_slaves:2
slave0:ip=127.0.0.1,port=20001,state=online,offset=380,lag=0
slave1:ip=127.0.0.1,port=20000,state=online,offset=380,lag=0
master_repl_offset:380
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:2
repl_backlog_histlen:379
```

主节点写入一个key-value就能在两个从节点中get到

### 哨兵模式

> 主从机制，上面的方案中主服务器可能存在单点故障，万一主服务器宕机，这是个麻烦事情，所以Redis提供了Redis-Sentinel，以此来实现主从切换的功能，类似与zookeeper。
>
> Redis-Sentinel是Redis官方推荐的高可用性(HA)解决方案，当用Redis做master-slave的高可用方案时，假如master宕机了，Redis本身(包括它的很多客户端)都没有实现自动进行主备切换，而Redis-Sentinel本身也是一个独立运行的进程，它能监控多个master-slave集群，发现master宕机后能进行自动切换。
>
> 它的主要功能有以下几点
>
> - 监控（Monitoring）：不断地检查redis的主服务器和从服务器是否运作正常。
> - 提醒（Notification）：如果发现某个redis服务器运行出现状况，可以通过 API 向管理员或者其他应用程序发送通知。
> - 自动故障迁移（Automatic failover）：能够进行自动切换。当一个主服务器不能正常工作时，会将失效主服务器的其中一个从服务器升级为新的主服务器，并让失效主服务器的其他从服务器改为复制新的主服务器； 当客户端试图连接失效的主服务器时， 集群也会向客户端返回新主服务器的地址， 使得集群可以使用新主服务器代替失效服务器。

#### 本地配置

现在运行了三个redis实例，一主两从

```
$ ps -ef | grep redis
root       869     1  0 9月11 ?       00:01:34 redis-server 127.0.0.1:20001
root      1181     1  0 9月11 ?       00:01:35 redis-server 127.0.0.1:20000
root     23943 19684  0 10:27 pts/3    00:00:00 redis-server 127.0.0.1:6379
```

编写三个配置sentinel配置文件

```
port 10000
sentinel monitor mymaster 127.0.0.1 6300 2
port 10001
sentinel monitor mymaster 127.0.0.1 6300 2
port 10002
sentinel monitor mymaster 127.0.0.1 6300 2
```

用`redis-sentinel {sentinel-conf}`运行三个sentinel实例

然后将端口为6379的master干掉`redis-cli -p 6379 shutdown`

此时会发现某个slave通过选举晋升为了master

如端口为20000的slave晋升为了master，查看info

```
#Replication
role:master # 角色变为了master
connected_slaves:1
slave0:ip=127.0.0.1,port=20001,state=online,offset=30728,lag=0 #拥有一个端口为20001的slave
master_repl_offset:30728
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:2
repl_backlog_histlen:30727
```

### 集群模式







# 参考链接

https://juejin.im/post/5b76e732f265da4376203849

https://juejin.im/post/5b7d226a6fb9a01a1e01ff64

http://blog.720ui.com/2016/redis_action_04_cluster/

















> *大部分正义感都是虚伪的 聊胜于无   —————————————————————————沃德彭尤·胡某*

