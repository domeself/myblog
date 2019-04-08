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

## 服务提供端
* 接口定义

```
@RequestMapping("/email")
@FeignClient(serviceId = "MESSAGE-SERVICE",qualifier = "email",fallback = EmailServiceFallBack.class)
public interface IEmailClient{
@GetMapping("/sendVerificationCode")
    ResultEntity sendVerificationCode(@RequestParam("templetCode")String templetCode, @RequestParam("identification")String identification);
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

* 接口实现

```
@RestController
public class EmailServieController implements IEmailClient {
 @Override
    public ResultEntity sendVerificationCode(String templetCode, String identification) {
            xxx....
            return resultEntity;
    }
}
```

* 服务降级

```
@Component
@RequestMapping("/fallback/email")
public class EmailServiceFallBack implements IEmailClient {


    @Override
    public ResultEntity<MidSms> sendVerificationCode(String templetCode, String identification){
        return new ResultEntityFallBack();
    }
}

```

## 接口调用

```
@RestController
public class TestController implements TestInterface {
    public static Logger log = LoggerFactory.getLogger(TestController.class);
    @Autowired
    @Qualifier("email")
    private IEmailClient emailClient;
}

```
 


