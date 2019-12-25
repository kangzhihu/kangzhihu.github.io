---
layout: post
title: "中间件-dubbo之Filter过滤器"
subtitle: '中间件之dubbo整理总结四'
author: "Kang"
date: 2019-09-15 18:21:58
header-img: "img/post-head-img/old-houses-4433982_1280.jpg"
catalog: true
tags:
  - 中间件
  - RPC
  - dubbo
---

&emsp;&emsp;Filter在dubbo中的功能类似Spring的Aop，其对对象进行包装增强。

### 外部SPI对象的全加载
&emsp;&emsp;前文我们提到，ReferenceConfig为dubbo创建的入口，整个类如下：
```java
public class ReferenceConfig<T> extends AbstractReferenceConfig {
    //获取到的refprotocol为根据字符串拼接产生的Protocol$Adaptive类对象
    private static final Protocol refprotocol = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();
    
 }
```
&emsp;&emsp;在类被加载的时候，将触发static属性SPI的调用，其中，ExtensionLoader主要代码为：
```java
public class ExtensionLoader<T> {
    private static final String SERVICES_DIRECTORY = "META-INF/services/";
    private static final String DUBBO_DIRECTORY = "META-INF/dubbo/";
    private static final String DUBBO_INTERNAL_DIRECTORY = DUBBO_DIRECTORY + "internal/";

    @SuppressWarnings("unchecked")
    public T getAdaptiveExtension() {
        Object instance = cachedAdaptiveInstance.get();
        if (instance == null) {
            if (createAdaptiveInstanceError == null) {
                synchronized (cachedAdaptiveInstance) {
                    instance = cachedAdaptiveInstance.get();
                    if (instance == null) {
                        try {
                            instance = createAdaptiveExtension();    //获取扩展类对象
                            cachedAdaptiveInstance.set(instance);
                        } catch (Throwable t) {
                            createAdaptiveInstanceError = t;
                            throw new IllegalStateException("fail to create adaptive instance: " + t.toString(), t);
                        }
                    }
                }
            } else {
                throw new IllegalStateException("fail to create adaptive instance: " + createAdaptiveInstanceError.toString(), createAdaptiveInstanceError);
            }
        }

        return (T) instance;
    }
    
}

//加载扩展类形成Map列表，将路径SERVICES_DIRECTORY、DUBBO_DIRECTORY、DUBBO_INTERNAL_DIRECTORY下的数据全部解析放在cachedClasses中。
//其中cachedClasses的key为SPI文件中“=”之前部分
private Map<String, Class<?>> getExtensionClasses() {
    Map<String, Class<?>> classes = cachedClasses.get();
    if (classes == null) {
        synchronized (cachedClasses) {
            classes = cachedClasses.get();
            if (classes == null) {
                classes = loadExtensionClasses();    //根据路径，解析并加载配置文本中的所有类,并放在Holder对象cachedClasses中
                cachedClasses.set(classes);
            }
        }
    }
    return classes;
}

//对象的包装，任何对象都包含协议类型,根据传入的协议名，从Protocol.class的SPI扩展类中获取对应的对象
private T createExtension(String name) {
    Class<?> clazz = getExtensionClasses().get(name); // name为协议名，如：dubbo、registry，获取基础扩展类
    if (clazz == null) {
        throw findException(name);
    }
    try {
        T instance = (T) EXTENSION_INSTANCES.get(clazz);   //获取对应的外部扩展类，比如DubboProtocol
        if (instance == null) {
            EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.newInstance());
            instance = (T) EXTENSION_INSTANCES.get(clazz);
        }
        injectExtension(instance);
        Set<Class<?>> wrapperClasses = cachedWrapperClasses;
        if (wrapperClasses != null && !wrapperClasses.isEmpty()) {
            for (Class<?> wrapperClass : wrapperClasses) {
                instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));  //将上面获取的类进行包装，比如一次Wrapper：ProtocolFilterWrapper
            }
        }
        return instance;
    } catch (Throwable t) {
        throw new IllegalStateException("Extension instance(name: " + name + ", class: " +
                type + ")  could not be instantiated: " + t.getMessage(), t);
    }
}
```
&emsp;&emsp;核心代码为ExtensionLoader#getAdaptiveExtension，在这个过程中，当前业务类(如Protocol.class)的ExtensionLoader对象将扫描指定路径下的配置文件，并最终将调用到getExtensionClasses触发配置文件中对象的加载，放入Holder对象cachedClasses中。

### 对象的装饰包装
&emsp;&emsp;前面说过，ReferenceBean实现了FactoryBean，故最后加载在ReferenceConfig#createProxy中进行的:

```java
private T createProxy(Map<String, String> map) {
    ...
    if (isJvmRefer) {
        URL url = new URL(Constants.LOCAL_PROTOCOL, NetUtils.LOCALHOST, 0, interfaceClass.getName()).addParameters(map);
        invoker = refprotocol.refer(interfaceClass, url);
        if (logger.isInfoEnabled()) {
            logger.info("Using injvm service " + interfaceClass.getName());
        }
    } else {
        ...
        //urls为外部配置+组装后的服务地址，根据url协议类型获取对应的Protocol实现子类
        //refprotocol为通过前面介绍的getAdaptiveExtension获取，为一个Protocol$Adaptive对象
        if (urls.size() == 1) {   
            invoker = refprotocol.refer(interfaceClass, urls.get(0)); 
        } 
        ...
    }
    return (T) proxyFactory.getProxy(invoker);  // service服务对象代理，根据返回的invoker进行对象的反射代理。
}

```
&emsp;&emsp;对Dubbo中使用的协议Invoker对象进行代理，包装增强部分进行分析，refprotocol对象仅为转调过程，其代码为：
```java
public class Protocol$Adaptive implements com.alibaba.dubbo.rpc.Protocol {

    public com.alibaba.dubbo.rpc.Exporter export(com.alibaba.dubbo.rpc.Invoker arg0) throws com.alibaba.dubbo.rpc.RpcException {
        if (arg0 == null) { throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument == null"); }
        if (arg0.getUrl() == null) { throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument getUrl() == null"); }
        com.alibaba.dubbo.common.URL url = arg0.getUrl();
        String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
        if (extName == null) {
            throw new IllegalStateException(
                    "Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");
        }
        // ExtensionLoader.getExtensionLoader 返回ExtensionLoader的实例
        com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol) ExtensionLoader.getExtensionLoader(
                com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);
        return extension.export(arg0);
    }

    public com.alibaba.dubbo.rpc.Invoker refer(java.lang.Class arg0, com.alibaba.dubbo.common.URL arg1)
            throws com.alibaba.dubbo.rpc.RpcException {
        if (arg1 == null) { throw new IllegalArgumentException("url == null"); }
        com.alibaba.dubbo.common.URL url = arg1;
        String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
        if (extName == null) {
            throw new IllegalStateException(
                    "Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");
        }
        com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol) ExtensionLoader.getExtensionLoader(
                com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName); // getExtension对原始invoker进行了包装
        return extension.refer(arg0, arg1);
    }
}

```
&emsp;&emsp;可以看到refer中存在对ExtensionLoader#getExtension方法的调用，该方法在上面已经介绍了，调用了createExtension方法，也即对原始的返回进行了Wrapper，给Spring的对象最终为包装后的对象。


### Filter调用链
&emsp;&emsp;与远程的交互是通过Protocol的export和refer方法进行的，即ProtocolFilterWrapper实现了具体的服务。在调用远程服务时，refer方法中buildInvokerChain构建了整个Filter拦截链。

```java
private static <T> Invoker<T> buildInvokerChain(final Invoker<T> invoker, String key, String group) {
    Invoker<T> last = invoker;
    List<Filter> filters = ExtensionLoader.getExtensionLoader(Filter.class).getActivateExtension(invoker.getUrl(), key, group);
    if (!filters.isEmpty()) {
        for (int i = filters.size() - 1; i >= 0; i--) {
            final Filter filter = filters.get(i);
            final Invoker<T> next = last;
            last = new Invoker<T>() {

                @Override
                public Class<T> getInterface() {
                    return invoker.getInterface();
                }

                @Override
                public URL getUrl() {
                    return invoker.getUrl();
                }

                @Override
                public boolean isAvailable() {
                    return invoker.isAvailable();
                }

                @Override
                public Result invoke(Invocation invocation) throws RpcException {
                    return filter.invoke(next, invocation); // Filter调用链使用
                }

                @Override
                public void destroy() {
                    invoker.destroy();
                }

                @Override
                public String toString() {
                    return invoker.toString();
                }
            };
        }
    }
    return last;
}

```
### APM追踪
&emsp;&emsp;明白上面的Filter之后就很明了，对于主链路的追踪附加处理，外部系统通过继承Filter，并通过SPI注入即可。
