---
layout: post
title:  "Redis 主从复制"
categories: Redis
tags: Redis
author: supernova
description: 主从复制、master、slave
---
## 主从复制  
* 作用
    * 读写分离，减小master节点的压力。
    * 数据备份，master节点宕机时，将slave节点升级为master节点。
* 方式
    * slaveof  
        * 建立主从关系  
    从slave节点上执行slaveof命令从master节点上复制数据。建立主从关系成功后，slave节点之前的数据会被删除。
        * 取消主从关系  
     从slave节点上执行slaveof no one，断开与master节点联系。
        * 查看主从关系  
        info replication
    * 配置文件
    ```
        slaveof ip port
        slaveof-read-only yes 从节点只做读的操作
    ```

## 全量复制
* 名词说明
    * run_id  
    每次redis启动时会分配一个随机的run_id。
    查看：redis-cli  info server
    * 偏移量 repl_offset
    查看：redis-cli info replication
* 全量复制流程  
1.slave节点请求全量复制。  
2.执行bgsave，生成RDB文件。  
3.通过网络传输给slave节点。  
4.从节点清空数据，flushall。  
5.加载从master节点复制的RDB文件到内存中。  
6.可能的AOF重写。  

## 部分复制
* 部分复制流程
master节点在执行写操作的时候，会将命令写往复制积压缓冲区的队列中。
1.slave节点会向master节点发送pysnc{offset}{run_id}请求。  
2.如果offset不在缓存队列中（buffer默认1MB），说明错过了很多数据，master节点会选择全量复制。  
3.如果offset在婚车队列中，master会返回continue，并把从offset到最新的数据返回给slave，就是部分复制。 
## 运维
* 读写分离
master执行write操作，slave执行read操作。
    * 复制延迟，存在主从一致的问题  
    master异步复制数据给slave存在时间差。  
    slave阻塞，延迟接收到master的数据。
    * 读到过期的数据  
        redis删除过期数据有2中策略   
        1.懒惰性策略：在key被执行的时候，判断是否过期，过期删除返回给客户端null  
        2.定时采样：每隔一段时间检测一批key有没有过期。当数量非常多，采样速率跟不上key的产生速率。造成很多数据没有删除。
        slave节点不能删除数据，可能读到脏的数据。  
    * 从节点故障
* 主从配置不一致
    * 内存分配不一致，导致从节点数据丢失。
    * 主从优化参数一致，产生诡异问题。
* 规避全量复制
    * 第一次复制不可避免。
        * 主节点分配小一点。
        * 低峰的时候进行全量复制。
    * 节点运行run_id不匹配
        * 利用故障转移，哨兵或者集群解决。  
    * 复制积压缓冲区不足  
        * 网络中断，全量复制。
        * 修改默认值 rel_backlog_size
* 复制风暴
    * 单节点  
    master节点挂了很多slave节点。当master重启时，产生大量全量复制。
    * 单机器  
    一台机器上部署了多台master节点，当master重启时，产生大量全量复制。采用master分散部署或者高可用。

