---
layout: post
title:  "服务发现(service-discoveery)"
categories: springCloud
tags: springCloud
author: supernova
description: 通过注册中心，服务自动注册和发现 .
---
## 注册中心
### 常见的服务注册中心：
* Apache Zookeeper  
高一致行，可用性低  
* Netflix Eureka  
高可用性，状态可能不一致  
* Consul  
高一致性、高可用性(相对而言)  
特点：
注册中心+配置中心  
去中心化(所有的节点即是注册中心，也是客户端)  

### 高可用性  
通常来描述一个系统经过专门的设计，从而减少停工时间，而保持其服务的高度可用性。  
    * 平均无故障时间（MTTF）  
    计算机系统平均能够正常运行多长时间，才发生一次故障  
    * 平均维修时间（MTTR）  
    系统发生故障后维修和重新恢复正常运行平均花费的时间。
高可用性=MTTF/(MTTF+MTTR) * 100%  
![高可用等级]({{ "/assets/img/highlevel.png" | absolute_url }})  



## Netflix Eureka
### 服务端Eureka Server
Eureka Server是Euraka Client的注册服务中心、管理所有注册服务、实例信息和状态。  
运行Eureka Server：
依赖：

```
<dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
        </dependency>
```

激活：@EnableEurekaServer
### 服务注册
### 服务发现
   