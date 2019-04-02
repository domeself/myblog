---
layout: post
title:  "spring environment"
categories: springframework
tags: spring
author: supernova
description: springframework 容器环境 .
---
## 实现
* StandardEnvironment 一般spring应用
* StandardServletEnvironment web应用
## profile
在spring容器中，通过prifile定义bean的逻辑分组。一个spring应用可以同时激活多个profile。  
应用场景： 环境部署(生产、测试、开发)  
在程序中可以通过configurableEnvironment，选择性激活profile。
```
setActiveProfiles(String...);
addActiveProfile(String);
setDefaultProfiles(String...);
```
## propertySources

## SystemEnvironment

## systemProperties

## merge

## ConfigFileApplicationListener
### propertiesSourceLoader
* PropertiesPropertySourceLoader  
加载格式：property、xml
* YamlPropertySourceLoader  
加载格式：yaml、yml
   