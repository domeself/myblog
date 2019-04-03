---
layout: post
title:  "服务错误保护(Hystrix)"
categories: springCloud
tags: springCloud
author: supernova
description: 服务错误保护.
---
## 服务错误
在微服务架构中，系统被拆分成了很多个服务单元，各单元间同过服务注册和消息订阅的方式相互依赖。  
各单元之间的调用都是通过网络通信，网络通信不可避免的会出现故障、延迟，这又会直接导致调用该服务的服务
也会出现延迟。然而请求源源不断的过来，堵塞会越来越严重，最终导致整个网络瘫痪。
Spring Cloud Hystrix 提供了断路器、线程隔离等一系列的解决方案。  
@HystrixCommand使用了设计模式当中的命令模式，经典的命令模式包括4个角色：  
* Command：定义命令的统一接口
* ConcreteCommand：Command接口的实现者
* Receiver：命令的实际执行者
* Invoker：命令的请求者，是命令模式中最重要的角色。这个角色用来对各个命令进行控制。  

## 引入Hystrix
* 依赖

```
        <dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-hystrix</artifactId>
		</dependency>
```

* 开启
@ EnableCircuitBreaker

## 使用方式
### 注解模式

####同步调用
```
        @Autowired
        RestTemplate restTemplate;

        @HystrixCommand(fallbackMethod = "fallback")
        public String consumer() {
            return restTemplate.getForObject("http://eureka-client/dc", String.class);
        }

        public String fallback() {
            return "fallback";
        }
```



#### 异步调用

```
        @Autowired
        RestTemplate restTemplate;

        @HystrixCommand(fallbackMethod = "fallback")
        public Future<String> consumer() {
            return  new AsynResult<String>{
                @override
                public String invoke(){
                    return restTemplate.getForObject("http://eureka-client/dc", String.class);
                }
            }
            
        }

        public String fallback() {
            return "fallback";
        }
```

### 编程模式
#### 定义

```
public class UserCommand extends HystrixCommand<Object> {
    private RestTemplate restTemplate;
    private long id;
    
    public UserCommand(Setter setter, RestTemplate restTemplate,Long id){
        super(setter);
        this.restTemplate = restTemplate;
        this.id = id;
    }
    @Override
    protected Object run() throws Exception {
        return restTemplate.getForObject("http://USER-SERVICE/user/{1}",Object.class,id);
    }

    @Override
    protected Object getFallback() {
        return new Object();
    }
}
```

#### 同步调用
Object user =      new UserCommand(HystrixCommandGroupKey.Factory.asKey("gruop"),new RestTemplate(),1).execute();

#### 异步调用
Future<Object> user =     new UserCommand(HystrixCommandGroupKey.Factory.asKey("gruop"),new RestTemplate(),1).queue();




无论是restTemplate无法访问到请求的服务，还是testHystrix方法本身>2000毫秒(默认配置参数)，程序都会去调用fallbackMethod指定的  
testHystrixFallback方法返回"error";



 


