---
layout: post
title:  "Ribbon 负载均衡器"
categories: springCloud
tags: springCloud
author: supernova
description: 客户端负载均衡
---
## 服务端与客户端负载均衡对比
* 服务端负载均衡
服务端的负载均衡分为硬件和软件2种  
硬件负载均衡是指在各个节点之间安装硬件设备，比较有名的就是F5。  
软件负载均衡是指专门用一台或者多台部署了负载均衡的软件进行请求的转发，比如Ngnix。
服务端负载均衡的原理就是，在负载均衡器上维护一份可用服务的清单。  
每隔一段时间会通过心跳的方式检测服务的可用性，如果不可用就踢出可用列表。每隔若干时间，还会对不可用的服务进行检查，  
如果已经恢复可用，在加入到可用列表。  
每当一个请求过来，负载均衡器会按照规则(轮询[循环顺序调用])、随机([生成ramdom随机数])、权重([加权因子])、一致性哈希)等  
获得一个服务进行转发。

* 客户端负载均衡
与服务端负载均衡不用，在微服务的客户端中就已经本地缓存了服务提供者的信息。有了这份名单客户端就可以自主选择去调用哪个服务实例。

## restTemplet
这里只对restTemplet进行一个简单的介绍。  
restTemplet是spring对http请求restful风格的封装。他的实现有3种：  
* httpclient
* okHttp3
* 如果上述2个都没有引入，就会选择jdk的HttpURLConnection
普通restTemplet使用

```
        RestTemplate restTemplate = new RestTemplate();
        ResponseEntity<ResultEntity> forEntity = restTemplate.getForEntity("http://127.0.0.1:8762/member", ResultEntity.class);
        System.out.println(forEntity.getBody().getData());
```

在微服务客户端开发过程中，我们是不知道服务的ip地址和端口的，只知道服务名，那么我们如何调用？  
将ribbon与restTemplet结合可以做到更加服务名调用

## 引入ribbon
* 依赖

```
    <dependency>
		<groupId>org.springframework.cloud</groupId>
		<artifactId>spring-cloud-starter-ribbon</artifactId>
	</dependency>
```

* 结合restTemplate
配置restTemplateBean的时候，用@LoadBalanced注解  

```
    @Bean
    @LoadBalanced
    RestTemplate restTemplate() {
        return new RestTemplate();
    }
```

## 服务调用
在服务发现中，我们介绍了客户端是保存了服务提供者的缓存。
ribbon会根据服务名从本地缓存中找到对应的多个实例选择一个调用。ribbon会默认采用轮询的方式循环调用，从而实现负载均衡。
通过ribbon调用

```

            ResponseEntity<ResultEntity> forEntity = template.getForEntity("http://MEMSERVICE-SERVICE/member", ResultEntity.class);
            System.out.println("返回=="+forEntity.getStatusCode());
        
    }
``` 

ribbon是怎么将MEMSERVICE-SERVICE转成IP地址的呢？
和之前普通调用唯一的区别就是在创建RestTemplate的时候用了@LoadBalanced注解。
查看@LoadBalanced注解

```
@Qualifier
public @interface LoadBalanced {
}
```
@LoadBalanced只是一个引用了@Qualifier的标记注解，@Qualifier的作用就是在和@Autowired结合使用，  
@Autowired注入的对象是按照对象类型匹配，如果ioc容器里有多个实列，程序不能确定注入哪个就会发生异常。就需要结合@Qualifier指定注入哪个。  
所以我们全局搜索@LoadBalanced,发现org.springframework.cloud.client.loadbalancer.LoadBalancerAutoConfiguration中刚好是这样使用的。

```
public class LoadBalancerAutoConfiguration {

	@LoadBalanced
	@Autowired(required = false)
	private List<RestTemplate> restTemplates = Collections.emptyList();

	@Bean
	public SmartInitializingSingleton loadBalancedRestTemplateInitializerDeprecated(
			final ObjectProvider<List<RestTemplateCustomizer>> restTemplateCustomizers) {
		return () -> restTemplateCustomizers.ifAvailable(customizers -> {
            for (RestTemplate restTemplate : LoadBalancerAutoConfiguration.this.restTemplates) {
                for (RestTemplateCustomizer customizer : customizers) {
                    customizer.customize(restTemplate);
                }
            }
        });
	}

```

很明显这里将标记了@LoadBalanced的RestTemplate，注入到了restTemplates中。再继续看其他方法：  

```
@Bean
		public LoadBalancerInterceptor ribbonInterceptor(
				LoadBalancerClient loadBalancerClient,
				LoadBalancerRequestFactory requestFactory) {
			return new LoadBalancerInterceptor(loadBalancerClient, requestFactory);
		}
```

配置了一个LoadBalancerInterceptor拦截器，再继续看  

```
@Bean
		@ConditionalOnMissingBean
		public RestTemplateCustomizer restTemplateCustomizer(
				final LoadBalancerInterceptor loadBalancerInterceptor) {
			return restTemplate -> {
                List<ClientHttpRequestInterceptor> list = new ArrayList<>(
                        restTemplate.getInterceptors());
                list.add(loadBalancerInterceptor);
                restTemplate.setInterceptors(list);
            };
		}
```

把loadBalancerInterceptor添加到了restTemplate中。应该就是这个拦截器实现了ribbon的功能，继续看LoadBalancerInterceptor  

```
@Override
	public ClientHttpResponse intercept(final HttpRequest request, final byte[] body,
			final ClientHttpRequestExecution execution) throws IOException {
		final URI originalUri = request.getURI();
		String serviceName = originalUri.getHost();
		Assert.state(serviceName != null, "Request URI does not contain a valid hostname: " + originalUri);
		return this.loadBalancer.execute(serviceName, requestFactory.createRequest(request, body, execution));
	}

```
继续debug


eurekaServer有个region和zone的概念，一个region对应多个zone。所以一个服务客户端会被注册到某个region的zone中，  
当访问时会优先从同个zone中找。
region和zone作用可能是按照区域部署，设计容错集群。

```
eureka.client.region //配置region
eureka.client.availabiility.zones //配置zones
```

