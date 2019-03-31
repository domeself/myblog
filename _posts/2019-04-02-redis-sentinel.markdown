---
layout: post
title:  "sentinel"
categories: Redis
tags: Redis
author: supernova
description: sentinel、高可用、哨兵
---
## 三个定时任务 
* 获取redis从节点，确认主从关系  
每隔10s每个sentinel会向master节点进行info操作，获得主节点信息。
从主节点信息中获取从节点ip和端口，再向从节点发出info操作。
如果从节点也包含从节点，继续info操作知道获取全部节点的信息。
* sentinel彼此交互对其他sentisel节点的看法，和自身信息。  
每隔2秒每个sentinel通过sentinel的master节点的channel(_sentinel_:hello)和其他sentinel节点交换信息(pub/sub)
* 在sentinel集群和redis集群中检测心跳  
每1秒每个sentinel对其他sentinel和reids执行ping  
## 主观下线和客观下线
* 主观下线  
每个sentinel节点对redis节点下线的主观判断,由于网络原因或者自身的原因，会造成对redis节点状态的误判。
```
sentinel down-after-milliseconds masterName timeout
```
* 客观下线  
通过三个定时任务中的第二个任务，sentinel节点之间交换信息，来判定master节点是否真正的下线。
认为master节点下线的人数必须大于法定人数并且超过半数
```
sentinel monitor  masterName ip port  法定人数
```
## 故障转移
* sentinel选举
sentinel节点通过选举的方式，选出一个人去完成redis的故障转移。
    * 做了主观下线的sentinel向其他sentinel发出命令，请求将自己设置成领导者。
    * 收到命令的sentinel如果没有同意通过其他sentinel的命令，将同意请求，否则拒绝。  
    * 如果该sentinel发现投票数大于法定人数并且超过半数，那么他将成为领导者。
    * 如果多个sentinel成为领导者，则结果无效，重新选举。
* 故障转移
    * 从slave中选出一个合适的节点成为新的master节点。
    * 对上面的slave节点进行slaveof onone 命令，让其断开与所有节点的关系。
    * 向剩余的slave节点发送slaveof 命令，让它们成为新的master节点的从节点，并复制master的RDB。
    * 更新对原来的master节点为slave，并保持对它的关注，当其恢复后，命令它去master节点复制RDB数据。  
* slave选择(新的master节点)
    * 根据slave-priority(给个节点的配置文件中设置)值最高的节点。
    * 选择复制偏移量最大的节点(偏移量越大说明和master节点数据越接近)。
    * 选择run_id最小的节点(run_id 越小启动的越早，数据完整性最高)。