---
layout: post
title:  "声明式服务调用(feign)"
categories: springCloud
tags: springCloud
author: supernova
description: 声明式服务调用.
---
## Feign的组成
前面介绍了Ribbon和Hystrix，在微服务架构中，这两者几乎是组合应用，既然如此不如把他们整合在一起，  
所以feign就出现了，它不仅整合了Ribbon与Hystrix，而且提供了一种新的更加灵活的调用方式：声明式的web客户端。
  

## 引入Feign
* 依赖

```
        <dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-feign</artifactId>
		</dependency>
```

* 开启
@ EnableFeignClients

## 使用方式
* 接口定义

```
@FeignClients("hello-service") //服务名称，不区分大小写
public interface HelloService{
@RequestMapping("/hello")
String hello();
}
```

* 接口调用

```
@Autowired
HelloService helloService;

public String hello(){

return helloService.hello();
}
```

* 参数绑定  
上面展示的是最简单的示例，并没有方法参数，但是一般接口调用几乎都是有参数的。

 * 基本类型参数  
```
@RequestMapping(value="/hello",method=RequestMethod.GET)
public String hello(@RequestPram String name){
    return "hello "+ name;
}
```

 * 请求头参数

```
@RequestMapping(value="/hello",method=RequestMethod.GET)
public String hello(@RequestHeader String name){
    return "hello "+ name;
}

```

 * 对象

```
@RequestMapping(value="/hello",method=RequestMethod.POST)
public String hello(@RequestBody User user){
    return "hello "+ name.getName();
}
```

对象必须要有默认的构造方法，不然，在对象转JSON字符串的时候会报错。  
而且参数有对象一定要用POST方法 


 


