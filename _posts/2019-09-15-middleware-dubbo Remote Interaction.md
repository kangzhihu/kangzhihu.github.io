---
layout: post
title: "中间件-dubbo之远程交互"
subtitle: '中间件之dubbo整理总结二'
author: "Kang"
date: 2019-09-15 18:05:50
header-img: "img/post-head-img/old-houses-4433982_1280.jpg"
catalog: true
tags:
  - 中间件
  - RPC
  - dubbo
---
&emsp;&emsp;承接RPC模型，对consumer而言，Dubbo协议对每个Service默认是基于Netty单一长连接和NIO异步通讯的，适合小数据大并发的服务调用。在consumer端，会对需要调用的每个服务都创建一个服务代理bean，即Reference，该服务代理bean在consumer就是一个跟使用Spring的@Service注解修饰的普通Service类bean一样注册到spring容器，也是单例的。   
Dubbo是如何实现单一长连接并发请求时包干扰(“串线”)的呢；

### Dubbo调用流程(Dubbo.version = 2.6.4)
#### 1. 远程接口代理
&emsp;&emsp;对于每一个远程调用的接口，根据配置会产生一个ReferenceBean实例，调用方将以调用本地服务的方式无感知的调用ReferenceBean实例；
#### 1.1 URL总线
&emsp;&emsp;一个URL示例：
```xml
dubbo://127.0.0.1:9090/com.example.service1?param1=value1&param2=value2
```    
&emsp;&emsp;其包含了协议、主机、端口、交互参数等几部分  

##### 2. ReferenceBean的解析生成(全透明融入spring)
&emsp;&emsp;我们知道spring的入口方法就是refresh()，从该方法跟踪进去：obtainFreshBeanFactory()->refreshBeanFactory()->loadBeanDefinitions(beanFactory)->loadBeanDefinitions(beanDefinitionReader)->..->XmlBeanDefinitionReader#loadBeanDefinitions->registerBeanDefinitions：
```java
public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
  BeanDefinitionDocumentReader documentReader = this.createBeanDefinitionDocumentReader();
  int countBefore = this.getRegistry().getBeanDefinitionCount();
  documentReader.registerBeanDefinitions(doc, this.createReaderContext(resource)); //上下文中封装外部处理器
  return this.getRegistry().getBeanDefinitionCount() - countBefore;
}
```
createReaderContext通过外部classPath方法生成引入了处理器：
```java
protected NamespaceHandlerResolver createDefaultNamespaceHandlerResolver() {
        ClassLoader cl = this.getResourceLoader() != null ? this.getResourceLoader().getClassLoader() : this.getBeanClassLoader();
        return new DefaultNamespaceHandlerResolver(cl);
    }
```
```java
public DefaultNamespaceHandlerResolver(@Nullable ClassLoader classLoader) {
     this(classLoader, "META-INF/spring.handlers");  //classpath目录，也即外部如dubbo等框架的解析器扩展
 }
```
&emsp;&emsp;接着看代码到DefaultBeanDefinitionDocumentReader#parseBeanDefinitions:
```java
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
  if (delegate.isDefaultNamespace(root)) {
      NodeList nl = root.getChildNodes();

      for(int i = 0; i < nl.getLength(); ++i) {
          Node node = nl.item(i);
          if (node instanceof Element) {
              Element ele = (Element)node;
              if (delegate.isDefaultNamespace(ele)) {
                  this.parseDefaultElement(ele, delegate);
              } else {
                  delegate.parseCustomElement(ele);
              }
          }
      }
  } else {
      delegate.parseCustomElement(root);
  }

}
```
&emsp;&emsp;可以看到，当不为默认Namespace命名空间(spring)时，走的上面上下文中存储的外部定制处理：
```java
public BeanDefinition parseCustomElement(Element ele, @Nullable BeanDefinition containingBd) {
  String namespaceUri = this.getNamespaceURI(ele);
  if (namespaceUri == null) {
      return null;
  } else {
      NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
      if (handler == null) {
          this.error("Unable to locate Spring NamespaceHandler for XML schema namespace [" + namespaceUri + "]", ele);
          return null;
      } else {
          return handler.parse(ele, new ParserContext(this.readerContext, this, containingBd));
      }
  }
}

```
根据元素的namespaceUri（也就是xml文件头那一串schema）找到对应的NamespaceHandler.【这里就是扩展自定义配置的入口】，然后调用解析器进行parse解析出BeanDefinition。

#### 3. 代用总出口
&emsp;&emsp;ReferenceAnnotationBeanPostProcessor实现了@Reference解析管理，其核心方法就是调用到ReferenceBean.get()，在通过FactoryBean接管了Dubbo代理对象的创建。  
&emsp;&emsp;ReferenceBean实例为一个服务代理refer，该refer包含一个DubboInvoker调用器invoker实例；
1. ReferenceBean继承了ReferenceConfig类并实现了FactoryBean接口，ReferenceConfig实现了FactoryBean.getObject()转调用的get()方法，其中get-->init中存在：    

```java
 private void init() {
  ...
  StaticContext.getSystemContext().putAll(attributes);
  this.ref = this.createProxy(map);
  ...
 }
  private T createProxy(Map<String, String> map) {
    ...
    if(this.urls.size() == 1) {
       this.invoker = refprotocol.refer(this.interfaceClass, (URL)this.urls.get(0));  //默认为dubbo协议，即invoker默认为DubboInvoker
    } 
     ...
     return proxyFactory.getProxy(this.invoker);   //对Dubbo中使用的协议Invoker对象进行代理
  }
   public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
    return Proxy.getProxy(interfaces).newInstance(new InvokerInvocationHandler(invoker));
  }
```
其中，refprotocol为SPI注入，根据Protocol类的@SPI("dubbo")，<font color="red">取默认为DubboProtocol实例。DubboProtocol#refer返回DubboInvoker实例对象</font>。需要注意的是：<font color='red'>代理对象接口为业务接口，其处理类Handler中包装了传入的invoker对象</font>，关键代码如下：  

```java
public class InvokerInvocationHandler implements InvocationHandler {

    private final Invoker<?> invoker;

    public InvokerInvocationHandler(Invoker<?> handler) {
        this.invoker = handler;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        String methodName = method.getName();
        Class<?>[] parameterTypes = method.getParameterTypes();
        if (method.getDeclaringClass() == Object.class) {
            return method.invoke(invoker, args);  
        }
        ...
        return invoker.invoke(new RpcInvocation(method, args)).recreate();// 参数转调到invoker对象
    }

}
```

#### 4. 底层通讯
&emsp;&emsp;DubboInvoker实例包含一个或多个ExchangerClient实例，默认为1个，通过DubboProtocol#refer#getClients中的connections参数指定，其默认实现类为HeaderExchangeClient，初始化方式为DubboInvoker生成实例时，ExchangeClient[]取值：
```java
private ExchangeClient[] getClients(URL url) {
  // whether to share connection
  boolean service_share_connect = false;
  int connections = url.getParameter(Constants.CONNECTIONS_KEY, 0);
  // if not configured, connection is shared, otherwise, one connection for one service
  if (connections == 0) { //默认
      service_share_connect = true;
      connections = 1;
  }

  ExchangeClient[] clients = new ExchangeClient[connections];
  for (int i = 0; i < clients.length; i++) {
      if (service_share_connect) {
          clients[i] = getSharedClient(url); //默认
      } else {
          clients[i] = initClient(url);
      }
  }
  return clients;
}
```
一路追踪:getSharedClient->initClient->Exchangers.connect->HeaderExchanger#connect,在connect中给出了默认HeaderExchangeClient类型的client对象，其最终在getSharedClient方法中被ReferenceCountExchangeClient进行了装饰包装。
```java
public ExchangeClient connect(URL url, ExchangeHandler handler) throws RemotingException {
     return new HeaderExchangeClient(Transporters.connect(url, new DecodeHandler(new HeaderExchangeHandler(handler))), true);
 }

```
对Transporters.connect进行追踪，可以看到下面NettyTransporter#connect的方法：
```java
 @Override
 public Client connect(URL url, ChannelHandler listener) throws RemotingException {
     return new NettyClient(url, listener);
 }
```
故可以知道，<font color="red">默认情况下使用NettyClient进行通讯</font>。    
&emsp;&emsp;HeaderExchangeClient使用Netty的Bootstrap作为远程通讯client。并使用HeaderExchangeChannel作为channel作为通讯通道。
```java
public HeaderExchangeClient(Client client) {
        if(client == null) {
            throw new IllegalArgumentException("client == null");
        } else {
            this.client = client;
            this.channel = new HeaderExchangeChannel(client); // 使用HeaderExchangeChannel作为底层通讯通道
            String dubbo = client.getUrl().getParameter("dubbo");
            this.heartbeat = client.getUrl().getParameter("heartbeat", dubbo != null && dubbo.startsWith("1.0.")?'\uea60':0);
            this.heartbeatTimeout = client.getUrl().getParameter("heartbeat.timeout", this.heartbeat * 3);
            if(this.heartbeatTimeout < this.heartbeat * 2) {
                throw new IllegalStateException("heartbeatTimeout < heartbeatInterval * 2");
            } else {
                this.startHeatbeatTimer();
            }
        }
    }
```

#### 5. 服务调用
前面介绍到，在Spring启动的时候，在Spring容器中注入的是proxyFactory动态创建的一个远程服务的代理对象。在调用动态代理对象时，将调用代理InvokerInvocationHandler对象invoke方法，也即如前面描述，转调到AbstractInvoker#invoke，而调用其子类DubboInvoker(默认)的doInvoke方法：
```java
public Result invoke(Invocation inv) throws RpcException {
...
try {
    return this.doInvoke(invocation);
}
...
}
```
#### 6. 远程返回
DubboInvoker的doInvoke如下，其中参数Invocation封装了接口、方法、参数、附加参数(Attachments)等信息。其中，<font color="red">DefaultFuture构造函数中存在Static属性FUTURES存放了请求任务。</font>  
```
public DefaultFuture(Channel channel, Request request, int timeout) {
    this.channel = channel;
    this.request = request;
    this.id = request.getId();
    this.timeout = timeout > 0 ? timeout : channel.getUrl().getPositiveParameter(Constants.TIMEOUT_KEY, Constants.DEFAULT_TIMEOUT);
    // put into waiting map.
    FUTURES.put(id, this);
    CHANNELS.put(id, channel);
}
```
```java
protected Result doInvoke(final Invocation invocation) throws Throwable {
...
ExchangeClient currentClient;
if (clients.length == 1) {
   currentClient = clients[0];
} else {
   currentClient = clients[index.getAndIncrement() % clients.length];
}
...
try {
   boolean isAsync = RpcUtils.isAsync(getUrl(), invocation);
   boolean isOneway = RpcUtils.isOneway(getUrl(), invocation);
   int timeout = getUrl().getMethodParameter(methodName, Constants.TIMEOUT_KEY, Constants.DEFAULT_TIMEOUT);
   if (isOneway) { //无结果
       boolean isSent = getUrl().getMethodParameter(methodName, Constants.SENT_KEY, false);
       currentClient.send(inv, isSent);
       RpcContext.getContext().setFuture(null);
       return new RpcResult();
   } else if (isAsync) { //异步方式
       ResponseFuture future = currentClient.request(inv, timeout);// 前面分析指知currentClient为ReferenceCountExchangeClient,检查超时时间线程为DefaultFuture
       RpcContext.getContext().setFuture(new FutureAdapter<Object>(future)); // 放在ThreadLocal中，FutureAdapter 是一个适配器，用于将 Dubbo 中的 ResponseFuture 与 JDK 中的 Future 进行适配。
       return new RpcResult(); //异步返回结果
   } else { //同步调用
       RpcContext.getContext().setFuture(null);
       return (Result) currentClient.request(inv, timeout).get();
   }
}
...
}
```
前面讲到ReferenceCountExchangeClient对HeaderExchangeClient进行了包装，故最终request底层通讯在HeaderExchangeChannel。
```java
public ResponseFuture request(Object request, int timeout) throws RemotingException {
  if (closed) {
      throw new RemotingException(this.getLocalAddress(), null, "Failed to send request " + request + ", cause: The channel " + this + " is closed!");
  }
  // 其中，Request中mId为当前应用的全局id，生成方法见newId()
  Request req = new Request();
  req.setVersion(Version.getProtocolVersion());
  req.setTwoWay(true);
  req.setData(request);
  DefaultFuture future = new DefaultFuture(channel, req, timeout); //异步响应对象，返回给调用方，携带了唯一ID
  try {
      channel.send(req); //最终调用底层Netty等通讯框架进行数据传输
  } catch (RemotingException e) {
      future.cancel();
      throw e;
  }
  return future;
}

//INVOKE_ID为AtomicLong
private static long newId() {
    // getAndIncrement() When it grows to MAX_VALUE, it will grow to MIN_VALUE, and the negative can be used as ID
    return INVOKE_ID.getAndIncrement();
}
```

#### 7. 结果获取
此处讲两种处理方式：
1. 主动调用模式
异步调用模式下，则由用户调用ResponseFuture的 get 方法。通过RpcContext来获取上述DefaultFuture对象来获取请求结果，会阻塞至服务器端返产生结果给客户端。  

```java
//配置需要设置异步标识
<dubbo:reference id="xxx" ....>
    <dubbo:method name="method1" async="true" />
</dubbo:reference>

RpcContext.getContext().getFuture().get();
```
2. 事件通知模式
为了支持时间回调，dubbo引入了特定的过滤器FutureFilter来处理异步调用相关的逻辑。  

```java
//需要配置异步回调处理对象
<bean id="notify" class="com.alibaba.dubbo.callback.implicit.NofifyImpl" />
<dubbo:reference >
   <dubbo:method name="method1" async="true" onreturn="notify.onreturn" onthrow="notify.onthrow" oninvoke="notify.invoke" />
</dubbo:reference>
@Activate(group = Constants.CONSUMER)
public class FutureFilter implements Filter {

    @Override
    public Result invoke(final Invoker<?> invoker, final Invocation invocation) throws RpcException {
        final boolean isAsync = RpcUtils.isAsync(invoker.getUrl(), invocation);
        fireInvokeCallback(invoker, invocation);// 反射调用配置的通知对象，如：上面配置的Notify对象处理方法(--怎么感觉不是有结果后通知，而是发生远程调用回调通知？)
        Result result = invoker.invoke(invocation);
        if (isAsync) {
            asyncCallback(invoker, invocation);
        } else {
            syncCallback(invoker, invocation, result);
        }
        return result;
    }
    
}
```
在过滤器中，fireInvokeCallback去调用回调对象的方法做通知。asyncCallback进入口可以明显的看到和主动调用一样的代码，getFuture()异步等待：
```java
private void asyncCallback(final Invoker<?> invoker, final Invocation invocation) {
  Future<?> f = RpcContext.getContext().getFuture();
  if (f instanceof FutureAdapter) {
      ResponseFuture future = ((FutureAdapter<?>) f).getFuture();
      future.setCallback(new ResponseCallback() {
          @Override
          public void done(Object rpcResult) {
              if (rpcResult == null) {
                  logger.error(new IllegalStateException("invalid result value : null, expected " + Result.class.getName()));
                  return;
              }
              ///must be rpcResult
              if (!(rpcResult instanceof Result)) {
                  logger.error(new IllegalStateException("invalid result type :" + rpcResult.getClass() + ", expected " + Result.class.getName()));
                  return;
              }
              Result result = (Result) rpcResult;
              if (result.hasException()) {
                  fireThrowCallback(invoker, invocation, result.getException());
              } else {
                  fireReturnCallback(invoker, invocation, result.getValue());
              }
          }

          @Override
          public void caught(Throwable exception) {
              fireThrowCallback(invoker, invocation, exception);
          }
      });
  }
}
```
&emsp;&emsp;从上面可以看到，ResponseFuture future = ((FutureAdapter<?>) f).getFuture(); 和主动调用一样先获取Futrue，然后在当前未拿到响应的时候，先赋值给DefaultFuture的实例ResponseCallback对象作为异步处理回调。   
&emsp;&emsp;那么之后会在什么时候执行回调函数的方法呢？当consumer接收到provider的响应的时候！

#### 8. 异步回调结果接收
&emsp;&emsp;前面提到，客户端底层默认通过NettyClient来进行网络传输，在NettyClient#doOpen打开网络时，底层handler传入了NettyHandler/NettyClientHandler(netty4，同样可知，服务端为NettyServerHandler)对象，NettyHandler接收网络消息：
```java
@Override
public void messageReceived(ChannelHandlerContext ctx, MessageEvent e) throws Exception {
  NettyChannel channel = NettyChannel.getOrAddChannel(ctx.getChannel(), url, handler);
  try {
      handler.received(channel, e.getMessage());
  } finally {
      NettyChannel.removeChannelIfDisconnected(ctx.getChannel());
  }
}
```
HeaderExchangeHandler的received处理方法对于Response响应进行转调handleResponse：
```java
static void handleResponse(Channel channel, Response response) throws RemotingException {
  if (response != null && !response.isHeartbeat()) {
      DefaultFuture.received(channel, response);
  }
}
```
此处调用了DefaultFuture，也即在received方法中进行了**<font color='red'>异步转同步，其中，根据请求-响应头中携带的ID作为标识,获取Task任务</font>**
```java
public static void received(Channel channel, Response response) {
  try {
      DefaultFuture future = FUTURES.remove(response.getId()); 
      if (future != null) {
          future.doReceived(response);
      } else {
          logger.warn("The timeout response finally returned at "
                  + (new SimpleDateFormat("yyyy-MM-dd HH:mm:ss.SSS").format(new Date()))
                  + ", response " + response
                  + (channel == null ? "" : ", channel: " + channel.getLocalAddress()
                  + " -> " + channel.getRemoteAddress()));
      }
  } finally {
      CHANNELS.remove(response.getId());
  }
}

```
其中FUTURES为ConcurrentHashMap类型，通过唯一id进行查找。
```java
private void doReceived(Response res) {
  lock.lock();
  try {
      response = res;
      if (done != null) {
          done.signal();
      }
  } finally {
      lock.unlock();
  }
  if (callback != null) {
      invokeCallback(callback);
  }
}

```
doReceived中invokeCallback调用了前面FutureFilter#asyncCallback方法设置回调对象，进而最终在fireReturnCallback进行反射回调。
