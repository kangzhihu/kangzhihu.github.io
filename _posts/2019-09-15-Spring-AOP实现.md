---
layout: post
title: "Spring-AOP实现"
subtitle: 'Spring介绍四'
author: "Kang"
date: 2019-09-15 14:23:49
header-img: "img/post-head-img/copy-2518265_1280.jpg"
catalog: true
tags:
  - Spring
  - AOP
---
&emsp;&emsp;在《Spring-自身实例化与属性实例化》中我们提到<font color="red">“AOP功能就是用过InstantiationAwareBeanPostProcessor来实现的，其返回了对象的代理。”</font>，现在来通过代码具体说明。


### 预获取
&emsp;&emsp;直接看关注的核心代码,在createBean是先通过resolveBeforeInstantiation进行了代理类的调用。
```java
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) throws BeanCreationException {
  //省略 ...
  try {
    // Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
    // 先调用InstantiationAwareBeanPostProcessor，可以直接返回一个代理对象，而不用去实例化
    // 此处其实是AOP功能的核心要素
    Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
    // 若存在代理，则直接返回，中断结束整个初始化流程
    if (bean != null) {
      return bean;
    }
  }
  //省略 ...

  try {
    Object beanInstance = doCreateBean(beanName, mbdToUse, args);
    if (logger.isDebugEnabled()) {
      logger.debug("Finished creating instance of bean '" + beanName + "'");
    }
    return beanInstance;
  }
  //省略 ...
}

protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
  Object bean = null;
  if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
    // Make sure bean class is actually resolved at this point.
    if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
      Class<?> targetType = determineTargetType(beanName, mbd);
      if (targetType != null) {
        // 若存在则接着处理
        bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
        if (bean != null) {
          bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
        }
      }
    }
    mbd.beforeInstantiationResolved = (bean != null);
  }
  return bean;
}
```
&emsp;&emsp;需要注意的是，<font color='green'>applyBeanPostProcessorsBeforeInstantiation中存在targetSource，则直接在对象初始化之前进行创建代理, 避免了目标对象不必要的实例化</font>。此处若不存在targetSource，则在完成实例化后，于initializeBean方法中接着调用applyBeanPostProcessorsBeforeInitialization和applyBeanPostProcessorsAfterInitialization进行处理。    

### BeanPostProcessor引入  
#### 实例化前的BeanPostProcessor
&emsp;&emsp;我们简单看下实例化前，对于BeanPostProcessor的使用。resolveBeforeInstantiation尝试获取一个targetSource，其子方法applyBeanPostProcessorsBeforeInstantiation循环处理了所有的InstantiationAwareBeanPostProcessor，我们主要看关于AOP的子类AnnotationAwareAspectJAutoProxyCreator,在其父类中AbstractAutoProxyCreator中：   
1. 先创建代理对象：postProcessBeforeInstantiation
2. 对创建的代理对象使用BeanPostProcessor：postProcessAfterInitialization   

```java
public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
  Object cacheKey = getCacheKey(beanClass, beanName);

  if (!StringUtils.hasLength(beanName) || !this.targetSourcedBeans.contains(beanName)) {
    if (this.advisedBeans.containsKey(cacheKey)) {
      return null;
    }
    // 是否是Advice、Pointcut、Advisor、AopInfrastructureBean
    // 判断是否是切面（判断是否有aspect注解，@Aspect注解）等特殊的bean处理
    if (isInfrastructureClass(beanClass) || shouldSkip(beanClass, beanName)) {
      this.advisedBeans.put(cacheKey, Boolean.FALSE);
      return null;
    }
  }

  // 获取targetSource, 如果存在则直接在对象初始化之前进行创建代理, 避免了目标对象不必要的实例化
  // 如果没有自定义TargetSource，则走到postProcessAfterInitialization方法创建代理
  TargetSource targetSource = getCustomTargetSource(beanClass, beanName);
  if (targetSource != null) {
    if (StringUtils.hasLength(beanName)) {
      this.targetSourcedBeans.add(beanName);
    }
    // 查找当前类所有的有效的Advisors
    Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(beanClass, beanName, targetSource);
    // 根据JDK或者cglib生成动态代理对象
    Object proxy = createProxy(beanClass, beanName, specificInterceptors, targetSource);
    this.proxyTypes.put(cacheKey, proxy.getClass());
    return proxy;
  }

  return null;
}

```
#### 实例化后的BeanPostProcessor(后置处理器)
&emsp;&emsp;<font color='blue'>上面说到，只有在targetSource能获取到的时候才直接处理，否则将在对象初始化完成后initializeBean方法下的applyBeanPostProcessorsBeforeInitialization和applyBeanPostProcessorsAfterInitialization，我们简单看下实例化完成之后的调用beanProcessor#postProcessAfterInitialization来进行Bean的属性/行为的修改，也在这个地方实现了AOP功能。</font>    
```java
// AbstractAutoProxyCreator.java

//bean实例化完成后处理
public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
    throws BeansException {

  Object result = existingBean;
  for (BeanPostProcessor beanProcessor : getBeanPostProcessors()) {
    Object current = beanProcessor.postProcessAfterInitialization(result, beanName);
    if (current == null) {
      return result;
    }
    result = current;
  }
  return result;
}

public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) throws BeansException {
  if (bean != null) {
    Object cacheKey = getCacheKey(bean.getClass(), beanName);
    if (!this.earlyProxyReferences.contains(cacheKey)) {
      return wrapIfNecessary(bean, beanName, cacheKey);
    }
  }
  return bean;
}
```
### advice增强器代理包装
&emsp;&emsp;对于AOP来说，我们关注AbstractAutoProxyCreator的postProcessAfterInitialization，其调用了wrapIfNecessary方法来对前面生成的代理对象进一步进行有条件包装：
```java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
  // 省略 ...
  // 获取所有的advice增强器
  Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
  if (specificInterceptors != DO_NOT_PROXY) {
    this.advisedBeans.put(cacheKey, Boolean.TRUE);
    // 对传入的代理对象进一步进行包装
    Object proxy = createProxy(
        bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
    this.proxyTypes.put(cacheKey, proxy.getClass());
    return proxy;
  }

  this.advisedBeans.put(cacheKey, Boolean.FALSE);
  return bean;
}
```
&emsp;&emsp;若获取到了advice增强器，则将增强器作为一个链，设置在代理对象中，在invoke时进行调用。  
&emsp;&emsp;由于上面返回了对象的一个代理，所以在调用的时候通过代理对象进行对外服务，以jdk实现的代理对象为例，其invoke方法中引入了拦截链：
```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
   // 省略 ...

    Object retVal;
    // 
    target = targetSource.getTarget();
    Class<?> targetClass = (target != null ? target.getClass() : null);

    // 获取拦截链
    List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

    // Check whether we have any advice. If we don't, we can fallback on direct
    // reflective invocation of the target, and avoid creating a MethodInvocation.
    if (chain.isEmpty()) {
      // We can skip creating a MethodInvocation: just invoke the target directly
      // Note that the final invoker must be an InvokerInterceptor so we know it does
      // nothing but a reflective operation on the target, and no hot swapping or fancy proxying.
      Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
      retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
    }
    else {
      // 若存在拦截链，则在ReflectiveMethodInvocation#invocation中递归调用拦截链
      invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
      // Proceed to the joinpoint through the interceptor chain.
      retVal = invocation.proceed();
    }
    // 省略 ...
}
```
### 拦截器链调用与AOP  
&emsp;&emsp;循环调用拦截器链，interceptor包含了根据配置解析拦截器：
+ MethodBeforeAdviceInterceptor 前置通知
+ AspectJAfterAdvice 后置通知
+ AfterReturningAdviceInterceptor 返回通知
+ AspectJAfterThrowingAdvice 异常通知
```java
public Object proceed() throws Throwable {
  if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
     //执行目标方法
    return invokeJoinpoint();
  }

  Object interceptorOrInterceptionAdvice =
      this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
  if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
    // Evaluate dynamic method matcher here: static part will already have
    // been evaluated and found to match.
    InterceptorAndDynamicMethodMatcher dm =
        (InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
    if (dm.methodMatcher.matches(this.method, this.targetClass, this.arguments)) {
      return dm.interceptor.invoke(this);
    }
    else {
      // Dynamic matching failed.
      // Skip this interceptor and invoke the next in the chain.
      return proceed();
    }
  }
  else {
    // It's an interceptor, so we just invoke it: The pointcut will have
    // been evaluated statically before this object was constructed.
    return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
  }
}
```
