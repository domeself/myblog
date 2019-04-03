---
layout: post
title:  "服务治理"
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
* 依赖：

```
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
        </dependency>
```

* 激活：@EnableEurekaServer
* 配置

```
server:
  port: 8761
eureka:
  instance:
    hostname: localhost
  server:
    enable-self-preservation: false # 关闭自我保护模式（默认为打开）
    eviction-interval-timer-in-ms: 5000  # 续期时间，即扫描失效服务的间隔时间（缺省为60*1000ms）
  client:
      register-with-eureka: false #不需要作为注册中心的客户端
      fetch-registry: false #取消向注册中心获取信息
      service-url:
        defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
```

### 客户端Eureka Client
客户端为当前服务提供注册、同步、查找服务以及其实例信息和状态。  
运行Euraka Client
* 依赖：

```
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
```

* 激活：@EnableEurekaClient或者@EnableDiscoveryClient
* 配置

```
eureka:
  instance:
      lease-renewal-interval-in-seconds: 5      # 心跳时间，即服务续约间隔时间（缺省为30s）
      lease-expiration-duration-in-seconds: 15  # 发呆时间，即服务续约到期时间（缺省为90s）
  client:
      registry-fetch-interval-seconds: 10 # 拉取服务注册信息间隔（缺省为30s）
      service-url:
        defaultZone: http://192.168.1.152:8761/eureka/
      healthcheck:
        enabled: true # 开启健康检查（依赖spring-boot-starter-actuator）
```

### EurekaServer高可用配置
在微服务架构这样的分布式环境中，需要将注册中心高可用部署。
做法就是让EurekaServer相互注册。
以三台EurekaServer高可用为例：ip:port为别EurekaServer1，EurekaServer2，EurekaServer3.
此时我们就需要打开注册功能了

```
register-with-eureka: true #需要作为注册中心的客户端
fetch-registry: true #向注册中心获取信息
```

* EurekaServer1的配置

```
server:
  port: 8761
eureka:
  instance:
    hostname: eurekaServer1
  server:
    enable-self-preservation: false # 关闭自我保护模式（默认为打开）
    eviction-interval-timer-in-ms: 5000  # 续期时间，即扫描失效服务的间隔时间（缺省为60*1000ms）
  client:
      register-with-eureka: true #不需要作为注册中心的客户端
      fetch-registry: true #取消向注册中心获取信息
      service-url:
        defaultZone: http://EurekaServer2/eureka/,http://EurekaServer3/eureka/
```

* EurekaServer2的配置

```
server:
  port: 8762
eureka:
  instance:
    hostname: eurekaServer2
  server:
    enable-self-preservation: false # 关闭自我保护模式（默认为打开）
    eviction-interval-timer-in-ms: 5000  # 续期时间，即扫描失效服务的间隔时间（缺省为60*1000ms）
  client:
      register-with-eureka: true #需要作为注册中心的客户端
      fetch-registry: true #向注册中心获取信息
      service-url:
        defaultZone: http://EurekaServer1/eureka/,http://EurekaServer2/eureka/
```

* EurekaServer3的配置

```

server:
  port: 8763
eureka:
  instance:
    hostname: erekaServer3
  server:
    enable-self-preservation: false # 关闭自我保护模式（默认为打开）
    eviction-interval-timer-in-ms: 5000  # 续期时间，即扫描失效服务的间隔时间（缺省为60*1000ms）
  client:
      register-with-eureka: true #需要作为注册中心的客户端
      fetch-registry: true #向注册中心获取信息
      service-url:
        defaultZone: http://EurekaServer1/eureka/,http://EurekaServer2/eureka/

```

EurekaClient端的修改  

```
service-url:
        defaultZone: http://EurekaServer1/eureka/,http://EurekaServer2/eureka/,http://EurekaServer3/eureka/
```

## 服务注册
eurekaClient在启动的时候会通过发送rest请求的方式，将自己注册到eurekaServer，同时带上了自身服务的一些元数据。  
eurekaServer将元数据储存在一个双层Map中，第一层的key是服务名，第二层的key是实例名。
## 服务同步  
如果是高可用的eurekaServer集群，可能相同的2个服务实例被注册到了不同的eurekaServer上。  
因为eurekaServer也是相互注册的，当eurekaClient注册到一个eurekaServer上时，eurekaServer会将该请求转发给其他节点。
## 服务续约(renew)
eurekaClient会维护一个心跳持续告诉eurekaServer自己还活着，防止被eurekaServer的"剔除任务"剔除。

```
eureka:
  instance:
      lease-renewal-interval-in-seconds: 5      # 心跳时间，即服务续约间隔时间（缺省为30s），定义服务续约任务的调用间隔
      lease-expiration-duration-in-seconds: 15  # 发呆时间，即服务续约到期时间（缺省为90s），定义服务失效的时间
```

  
## 服务发现 
服务发现是由eurekaClient去完成，eurekaClient在启动的时候会通过发送rest请求的方式，从eurekaServer获得服务清单。 
客户端缓存该份清单，每隔30秒更新一次。
通过eureka.client.registry-fetch-interval-seconds: 10可修改更新间隔。


```
@Autowired
private EurekaDiscoveryClient client;

@Bean
    ApplicationRunner runner()  {
        return args -> {
            client.getServices().forEach(t->System.out.println("服务名："+t)); //所有的服务名
            //获取一个服务的所有可用实例
            client.getInstances("MEMBER-SERVICE").stream().forEach(
                   t-> {
                       System.out.println(t.getHost()+":"+t.getPort()+"instandceId="+t.getInstanceId()+"uri="+t.getUri()+"scheme="+t.getScheme());
                       t.getMetadata().forEach((key,value)->{
                           System.out.println("key="+key+"="+value);
                       });
                   }
            );
        };
```

结果打印 
 
```
服务名：member-service

DESKTOP-A1N9C51:8762instandceId=nulluri=http://DESKTOP-A1N9C51:8762scheme=null
key=management.port=8762
USER-20170717OB:8762instandceId=nulluri=http://USER-20170717OB:8762scheme=null
key=management.port=8762
LAPTOP-LIUQN:8762instandceId=nulluri=http://LAPTOP-LIUQN:8762scheme=null
key=management.port=8762
key=jmx.port=49170
```

## 服务下线
当程序正常的关闭或者重启时，客户端会发送一个rest请求给一个eurekaServr，  
eurekaServr会将该服务置为下线状态，广播给其他的eurekaServr。  
