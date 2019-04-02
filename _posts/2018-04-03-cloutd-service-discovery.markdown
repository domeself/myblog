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
register-with-eureka: true #不需要作为注册中心的客户端
fetch-registry: true #取消向注册中心获取信息
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
        defaultZone: http://EurekaServer2/eureka/,http://EurekaServer2/eureka3/
```

* EurekaServer2的配置

```
server:
  port: 8761
eureka:
  instance:
    hostname: eurekaServer2
  server:
    enable-self-preservation: false # 关闭自我保护模式（默认为打开）
    eviction-interval-timer-in-ms: 5000  # 续期时间，即扫描失效服务的间隔时间（缺省为60*1000ms）
  client:
      register-with-eureka: true #不需要作为注册中心的客户端
      fetch-registry: true #取消向注册中心获取信息
      service-url:
        defaultZone: http://EurekaServer1/eureka/,http://EurekaServer2/eureka3/
```

* EurekaServer3的配置

```

server:
  port: 8761
eureka:
  instance:
    hostname: erekaServer3
  server:
    enable-self-preservation: false # 关闭自我保护模式（默认为打开）
    eviction-interval-timer-in-ms: 5000  # 续期时间，即扫描失效服务的间隔时间（缺省为60*1000ms）
  client:
      register-with-eureka: true #不需要作为注册中心的客户端
      fetch-registry: true #取消向注册中心获取信息
      service-url:
        defaultZone: http://EurekaServer1/eureka/,http://EurekaServer2/eureka3/

```

EurekaClient端的修改  

```
service-url:
        defaultZone: http://EurekaServer1/eureka/,http://EurekaServer2/eureka/,http://EurekaServer3/eureka/
```
  
## 服务发现 
服务发现是由eurekaClient去完成

```
@Autowired
private EurekaDiscoveryClient client;

@Bean
    ApplicationRunner runner()  {
        return args -> {
            client.getServices().forEach(System.out::println); //所有的服务名
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