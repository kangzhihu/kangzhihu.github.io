---
layout: post
title: "Spring-AOP与IOC总体预览"
subtitle: 'Spring介绍一'
author: "Kang"
date: 2019-09-15 13:42:43
header-img: "img/post-head-img/copy-2518265_1280.jpg"
catalog: true
tags:
  - Spring
  - 实例化
---
&emsp;&emsp;最终还是觉得应该将AOP和IOC主线简单总结下，不知道为啥，心情有点沉重，太多不懂。
### IOC
&emsp;&emsp;网上有一句话总结的十分精辟：IoC不是一种技术，只是一种思想。    
&emsp;&emsp;IoC对编程带来的最大改变不是从代码上，而是从思想上，发生了“主从换位”的变化。应用程序原本是老大，要获取什么资源都是主动出击自动动手，但是在IoC/DI思想中，应用程序就变成被动接受的(只管享受)了，被动的等待IoC容器来创建并注入它所需要的资源了。

#### IOC容器
&emsp;&emsp;IOC容器是由一系列的组件，包括BeanDefinitionMap、singletonObject(bean缓存池)、BeanPostProcessor等等，这些组件组合起来完成bean的依赖注入&控制反转功能的一个组件。
#### 父子容器
&emsp;&emsp;在创建Bean的时候会先到父容器中获取bean，没有获取到的时候，再在子类容器中获取。下面是SpringMVC的一个父子容器：  
![SpringMVC的父子工厂](https://raw.githubusercontent.com/kangzhihu/images/master/spring-mvc-context-hierarchy.png)  
&emsp;&emsp;其中Root WebApplicationContext就是基础IOC容器。  

### AOP
[推荐阅读-这个系列对AOP讲的比较全面](https://blog.csdn.net/zknxx)
###### 关于AOP的几个点：
- 1、代理两种方式实现，一种jdk动态代理，一种cglib代理。jdk代理需要和被代理类实现相同的接口，cglib则无要求。
```
a)、根据是否有接口条件创建AopProxy对象，该对象中封装着代理创建的上下文；
b)、使用AopProxy创建具体的代理对象。
```

- 2、<font color="red">被代理对象是何时被代理对象替换的</font>：  
&emsp;&emsp;在doCreateBean中initializeBean发放在初始化对象时调用了BeanPostProcessors，其中AbstractAutoProxyCreator为BeanPostProcessor的一种，其方法postProcessAfterInitialization->wrapIfNecessary中做了是否需要代理替换的判断，当存在advice时，则调用getProxy走AOP代理对象创建。

- 3、调用链  
&emsp;&emsp;AOP对象的拦截器执行链是一个MethodInterceptor的集合,在动态代理proceed中统一被使用。  

- 4、关于拦截器链使用：  
&emsp;&emsp;a)、在创建代理对象时，将advised作为一个属性封装在代理对象中；     
&emsp;&emsp;b)、以jdk代理的对象为例，不同interceptor调用自己invoke方法，在invoke方法中执行携带的advice属性对象的方法，简化后如下:  
```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    // 如果执行到链条的末尾 则直接调用连接点方法 即 直接调用目标方法
    if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
        return invokeJoinpoint();
    }
    ...
    // 从Advised中根据方法名和目标类获取 AOP拦截器执行链 重点要分析的内容
    List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
    //如果链路为空则直接调用方法
    if (chain.isEmpty()) {
	Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
	retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
    }
    //封装链路调用
    else{
      invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
      // invocation方法内部循环调用拦截器链,
      // 在每次proceed中调用拦截器的invoke方法实现前置、环绕、后置等执行
      retVal = invocation.proceed();
    }
    ...
    return retVal;
}
```

#### 注意理解
&emsp;&emsp;不要将Spring中正常对象的创建和AOP的创建搞混淆了，AOP的对象创建依赖ProxyFactoryBean，通过生成代理对象，而正常的Bean则依赖反射或者cglib创建。  
- 对于正常创建流程，在反射创建或者cglib创建后，Bean会被BeanWrapperImpl包装，使用包装模式对外提供服务，原有的bean被创建后不动。  
- 对于在Bean创建过程中，通过BeanPostProcessor的after来进行原有对象的替换。  


### Spring初始化过程
- 1、反射初始化一个bean；
- 2、按照Spring上下文对bean的properties进行注入；
- 3、若实现了Aware接口，将相关参数注入到bean中去；
- 4、若实现BeanPostProcessor接口，调用postProcessBeforeInitialization方法，BeanPostProcessor经常被用作是Bean内容的更改；
- 5、若Bean配置了init-method方法，则自动调用方法进行初始化；
- 6、若实现BeanPostProcessor接口，调用postProcessAfterInitialization方法；
- 7、当Bean不再需要时，会经过清理阶段，如果Bean实现了DisposableBean这个接口，会调用那个其实现的destroy()方法；
- 8、最后，如果这个Bean的Spring配置中配置了destroy-method属性，会自动调用其配置的销毁方法。

#### refresh方法
```java
@Override
public void refresh() throws BeansException, IllegalStateException {
    //startupShutdownMonitor对象在spring环境刷新和销毁的时候都会用到，确保刷新和销毁不会同时执行
    synchronized (this.startupShutdownMonitor) {
        // 准备工作，例如记录事件，设置标志，检查环境变量等，并有留给子类扩展的位置，用来将属性加入到applicationContext中
        prepareRefresh();

        // 创建beanFactory，这个对象作为applicationContext的成员变量，可以被applicationContext拿来用,
        // 并且解析资源（例如xml文件），取得bean的定义，放在beanFactory中(说到底核心是一个 beanName-> beanDefinition 的 map)
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // 对beanFactory做一些设置，例如类加载器、spel解析器、指定bean的某些类型的成员变量对应某些对象等,并手动注册几个特殊的bean
        // 设置固定BeanPostProcessor，为beanFactory添加内部的BeanPostProcessor的bean对象等
        prepareBeanFactory(beanFactory);

        try {
            // 子类扩展用，向BeanFactory中设置其他属性
            // 扩展注入BeanFactoryPostProcessor作为BeanFactory的后置处理器
            postProcessBeanFactory(beanFactory);

            //  获取所有的目前已有的BeanFactoryPostProcessor，并调用其postProcessBeanFactory对工厂进行处理
            // 加载了Bean定义，生成Bean定义的集合
            // 注意：有别于bean后置处理器处理bean实例，beanFactory后置处理器是对bean工厂的扩展处理
            invokeBeanFactoryPostProcessors(beanFactory);

            // 将所有的bean的后置处理器排好序，但不会马上用，bean实例化之后会用到
            // 注意： 前面提到的区别，一个是对工厂处理，另一个是对Bean处理
            registerBeanPostProcessors(beanFactory);

            // 初始化国际化服务
            initMessageSource();

            // 创建事件广播器
            initApplicationEventMulticaster();

            // 空方法，留给子类自己实现的，在实例化bean之前做一些ApplicationContext相关的操作
            onRefresh();

            // 注册一部分特殊的事件监听器，剩下的只是准备好名字，留待bean实例化完成后再注册
            registerListeners();

            // 单例模式的bean的实例化、成员变量注入、初始化等工作都在此完成
            //也即创建bean
            finishBeanFactoryInitialization(beanFactory);

            // applicationContext刷新完成后的处理，例如生命周期监听器的回调，广播通知等
            finishRefresh();
        }

        catch (BeansException ex) {
            logger.warn("Exception encountered during context initialization - cancelling refresh attempt", ex);

            // 刷新失败后的处理，主要是将一些保存环境信息的集合做清理
            destroyBeans();

            // applicationContext是否已经激活的标志，设置为false
            cancelRefresh(ex);

            // Propagate exception to caller.
            throw ex;
        }
    }
}
```
