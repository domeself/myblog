---
layout: post
title:  "cluster 集群"
categories: Redis
tags: Redis
author: supernova
description: cluster、集群
---
## 分区
* 节点取余(hash值%分组数)
    * 客户端分区
    * 节点伸缩，分组变化，导致数据迁移
    * 迁移数量大，和添加的节点量有关，建议翻倍扩容
* 一致性哈希
    * 客户端分区
    * 只影响邻近节点，但是还有数据迁移。
    * 翻倍伸缩，保证最小迁移数和数据平衡。
* 虚拟槽分区
    * 每个槽映射一个数据子集，一般比节点数大。
    * 良好的hash函数，比如CRC16
    * 服务端分区，例如：Reids Cluster  
  
## 集群搭建  
### 原生安装  
* 配置  
按照单机配置即可，额外添加了2个集群配置相  

```
cluster-enable yes
cluster-config-file  nodes-{port}.conf
cluster-node-timeout 15000
cluster-require-full-converage  no //是否集群中只要有一个宕机，整个集群就停止服务 
```
* 启动  
正常启动即可  
* meet
```
reids-cli -h redis1 -p port1 cluster meet  redis2 prot2 
reids-cli -h redis1 -p port1 cluster meet  redis3 prot3
reids-cli -h redis1 -p port1 cluster meet  redis4 prot4
```
* 分配槽(16384个)
```
cluster addslots {slots...}
reids-cli -h redis1  -p port1 cluster addslot  xxx
reids-cli -h redis2 -p port2 cluster addslot  xxx
r
```
* 设置主从  
```
cluster replicate node-id
reids-cli -h redis2 -p port2 cluster replicate  redis1-node-id 
reids-cli -h redis4 -p port4 cluster replicate  redis3-node-id
```  

### 通过Ruby安装(推荐)
* 安装ruby
    * 下载
    wget https://cache.ruby-lang.org/pub/ruby/2.3/ruby-2.3.1.tar.gz
    * 安装
    tar -xvf ruby-2.3.1.tar.gz  
    ./configure -prefix=/usr/local/ruby  
    make  
    make install  
    cd /usr/local/ruby  
    cp bin/ruby/usr/local/bin
* 安装rubygem redis
    * 下载
    wget http://rubygems.org/downloads/redis-3.3.0.gem
    * 安装 redisgem  
    gem install -l redis-3.3.0.gem  
    gem list --check redis gem
    * 安装reids-trib.rb  
    cp ${redis_home}/src/reids-trib.rb /usr/local/bin    
* 搭建集群
    * 启动所有节点
    * 开启集群  
    ./redis-trib.rb create --replicas 1  redis1:port1 redis2:port2 redis3:prot3 redis4:port4
 ## 集群伸缩
 * 扩容
    * 启动一个新节点(孤立)
    * 加入集群  
    redis-trib.rb add-node new_host:new_port  exsitingip:exsitingport --slave master-id(如果是从节点，需要指定主节点) 
    * slot迁移 (手动)   
    1.cluster setslot {slot} importing {souceNodeId}//让目标节点准备导入槽的数据。  
    2.cluster setslot {slot} migrating {targetNodeId}//让源节点准备迁出槽的数据。  
    3.源节点循环执行 cluster getkeysinslot{slot} {count}命令，每次获取count个属于的槽的键。  
    4.在源节点上执行migrate {targetIp}{targetport} key 0 {timeport} 命令把指定key迁移。  
    5.重复执行3，4直到槽下所有的key数据迁移到目标节点。  
    6.向集群内所有的节点发送cluster setlot{lots} node{targetNodeId}，通知槽已经分配个了目标节点。
    * slot迁移(redis-trib.rb,推荐)  
    ./redis.trib.rb reshard  host:ip(集群中随便选一个)  
    ```
        1.how many slots do you want to move(from 1 to 16384)?  4096
        2.what is the receiveing node Id? xxxxxxxxxx
        3.source node:  all:表示从所有的node中迁移部分到新节点中。none:表示要自己填nodeId
    ```
 * 缩容
    * 迁移槽  
    reids-trib.rb reshared --from  nodeId1 --to nodeId2 --slots  count(槽的个数)  ip:port(在哪个节点上执行)
    * 脱离集群并下线
    reids-trib.rb del-node ip:port(在那个节点执行)  nodeId  

## 客户端访问

### moved error
客户端向一个节点发起对key的操作命令，这个节点先对这个key进行hash计算，如果这个key正好落在这个节点，  
则执行命令并返回结果。如果不在本节点，则返回一个moved error和命中的节点ip、port。  
客户端收到后再次向正确的节点发送命令。  

### ask重定向
客户端发送对key的操作命令，如果这个key所在的节点正在进行数据的迁移。则会返回ask重定向的error。
客户端需要向转移后的节点发起asking命令，并执行key的操作，返回操作结果。  

### smart客户端(Jedis Cluster)  
* 步骤
    * 1.从cluster中选择一个node，进行cluster slot命令，获取所有节点和slot的对应关系。
    * 2.将节点和slot的映射关系，保存在本地并为每一个节点，准备一个Jedispool。
    * 3.客户端接受请求时，对key进行CRC16获得hash值%16383得到slot。根据映射关系从连接池中获取连接。
    * 4.向这个连接发送正常的请求，如果请求不成功可能映射关系有变动。随机向另外的一个节点，发送请求。  
    * 5.如果返回moved error和 key所在的ip，说明映射关系的确有变动。执行1的操作重新更新本地的映射缓存。
    * 6.如果连续5次都不成功，则抛出Too many clouster replication ！异常。
* 集群环境下的批量操作(mget、mset)
    * 串行mget、mset  
    循环执行每条命令
    * 串行IO  
    根据本地的node-slot缓存映射，按node分组执行。
    * 并行IO  
    将串行IO的node分组利用多线程，并行执行。
    * hash_tag  
    hash计算时，为了让相关的key落到同一个节点上，不用真正的key，而是采取key的一个相同部分{}替代key去计算。如：age{name}、sex{name}
    

|方案|优点|缺点|网络IO|
|:---:|:---:|:---:|:---:|
|串行mget|编程简单，少量key满足需求|大量key请求延迟严重|O(keys)|
|串行IO|编程简单，少量key满足需求|大量node请求延迟严重|O(nodes)|
|并行IO|延迟取决于最慢的节点|多线程编程复杂，问题定位困难|O(slow node)|
|hash_tag|性能最高|读写增加维护tag的成本，tag分布已出现倾斜|O(1)|

## 故障发现 
* 通过ping/pong的消息模式实现故障发现，不需要sentinel。
* 主观下线  
某个节点认为另一个节点不可用，带有’偏见‘
节点1发送ping信息给节点2，如果ping通，节点2则会回复一条pong信息给节点1，节点1记录最后一次与节点2的通信时间。  
如果ping失败则会断开与节点2的连接。在下次触发ping消息的时候，依然会进行ping/pong操作，如果发现与节点2的最后一次通信时间  
大于node-timeout时间 则会间节点2的状态置为pfail，为主动下线。
* 客观下线  
半数以上持有槽的主节点都标记某个节点主观下线。  
1.一个节点通过在ping/pong信息交换过程中，搜集了其他节点对别的节点pfail看法，放在自身故障列表中。  
故障列表保存的数据周期时node-timeout的2倍，保证较早前的故障信息不会被作为下线依据。  
2.计算有效下线报告数据，是否大于持有槽的主节点数量的一半。  
3.没有大于一半则退出。
4.大于一半则更新为客户端下线，向集群广播下线节点的fail消息。通知故障节点的从节点准备成为主节点。

## 故障恢复
* 资格检查
子节点检查与故障节点最后通信的时间。如果大于cluster-node-timeout(15m) * cluster-slave-validity-factory(10),则失去成为主节点资格。
* 准备选举时间
让偏移量最大的节点获得最小的选举准备时间，让它更早的发起投票，提高被选举成功的概率。
* 选举投票
从节点在准备时间结束之后，开始请求投票，主节点选择肯定或者拒绝。获得(主节点数/2)+1的的票方可成为主节点。  
* 替换主节点  
    * 从节点执行 slaveof no one，清楚复制的数据和之前的主从关系。  
    * 执行clusterDelslot撤销故障主节点负责的slot，分配个自己。
    * 向集群广播自己的pong，表明已经替换了主节点。

## 常见运维问题
* 集群完整性  
cluster-require-full-coverage = no  
默认值yes，表示只要有一个节点下线，整个集群就不可用。
* 带宽消耗  
集群内节点通信PING/PONG，协议为Gossip，官方要求节点数小于1000.如果集群的规模很大，对带宽的要求也是非常大的。尽量避免使用一个集群，  
如果业务很大，可以拆分成多个集群。
* 集群倾斜
    * 数据倾斜，内存不均  
    1.节点的槽分配不均  
    2.不同槽对应的键值数差异大
    3.包含bigkey，一个key的集合存了上百万条数据  
    4.内存配置不一致
    * 请求倾斜，热点key
* 读写分离  
集群下，从节点只做替补，不负责读操作。 读写分离会面临数据一致性、数据过期、节点故障等一系列问题。  
官方并没有提出解决方案，可能也是因为redis集群的性能已经足够使用了。
* 数据迁移(线上、线下)
    * 官方迁移工具 redis-trib.rb import
        * 只能从单机迁移到集群
        * 不支持断点续传
        * 单线程迁移速度慢
        * 不支持在线迁移
    * 线上迁移
        * 唯品会redis-migrate-tool
        * 豌豆荚redis-port
* 限制
    * 批量操作受限mset、mget只支持同一个槽
    * key事务和lua支持有限，操作key必须在一个节点
    * key是数据最小分区，不支持bigkey分区
    * 不支持多数据库，只有一个db 0
    * 复制只支持一层，不支持树形结构
* 分布式集群不一定好
    * 客户端性能会降低(相对)，因为数据分散在各个节点。
    * 很多命令不支持
    * key事务和lua支持有限，不能跨节点
    * 客户端(SDK)维护复杂
    * redis sentinel 已经足够好
    * 业务需要不一定需要，因为单点的QPS已经接近10万了。