---
layout: post
title: "Spring-循环依赖的解决"
subtitle: 'Spring介绍三'
author: "Kang"
date: 2019-09-15 14:09:53
header-img: "img/post-head-img/copy-2518265_1280.jpg"
catalog: true
tags:
  - Spring
  - 实例化
---
&emsp;&emsp;前面介绍到：Spring实例化一个bean的时候，是分两步进行的，首先实例化目标bean，然后为其注入属性。[转载](https://mp.weixin.qq.com/s/6MHkZUzCjUc8O9ZTHMLjpg)


### 循环引用实例
```java
@Component
public class A {

  private B b;

  public void setB(B b) {
    this.b = b;
  }
}
@Component
public class B {
  private A a;

  public void setA(A a) {
    this.a = a;
  }
}
```
### 如何解决循环引用
&emsp;&emsp;我们知道，Spring在实例化时，在方法ApplicationContext.getBean()中存在一个递归过程，通过递归的方式连锁式实例化所引用的属性对象。   
如何解决循环依赖？  
&emsp;&emsp;答案是<font color='red'>结合对象分两步实例化的特性，在对象创建但是属性未实例化时，将对象本身的引用提前暴露出来。<font>这样通过提前暴露半成品对象，从根本上已经打断了循环依赖。  

![Spring依赖获取示意图](https://raw.githubusercontent.com/kangzhihu/images/master/spring-%E5%BE%AA%E7%8E%AF%E4%BE%9D%E8%B5%96%E8%A7%A3%E5%86%B3.jpg)
#### 代码分析
&emsp;&emsp;在初始化对象时，AbstractBeanFactory#doGetBean为总的创建入口。精简后核心有如下代码：
```java
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
    @Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {
    ...
    // 检查是否已经存在bean，也即检查前面提到的成品/半成品对象
    // 尝试通过bean名称获取目标bean对象，比如这里的A对象
    Object sharedInstance = getSingleton(beanName);
    if (sharedInstance != null && args == null) {
      bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
    }else{
      ...
      if (mbd.isSingleton()) {
          // 若不存在，则才开始创建，第二个参数传的就是一个ObjectFactory类型的对象
     	sharedInstance = getSingleton(beanName, () -> {
		try {
			   return createBean(beanName, mbd, args);
		  }
		  ...
	   });
	   bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
	   }
      ...
    }

}
```
&emsp;&emsp;我们先下getSingleton方法，其即前面介绍到的对Bean实例对象的成品/半成品检查预获取。singletonObjects中保存所有的成品bean实例，而在earlySingletonObjects则保留了所有提前暴露的半成品对象。
```java
// 成品/半成品bean的预获取(无锁)
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
  // 到单利池，也即一级缓存中获取
  Object singletonObject = this.singletonObjects.get(beanName);
  // 若无成品实例，则检查当前需要的bean是否正在创建中
  if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
    synchronized (this.singletonObjects) {
      //在半成品(二级缓存)中查找
      singletonObject = this.earlySingletonObjects.get(beanName);
      //若没查找到&允许提前暴露
      if (singletonObject == null && allowEarlyReference) {
        // 三级缓存中查找
        ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
        //已经存在对应的FactoryBean，则直接先创建一个半成品，并提前暴露出去(singletonFactory只负责创建，不负责属性实例化)
        // 一般来说对于一个bean，第一次尝试获取时，singletonFactory都为空，具体在后面doCreateBean中通过addSingletonFactory传入
        if (singletonFactory != null) {
          singletonObject = singletonFactory.getObject();
          this.earlySingletonObjects.put(beanName, singletonObject);
          //一旦进入成品集合(Bean被创建)，则将对应封装的工厂删除
          this.singletonFactories.remove(beanName);
        }
      }
    }
  }
  return singletonObject;
}
```
&emsp;&emsp;我们回到前面的提到的第二次getSingleton，在该方法中，才开始加锁进行对象的创建：
```java
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
  Assert.notNull(beanName, "Bean name must not be null");
  // 全局实例化锁
  synchronized (this.singletonObjects) {
    //仍然先获取一次，防止并发已完成创建
    Object singletonObject = this.singletonObjects.get(beanName);
			if (singletonObject == null) {
        // 并发检查，防止有其他流程正在创建(添加到singletonsCurrentlyInCreation)
        beforeSingletonCreation(beanName);
        try {
            // 调用外部实例化的工厂进行bean的创建，转调createBean
            singletonObject = singletonFactory.getObject();
            newSingleton = true;
          }
      }
  }
}
```
&emsp;&emsp;一旦对象创建完毕，达到提前暴露条件则在属性实例化未开始时，直接暴露出去：
```java
// 前半部分创建Bean本身，后半部分处理属性对象实例化
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args) throws BeanCreationException {
  // 实例化当前尝试获取的bean对象，比如A对象和B对象都是在这里实例化的
  BeanWrapper instanceWrapper = null;
  if (mbd.isSingleton()) {
    instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
  }
  if (instanceWrapper == null) {
    instanceWrapper = createBeanInstance(beanName, mbd, args);
  }

  // 判断Spring是否配置了支持提前暴露目标bean，也就是是否支持提前暴露半成品的bean
  boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences
    && isSingletonCurrentlyInCreation(beanName));
  if (earlySingletonExposure) {
    // 如果支持，这里就会直接将当前生成的半成品的bean(无属性实例化)放到singletonFactories中，这个singletonFactories
    // 就是前面第一个getSingleton()方法中所使用到的singletonFactories属性，也就是说，这里就是封装半成品的bean的地方。
    // 当其他对象再次查找当前对象时，则直接从前面的singletonFactories中获取半成品对象。
    addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
  }

  try {
    // 在初始化实例之后，这里就是判断当前bean是否依赖了其他的bean，如果依赖了，就会递归的调用getBean()方法尝试获取目标bean
    populateBean(beanName, mbd, instanceWrapper);
  } catch (Throwable ex) {
    ...
  }

  return exposedObject;
}
```
&emsp;&emsp;从上面可以看出，一个可能对象可能的创建流程为：半成品先进入singletonFactories，后进入earlySingletonObjects，最后成品进入singletonFactories
