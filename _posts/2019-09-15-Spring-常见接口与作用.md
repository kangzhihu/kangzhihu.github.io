---
layout: post
title: "Spring-常见接口与作用"
subtitle: 'Spring介绍五'
author: "Kang"
date: 2019-09-15 14:53:57
header-img: "img/post-head-img/copy-2518265_1280.jpg"
catalog: true
tags:
  - Spring
  - 实例化
---
### Aware
&emsp;&emsp;Aware是感知的意思，在Spring中一般我们是不需要关注所使用的Bean的状态的，XxxAware是在框架内部使用的，当实现后，容器将会将特定的内容传递给该Bean，这些Bean将携带这些信息出来(即Bean感知到了这下信息)。  

#### 常见Aware子类
- BeanNameAware：将Spring配置文件中Bean的id值传递给实现了该接口的Bean。  
- BeanFactoryAware：将Spring工厂自身传递给实现了该接口的Bean。   
- ApplicationContextAware：将Spring上下文传递给实现了该接口的Bean。   


### BeanPostProcessor
&emsp;&emsp;在Spring容器中完成bean实例化、配置以及其他初始化方法前后要添加一些自己逻辑处理。我们需要定义一个或多个BeanPostProcessor接口实现类，将生成的BeanWrapper传递出来做处理，然后再注册到Spring IoC容器中。  


### NamespaceHandler
&emsp;&emsp;我们在使用Spring中不同的功能的时候可能会引入不同的命名空间比如xmlns:context，xmlns:aop，xmlns:tx等等。在Spring中定义了一个这样的抽象类专门用来解析不同的命名空间。这个类是NamespaceHandler。     
&emsp;&emsp;在xml中若我们有aop配置项：   
```xml
<aop:aspectj-autoproxy/>
```
则在解析Bean定义的时候，将自动使用AopNamespaceHandler#init中注册的对应处理子类--AspectJAutoProxyBeanDefinitionParser来完成xml的解析处理，将解析内容存入BeanDefinition。  

##### AnnotationAwareAspectJAutoProxyCreator
&emsp;&emsp;之所以要单独拿出来说，是应为其findCandidateAdvisors方法从Spring容器中获取所有Advisor类型的Bean和切面中所有带有通知注解的方法并将其封装为Advisor。  
