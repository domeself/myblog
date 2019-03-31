---
layout: post
title:  "redis 持久化"
categories: Redis
tags: Redis
author: supernova
description: redis持久化、persistence
---
## 持久化方式  
* 快照(RDB二进制文件)  
    * 三种写入策略
        * save(同步)  
    同步过程操作redis客户端操作阻塞。  
    生成新的RDB文件，结束之后才替换旧的RDB文件。  
    复杂度O(n)
        * bgsave(异步)  
    创建子进程，执行fork()操作，生成RDB文件，结束之后才替换旧的RDB文件。
    此时redis客户端仍然可用，但是fork()可能阻塞，即使 fork操作相对同步来说较快。  
   复杂度O(n)，消耗更多内存。
        * auto(自动)  
   根据配置规则，调用bgsave完成异步操作。
   save 900 1  900秒1条操作
   save 300 10  300秒10条操作
   save 60   5  60秒5条操作
   缺点不可控，一般不使用。    
    * 触发机制
        * 全量赋值
        * debug reload
        * shutdown  
    * 缺点  
        * 耗时，耗性能(io)  
        * 不可控，丢失数据
* 日志(AOF)  
       
    * 三种写入策略
        * always    执行每条命令都会被从缓冲区fsync写入到AOF文件中。
        * everysec  每秒把缓冲区fsync到磁盘中的AOF文件中。这是默认的策略。  
        * no    操作系统自己决定。  
        
        |命令 | always | everysec |no|
        | :---:| :---: | :---: |:---:|
        | 优点 | 不丢失数据 | 只丢失一秒数据 |不用管|
        | 缺点 | IO开销大 | 丢失一秒数据 |不可控|
          
