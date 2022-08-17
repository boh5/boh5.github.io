---
categories:
- 学习笔记
- redis
date: "2022-07-29T00:00:00+08:00"
lastmod: 2022-07-30T21:16:00+08:00
tags:
- redis
- 数据库
- 缓存
- 分布式
- 集群
title: 高级篇（一）Redis 分布式缓存
---
单机的Redis存在四大问题:

1. 数据丢失问题：redis 持久化
2. 并发能力问题：主从集群，读写分离
3. 储存能力问题：分配集群，插槽机制动态扩容
4. 故障恢复问题：redis 哨兵，健康检测和自动恢复

## 1. Redis 持久化

### 1.1 RDB （Redis Database Backup file）

Redis数据快照，把内存中的所有数据都记录到磁盘中。

#### 1.1.1 执行时机

- 执行 save 命令：主进程执行，会阻塞所有命令
- 执行 bgsave 命令：开启子进程执行
- Redis 停机时
- 触发 RDB 条件时：redis.conf 中配置，如 `save 900 1 ` 表示 900 秒内至少一个 key 被修改，则执行 `bgsave`

#### 1.1.2 RDB 原理

fork采用的是copy-on-write技术：

- 当主进程执行读操作时，访问共享内存
- 当主进程执行写操作时，则会拷贝一份数据，执行写操作
- 极端情况内存占用翻倍

![rdb 原理](../images/image-20210725151319695.png)

#### 1.1.3 总结

- RDB 基本流程：fork (共享内存空间) -> 子进程读内存写入新 RDB 文件 -> 替换旧 RDB 文件
- `save 60 1000` 的含义？
- RDB 缺点：
    - 数据丢失风险
    - 耗时长

### 1.2 AOF (Append Only File)

#### 1.2.1 AOF 原理

AOF 记录写命令，可以看作命令日志，恢复时把记录的命令执行一遍

#### 1.2.2 AOF 配置

- 默认关闭
- 记录频率配置
    - `appendfsync always`：立即记录
    - `appendfsync everysec`：每秒记录一次(**默认**)
    - `appendfsync no`：由操作系统 fsync

#### 1.2.3 AOF 文件重写

AOF会记录对同一个key的多次写操作，但只有最后一次写操作才有意义。通过执行 `bgrewriteaof` 命令，可以让AOF文件执行重写功能，用最少的命令达到相同效果。

- `bgrewriteaof`
- 也可以在 redis.conf 中配置阈值触发

### 1.3 RDB vs AOF

![RDB vs AOF](../images/image-20210725151940515.png)

## 2. Redis 主从

命令或配置文件创建主从关系：`{slaveof | replicaof} <masterip> <masterport>`

### 2.2 主从同步原理

#### 2.2.1 全量同步

主从第一次建立连接时，会执行**全量同步**

![全量同步](../images/image-20210725152222497.png)

master如何得知salve是第一次来连接呢？？

有几个概念，可以作为判断依据(版本信息)：

- **Replication Id**：简称 replid，是数据集的标记，id一致则说明是同一数据集。每一个 master 都有唯一的 replid，slave 则会继承
  master 节点的 replid
- **offset**：偏移量，随着记录在 repl_backlog 中的数据增多而逐渐增大。slave 完成同步时也会记录当前同步的 offset。如果 slave
  的 offset小于 master 的 offset，说明 slave 数据落后于 master，需要更新。

因此slave做数据同步，必须向 master 声明自己的 replication id 和 offset，master 才可以判断到底需要同步哪些数据。

**master判断一个节点是否是第一次同步的依据，就是看replid是否一致**。

#### 2.2.2 增量同步

如果 slave 重启，执行增量同步

![增量同步](../images/image-20210725153201086.png)

#### 2.2.4 repl_backlog 原理

这个文件是一个固定大小的数组，只不过数组是环形，也就是说**角标到达数组末尾后，会再次从0开始读写**，这样数组头部的数据就会被覆盖。

直到数组被填满，会覆盖旧数据，如果覆盖的是已同步的数据，没有影响；但如果被覆盖的是未同步的数据，那只能做**全量同步**了。

### 2.3 主从同步优化

- 在master中配置 `repl-diskless-sync yes` 启用无磁盘复制，避免全量同步时的磁盘IO。
- Redis 单节点上的内存占用不要太大，减少 RDB 导致的过多磁盘 IO
- 适当提高 repl_backlog 的大小，发现 slave 宕机时尽快实现故障恢复，尽可能避免全量同步
- 限制一个 master 上的 slave 节点数量，如果实在是太多 slave，则可以采用主-从-从链式结构，减少 master 压力

### 2.4 小结

简述全量同步和增量同步区别？

- 全量同步：master 将完整内存数据生成 RDB，发送 RDB 到 slave。后续命令则记录在 repl_backlog，逐个发送给 slave。
- 增量同步：slave 提交自己的 offset 到 master，master 获取 repl_backlog 中从 offset 之后的命令给 slave

什么时候执行全量同步？

- slave 节点第一次连接 master 节点时
- slave 节点断开时间太久，repl_backlog 中的 offset 已经被覆盖时

什么时候执行增量同步？

- slave 节点断开又恢复，并且在 repl_backlog 中能找到 offset 时

## 3. Redis 哨兵

### 3.1 哨兵原理

#### 3.1.1 集群结构和作用
结构：

![哨兵结构](../images/image-20210725154528072.png)

作用：
- 监控
- 自动故障恢复：将一个 slave 提升为 master， 故障实力恢复后也以新 master 为主
- 通知：Sentinel 充当 Redis 客户端的服务发现来源，故障结构变化后，会通知客户端(谁是 master，谁是 slave)

#### 3.1.2 集群监控原理

Sentinel 基于心跳机制，每秒发个 ping 命令：
- 主观下线：某个 Sentinel 发现某个实例超时未响应
- 客观下线(实际判定为故障节点)：超过指定数量的 Sentinel 都认为该实例主观下线

#### 3.1.3 集群故障恢复原理

选举新的 master：
- 判断 slave 于 master 断开时间长短，超过指定值则排除该 slave
- 判断 slave 的 slave-priority，越小越优先
- 如果 slave-priority 一样，则判断 slave 的 offset，越新越优先
- 随便挑一个(id 越小越优先)

实现故障转移
- 向 slave 发送 `slaveof no one` 命令，让该节点成为 master
- Sentinel 让其他所有 slave 发送 `slaveof <ip> <port>`，使其成为新 master 的 slave
- 将故障节点标记为 slave，故障恢复后自动成为新 master 的 slave

#### 3.1.4.小结

Sentinel的三个作用是什么？

- 监控
- 故障转移
- 通知

Sentinel 如何判断一个 redis 实例是否健康？

- 每隔1秒发送一次 ping 命令，如果超过一定时间没有相向则认为是主观下线
- 如果大多数 sentinel 都认为实例主观下线，则判定服务下线

故障转移步骤有哪些？

- 首先选定一个slave作为新的master，执行 slaveof no one
- 然后让所有节点都执行 slaveof 新master
- 修改故障节点配置，添加 slaveof 新master

### 3.2 搭建哨兵集群

配置哨兵:

```conf
# sentinel.conf
port 27001
sentinel announce-ip 192.168.31.57
sentinel monitor mymaster 192.168.31.57 7001 2
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 60000
dir "/tmp/s1"
```

启动：`redis-sentinel s1/sentinel.conf`

### 3.3 redis-py 使用哨兵集群

```python
sentinel = redis.sentinel.Sentinel([('192.168.31.57', 27001), ('192.168.31.57', 27002), ('192.168.31.57', 27003)])
master = sentinel.master_for('mymaster')
master.set('foo', 'bar')
slave = sentinel.slave_for('mymaster')
resp = slave.get('foo')
print(resp)
```

## 4. Redis 分片集群

### 4.1 搭建分片集群

分配集群解决的问题(主从、哨兵无法解决):
- 海量数据存储
- 高并发写

分片集群特征：
- 集群中有多个 master，每个 master 保存不同数据
- 每个 master 都可以有多个 slave 节点
- master 之间通过 ping 监测彼此健康状态
- 客户端请求可以访问集群任意节点，最终都会被转发到正确节点

创建：
```bash
redis-cli --cluster create --cluster-replicas 1 192.168.31.57:7001 192.168.31.57:7002 192.168.31.57:7003 192.168.31.57:8001 192.168.31.57:8002 192.168.31.57:8003
```
`--replicas 1`或者`--cluster-replicas 1` ：指定集群中每个master的副本个数为1，此时`节点总数 ÷ (replicas + 1)` 得到的就是master的数量。因此节点列表中的前n个就是master，其它节点都是slave节点，随机分配到不同master

查看：
```bash
redis-cli -p 7001 cluster nodes
```

连接集群：
```bash
redis-cli -c -p 7001
# -c  Enable cluster mode (follow -ASK and -MOVED redirections).
```

### 4.2 散列插槽

Redis 会把每一个 master 节点映射到 0~16383 共 16384 个插槽（hash slot）上。

**数据 key 不是与节点绑定，而是与插槽绑定。**

redis 会根据 key 的有效部分计算插槽值，分两种情况：
- key 中包含"{}"，且“{}”中至少包含1个字符，“{}”中的部分是有效部分
- key 中不包含“{}”，整个key都是有效部分

计算方式：利用 CRC16 算法得到一个 hash 值，然后对 16384 取余，得到的结果就是 slot 值。

#### 4.2.1 小结

Redis 如何判断某个key应该在哪个实例？

- 将 16384 个插槽分配到不同的实例
- 根据 key 的有效部分计算哈希值，对 16384 取余
- 余数作为插槽，寻找插槽所在实例即可

如何将同一类数据固定的保存在同一个Redis实例？

- 这一类数据使用相同的有效部分，例如 key 都以 {typeId} 为前缀

### 4.3 集群伸缩

#### 4.3.3 添加新节点到redis

```bash
redis-cli --cluster add-node
```

#### 4.3.4 转移插槽

```bash
redis-cli --cluster reshard
```

### 4.4 故障转移

#### 4.4.1 自动故障转移

当分片集群中一个 master 宕机：
- 该实例与其他实例失去连接
- 疑似宕机
- 确定下线，提升 slave 为 master
- master 重新上线后成为 slave

#### 4.4.2 手动故障转移

利用 `cluster failover` 命令可以手动让集群中的某个 master 宕机，切换到执行 `cluster failover` 命令的这个 slave 节点，实现无感知的数据迁移。

![手动故障转移](../images/image-20210725162441407.png)

这种 failover 命令可以指定三种模式：

- 缺省：默认的流程，如图 1~6 步
- force：省略了对 offset 的一致性校验
- takeover：直接执行第5歩，忽略数据一致性、忽略 master 状态和其它master的意见

### 4.5 redis-py 使用分片集群

```python
nodes = [
    redis.cluster.ClusterNode(host='192.168.31.57', port=7001),
    redis.cluster.ClusterNode(host='192.168.31.57', port=7002),
    redis.cluster.ClusterNode(host='192.168.31.57', port=7003),
    redis.cluster.ClusterNode(host='192.168.31.57', port=8001),
    redis.cluster.ClusterNode(host='192.168.31.57', port=8002),
    redis.cluster.ClusterNode(host='192.168.31.57', port=8003),
]
cluster = redis.cluster.RedisCluster(startup_nodes=nodes)
cluster.set('foo', 'bar')
resp = cluster.get('foo')
print(resp)
```