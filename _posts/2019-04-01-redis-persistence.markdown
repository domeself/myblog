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
        
       * 配置
       ```
                appendonly yes  开启AOF
                appendfilename  xx.aof  文件保存路径
                appendfsync everysec  策略选择
                no-appendfsync-no-rewrite  no  是否在重写的时候，关闭正常aof
        
      ```
    * AOF重写，减少磁盘占用量，加速恢复速度。
        * 命令合并
        * 只保留影响数据结果的操作
        * 过期数据删除   
       
    * 重写方式
        * bgrewriteaof，客户端发送命令，启动fork进程去完成重写，类似resave。
        * 按照配置自动执行
        ```
            auto-aof-rewrite-min-size  64mb aof文件重写需要的大小
            auto-aof-rewrite-percentage 100 aof文件增长率
        ```
    * 使用策略
    
    |命令|RDB|AOF|
    |:---:|:---:|:---:|     
    |启动优先级|底|高|
    |体积|小|大|
    |恢复速度|快|慢|
    |数据安全性|丢数据|根据策略决定|
    |轻重|重|轻|
    