---
layout: post
title:  "Ribbon 负载均衡器"
categories: springCloud
tags: Ribbon
author: supernova
description: 客户端负载均衡
---
## 开启
* 依赖

```
    <dependency>
		<groupId>org.springframework.cloud</groupId>
		<artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
	</dependency>
```

* 开启 
配置restTemplate的时候，用@LoadBalanced注解  

```
    @Bean
    @LoadBalanced
    RestTemplate restTemplate() {
        return new RestTemplate();
    }
```
