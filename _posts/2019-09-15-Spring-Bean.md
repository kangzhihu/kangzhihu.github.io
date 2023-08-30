---
layout: post
title: "Spring-自身创建与属性实例化"
subtitle: 'Spring介绍二'
author: "Kang"
date: 2019-09-15 13:56:18
header-img: "img/post-head-img/copy-2518265_1280.jpg"
catalog: true
tags:
  - Spring
  - 实例化
---
&emsp;&emsp;一次完整的实例化动作其实分为三大部分：自身实例化、属性实例化、相关回调


### 自身实例化
&emsp;&emsp;直接看关注的核心代码。
```java
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) throws BeanCreationException {
  //省略 ...
  try {
    // Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
    // 先调用InstantiationAwareBeanPostProcessor，可以直接返回一个代理对象，而不用去实例化
    // 此处暂且不提，其实是宪法AOP功能的核心要素
    Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
    // 若存在代理，则直接返回
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

protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
    throws BeanCreationException {
  BeanWrapper instanceWrapper = null;
  if (mbd.isSingleton()) {
    instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
  }
  if (instanceWrapper == null) {
    // 创建bean并进行包装
    instanceWrapper = createBeanInstance(beanName, mbd, args);
  }
  final Object bean = instanceWrapper.getWrappedInstance();
  Class<?> beanType = instanceWrapper.getWrappedClass();
  // 省略...
  // 下面这块代码是为了解决循环依赖的问题
  boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
      isSingletonCurrentlyInCreation(beanName));
  if (earlySingletonExposure) {
    if (logger.isDebugEnabled()) {
      logger.debug("Eagerly caching bean '" + beanName +
          "' to allow for resolving potential circular references");
    }
    addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
  }

  Object exposedObject = bean;
  try {
    // 属性实例化
    populateBean(beanName, mbd, instanceWrapper);
    // 相关回调处理
    exposedObject = initializeBean(beanName, exposedObject, mbd);
  }
    ...
  return exposedObject;
}
```
&emsp;&emsp;上面createBean方法的 resolveBeforeInstantiation 发起了对 InstantiationAwareBeanPostProcessor 的调用，暂且不细讲，只需要知道，<font color="red">AOP功能就是用过InstantiationAwareBeanPostProcessor来实现的，其返回了对象的代理。</font>   
&emsp;&emsp;接着直接看一个简单的无参构造创建：
```java
protected BeanWrapper instantiateBean(final String beanName, final RootBeanDefinition mbd) {
  try {
    Object beanInstance;
    final BeanFactory parent = this;
    if (System.getSecurityManager() != null) {
      beanInstance = AccessController.doPrivileged((PrivilegedAction<Object>) () ->
          getInstantiationStrategy().instantiate(mbd, beanName, parent),
          getAccessControlContext());
    }
    else {
      // 实例化对象，核心操作
      beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, parent);
    }
    // 对实例化的对象包装一下
    BeanWrapper bw = new BeanWrapperImpl(beanInstance);
    initBeanWrapper(bw);
    return bw;
  }
  catch (Throwable ex) {
    throw new BeanCreationException(
        mbd.getResourceDescription(), beanName, "Instantiation of bean failed", ex);
  }
}

```
&emsp;&emsp;我们来看核心的初始化，其核心为instantiate方法
```java
public Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner) {
  // 如果不存在方法覆写(无子类)，那就使用 java 反射进行实例化，否则使用 CGLIB,
  // 方法覆写 请参见"方法注入"中lookup-method 和 replaced-method 两种方式
  // JDK动态代理只能对实现了接口的类生成代理，而不能针对类。
  if (!bd.hasMethodOverrides()) {
    Constructor<?> constructorToUse;
    synchronized (bd.constructorArgumentLock) {
      constructorToUse = (Constructor<?>) bd.resolvedConstructorOrFactoryMethod;
      if (constructorToUse == null) {
        final Class<?> clazz = bd.getBeanClass();
        if (clazz.isInterface()) {
          throw new BeanInstantiationException(clazz, "Specified class is an interface");
        }
        try {
          if (System.getSecurityManager() != null) {
            constructorToUse = AccessController.doPrivileged(
                (PrivilegedExceptionAction<Constructor<?>>) clazz::getDeclaredConstructor);
          }
          else {
            constructorToUse =	clazz.getDeclaredConstructor();
          }
          bd.resolvedConstructorOrFactoryMethod = constructorToUse;
        }
        catch (Throwable ex) {
          throw new BeanInstantiationException(clazz, "No default constructor found", ex);
        }
      }
    }
    // 直接使用jdk反射处理
    return BeanUtils.instantiateClass(constructorToUse);
  }
  else {
    // 使用CGLIB子类进行创建，使用场景有限？
    return instantiateWithMethodInjection(bd, beanName, owner);
  }
}
```
&emsp;&emsp;这次只是为了创建基础对象，而不是为了代理(代理是为了增强功能)，所以也使用了反射去生成对象。      
&emsp;&emsp;两种创建方式引出一个很经典的问题：如果不做特殊配置 spring的@Transactional注解，放在类中非接口内的方法上时，是不起作用的。因为Spring默认使用JDK的代理（只代理了接口中的方法），被Spring代理的类只能拦截接口中的方法，不能拦截非接口中的方法。

### 属性实例化
&emsp;&emsp;看完了 createBeanInstance(...) 方法，我们来看看 populateBean(...) 方法，该方法负责进行属性设值，处理依赖。
```java
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
  // 省略 ...
  // 获取Bean定义中的所有对象属性
  PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);

  if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME ||
      mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
    MutablePropertyValues newPvs = new MutablePropertyValues(pvs);

    // 根据BeanName进行注入
    // 递归调用getBean进行创建
    if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME) {
      autowireByName(beanName, mbd, bw, newPvs);
    }

    // 根据BeanType进行注入
    if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
      autowireByType(beanName, mbd, bw, newPvs);
    }

    pvs = newPvs;
  }

  boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
  boolean needsDepCheck = (mbd.getDependencyCheck() != RootBeanDefinition.DEPENDENCY_CHECK_NONE);

  if (hasInstAwareBpps || needsDepCheck) {
    if (pvs == null) {
      pvs = mbd.getPropertyValues();
    }
    PropertyDescriptor[] filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
    if (hasInstAwareBpps) {
      for (BeanPostProcessor bp : getBeanPostProcessors()) {
        if (bp instanceof InstantiationAwareBeanPostProcessor) {
          InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
          pvs = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
          if (pvs == null) {
            return;
          }
        }
      }
    }
    if (needsDepCheck) {
      checkDependencies(beanName, mbd, filteredPds, pvs);
    }
  }

  if (pvs != null) {
    applyPropertyValues(beanName, mbd, bw, pvs);
  }
}
```

### 相关回调
&emsp;&emsp;在属性对象处理完毕后，initializeBean方法进行相关回调处理，主要是对bean进行预处理或者功能增强，也即实现扩展的地方。
```java
protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
  if (System.getSecurityManager() != null) {
    AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
      invokeAwareMethods(beanName, bean);
      return null;
    }, getAccessControlContext());
  }
  else {
    // 如果 bean 实现了 BeanNameAware、BeanClassLoaderAware 或 BeanFactoryAware 接口，回调
    invokeAwareMethods(beanName, bean);
  }

  Object wrappedBean = bean;
  if (mbd == null || !mbd.isSynthetic()) {
    // BeanPostProcessor 的 postProcessBeforeInitialization 回调
    wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
  }

  try {
    // 处理 bean 中定义的 init-method 或如果 bean 实现了 InitializingBean 接口，调用 afterPropertiesSet() 方法
    invokeInitMethods(beanName, wrappedBean, mbd);
  }
  catch (Throwable ex) {
    throw new BeanCreationException(
        (mbd != null ? mbd.getResourceDescription() : null),
        beanName, "Invocation of init method failed", ex);
  }
  if (mbd == null || !mbd.isSynthetic()) {
    // BeanPostProcessor 的 postProcessAfterInitialization 回调
    wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
  }

  return wrappedBean;
}
```
