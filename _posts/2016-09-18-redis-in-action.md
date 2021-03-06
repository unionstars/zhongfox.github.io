---
layout: post
tags : [缓存, redis, nosql]
title: Redis实战干货
subtitle: "重剑无锋，大巧不工"
header-img: img/pic/2015/10/kanasi4.jpg

---


### expire 陷阱

执行类似zadd sadd hset等操作的时候，一定要先进行expire，如果miss(expire返回0表示不存在)则不执行，等到查询miss的时候在从db中load

如果先zadd, 在设置expire的话, 可能zadd前, 数据已经过期了, 再zadd后的缓存只有刚加入的数据, 造成缓存中数据不全

[缓存使用总结](http://lintanghui.com/2016/09/10/cache.html)

---

### redis 过期策略

* 定时删除: redis并不使用这种策略, 定时器对内存友好, 对cpu不友好

redis 结合使用:

* 惰性删除: CPU 友好, 内存不友好
* 定期删除: 是上面2中的折中 难点是确定删除操作执行的时长和频率

[Redis之过期键删除策略](http://blog.edagarli.com/2016/06/08/Redis%E4%B9%8B%E8%BF%87%E6%9C%9F%E9%94%AE%E5%88%A0%E9%99%A4%E7%AD%96%E7%95%A5/)

Redis是单线程的，基于事件驱动的，Redis中有个EventLoop, 有2类事件:

* IO事件
* 定时事件: 类似JavaScript的EventLoop

[Redis缓存失效机制](https://my.oschina.net/andylucc/blog/679222)

redis中大量的key的过期时间设置在同一时刻, 可能导致定期删除执行很久, 造成单线程的redis阻塞较长时间

所以，要善待redis数据，比如不用了就自己清理掉，不要等着redis来帮你; 或者把key的过期时间分散开

[善待Redis里的数据](http://neway6655.github.io/redis/2015/12/19/%E5%96%84%E5%BE%85Redis%E9%87%8C%E7%9A%84%E6%95%B0%E6%8D%AE.html)

---

### 持久化选择策略

#### Snapshot:

* 有2个命令可以主动save:
  * save 是由主进程进行快照操作，会阻塞其它请求;
  * bgsave 会通过 fork 子进程进行快照操作
* 开启: `save 时间内 写次数 时间内 写次数 ...` save  后接存储的时机 及时 在多少秒内发生多少次写, 就进行一次快照写入
* 关闭: `save ""`

其他配置:

    # 当snapshot 时出现错误无法继续时，是否阻塞客户端变更操作
    stop-writes-on-bgsave-error yes
    # 是否启用rdb文件压缩，默认为 yes cpu消耗，快速传输
    rdbcompression yes
    # rdb文件名称
    dbfilename dump.rdb

#### AOF

* 将 操作 + 数据 以格式化指令的方式追加到操作日志文件的尾部
* AOF Rewrite: 类似Snapshot, 遍历内存到aof文件, rewrite的触发机制是aof文件大小超过阈值
* 开启: `appendonly yes`
* 关闭: `appendonly no`

其他配置:

```
##只有在yes下，aof重写/文件同步等特性才会生效
appendonly no
##指定aof文件名称
appendfilename appendonly.aof
##指定aof操作中文件同步策略，有三个合法值：always everysec no，默认为everysec
appendfsync everysec
##在aof-rewrite期间，appendfsync 是否暂缓文件同步，no 表示不暂缓，yes 表示暂缓，默认为no
no-appendfsync-on-rewrite no
##aof文件rewrite触发的最小文件尺寸 只有大于此aof文件大于此尺寸是才会触发rewrite，默认64mb，建议512mb
auto-aof-rewrite-min-size 64mb
##相对于上一次rewrite，本次rewrite触发时aof文件应该增长的百分比
auto-aof-rewrite-percentage 100
```

#### 比较

* Snapshot: 数据可能缺失部分; copy-on-write造成极端情况内存是实际数据的2倍; db文件尺寸小; 恢复快
* AOF: AOF文件非常的庞大; 恢复时间长(用rewrite缓解); 更多的磁盘IO; 数据更完整
* 只要用户原因, 2种持久化可以同时开始

#### 选择:

* Snapshot 用于主从同步
* 通常 master 使用AOF，slave 使用 Snapshot，master 需要确保数据完整性，slave 提供只读服务
* 如果你的网络稳定性差， 物理环境糟糕情况下，那么 master， slave均采取 AOF，这个在 master， slave角色切换时，可以减少时间成本

[Redis 该选择哪种持久化配置](http://zheng-ji.info/blog/2016/03/10/gai-xuan-ze-na-chong-redischi-jiu-hua-pei-zhi/)

---

### 部署与扩展

* 业务垂直切分: 单redis 的0~15号逻辑库做业务切分, 适合低吞吐量, 高可用不高的业务组

* 业务水平切分: 分配对应单独物理机; 问题: 单redis只能利用单核, 不能充分利用多核, 整体可用性降低:

  1台99.9的可用性, 10台合并可用性降为99

* 10台机器10个分片, 20个实例, 互为主备, 好处提高了可用性, 多核还是没有很好利用

* 为了利用多核, 把redis和业务应用放一起, 问题: 应用和redis争抢cpu和内存, 死锁

* 解决方案: TODO

---

### 配置

* `> CONFIG GET * `  查看所有配置

* `dir` 是db文件的存储目录

---

### Redis 与网络流量整形

知识点:

* 令牌桶算法实现网络流量整形
* redis watch 乐观锁
* redis 事务: multi, execute

[令牌桶算法](http://baike.baidu.com/link?url=NP_yYC5SnzB2Z9vkfdx-8WRLlAR5I3YO47qzWOpVbamsQdmwd3vwacofBGxK3lpcUvmaV9AMufBS7rBrcHt77a)

* 用于 网络流量整形（Traffic Shaping）和速率限制（Rate Limiting）
* 能够限制数据的平均传输速率, 还允许某种程度的突发传输(令牌桶剩余令牌满足当前突发流量时)

过程:

1. 定期产生令牌: 令牌桶有固定最大容量, 超过容量, 令牌丢弃
2. 消耗令牌: 单位流量消耗单位令牌, 首先判断流量包需要的令牌是否足够, 足够发送, 不够的话无法发送
3. 令牌不足的流量, 考虑丢弃或者排队等待足够的令牌

[Redis 与网络流量整形](https://zhuanlan.zhihu.com/p/23624453)有详细的实现代码, 简单总结:

数据结构:

* 令牌桶的容量, 常量
* 令牌桶当前令牌数量`key: available`
* 上次流量通过时间戳(上次成功消耗流量的时间戳) `key: ts` , 用于计算当前消耗请求时间点和上次时间点之间, 需要补充的令牌数量(通过`key: available`)

  从数据结构可以看出, 令牌数量的增加类似一个`懒增加`, 不需要额外的定时任务去增加令牌, 额外的定时任务去维护`key: available` 可能会和消费者写冲突造成数据不一致.

大致过程:

1. watch `key: available`和`key: ts`, 可以看到此方案是一个串行消费, 和redis的单线程处理比较契合
2. 通过`key: ts`和当前时间, 计算需要补充的令牌, 算出总剩余令牌
3. 判断总剩余令牌是否满足此次消费需求, 剩下的就是if else, 然后更新以上2各值, 期间都是事务控制, 确保数据的一致性
4. 如果事务失败, 表示很可能其他消费者已经先于当前消费者更新了数据. 这时候重新来过(乐观锁)

---

### AOF定期重写会引起redis瓶颈

[由Redis客户端连接数大小说开去](http://www.jianshu.com/p/549d4555ae16)

解决思路是减少AOF重写的频率，两种方式:

* 让Redis决定是否做AOF重写操作，根据auto-aof-rewrite-percentage和auto-aof-rewrite-min-size两个参数
* 用crontab定时重写，命令是：BGREWRITEAOF
