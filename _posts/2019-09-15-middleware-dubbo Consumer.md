---
layout: post
title: "中间件-dubbo之服务端消息处理"
subtitle: '中间件之dubbo整理总结三'
author: "Kang"
date: 2019-09-15 18:19:02
header-img: "img/post-head-img/old-houses-4433982_1280.jpg"
catalog: true
tags:
  - 中间件
  - RPC
  - dubbo
---

### 1. 服务启动
#### 1.1 ServiceBean对象初始化
&emsp;&emsp;ServiceBean实现了ApplicationListener，在应用启动时，调用onApplicationEvent()方法。追踪：onApplicationEvent->export->doExport->ServiceConfig#doExportUrls->ServiceConfig#doExportUrlsFor1Protocol，该方法中存在:
```java
 Exporter<?> exporter = protocol.export(wrapperInvoker);
```
#### 1.2 服务打开
&emsp;&emsp;获取protocol对象类似消费端逻辑，获取ExtensionLoader的getAdaptiveExtension方法进行动态生成Protocol$Adaptive对象：
```java
private static final Protocol protocol = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();
```
&emsp;&emsp;Protocol$Adaptive中export方法也同样先返回DubboProtocol(默认时)，然后对该protocol进行Wrapper包装，其中参数invoker为包装后的DelegateProviderMetaDataInvoker。    
&emsp;&emsp;以DubboProtocol为例，在其export方法中，调用openServer进行服务的打开,其一个URL参数字符串：
```xml
dubbo://110.30.140.37:20880/com.example.service.IService?anyhost=true&application=study-example&bind.ip=110.30.140.37&bind.port=20880&dubbo=2.0.2&generic=false&interface=com.example.service.IService=slf4j&methods=getDetailList,queryAgreementPrepaid&pid=67958&revision=1.0.0&side=provider&timeout=2000&timestamp=1567420221411&version=1.0.0
```
调用链路openServer->createServer->Exchangers.bind，调用到SPI类HeaderExchanger：
```java
public static ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
    if (url == null) {
        throw new IllegalArgumentException("url == null");
    }
    if (handler == null) {
        throw new IllegalArgumentException("handler == null");
    }
    url = url.addParameterIfAbsent(Constants.CODEC_KEY, "exchange");
    return getExchanger(url).bind(url, handler);
}
```
bind动作返回HeaderExchangeServer对象，其封装了超时、服务器等信息：
```java
public class HeaderExchanger implements Exchanger {

    public static final String NAME = "header";

    @Override
    public ExchangeClient connect(URL url, ExchangeHandler handler) throws RemotingException {
        return new HeaderExchangeClient(Transporters.connect(url, new DecodeHandler(new HeaderExchangeHandler(handler))), true);
    }

    @Override
    public ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
        return new HeaderExchangeServer(Transporters.bind(url, new DecodeHandler(new HeaderExchangeHandler(handler))));
    }

}
```
&emsp;&emsp;Transporters.bind方法将Handler处理器绑定到底层通讯框架上，做服务监听，也即Handler存在代理关系：<font color="blue">DecodeHandler(HeaderExchangeHandler(ExchangeHandlerAdapter子类))</font>，故可以推测，<font color="red">整个消息的回复处理为DubboProtocol中ExchangeHandlerAdapter子类对象</font>


#### 2. 监听服务(消息接收)
&emsp;&emsp;NettyServer的父类构造函数中进行了doOpen()调用，传入网络处理器NettyHandler，其消息接收方法：
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
&emsp;&emsp;其中handler为上面提到的最外层DecodeHandler，最终到HeaderExchangeHandler#received。对于普通请求来说，使用handleRequest处理，最后到前面提到的DubboProtocol#ExchangeHandlerAdapter的子类做调用处理并回复：
```java
public class DubboProtocol extends AbstractProtocol {
    private ExchangeHandler requestHandler = new ExchangeHandlerAdapter() {
        public Object reply(ExchangeChannel channel, Object message) throws RemotingException {
            if (!(message instanceof Invocation)) {
                throw new RemotingException(channel, "Unsupported request: " + (message == null ? null : message.getClass().getName() + ": " + message) + ", channel: consumer: " + channel.getRemoteAddress() + " --> provider: " + channel.getLocalAddress());
            } else {
                Invocation inv = (Invocation)message;
                Invoker<?> invoker = DubboProtocol.this.getInvoker(channel, inv);
                if (Boolean.TRUE.toString().equals(inv.getAttachments().get("_isCallBackServiceInvoke"))) {
                    String methodsStr = (String)invoker.getUrl().getParameters().get("methods");
                    boolean hasMethod = false;
                    if (methodsStr != null && methodsStr.indexOf(",") != -1) {
                        String[] methods = methodsStr.split(",");
                        String[] arr$ = methods;
                        int len$ = methods.length;

                        for(int i$ = 0; i$ < len$; ++i$) {
                            String method = arr$[i$];
                            if (inv.getMethodName().equals(method)) {
                                hasMethod = true;
                                break;
                            }
                        }
                    } else {
                        hasMethod = inv.getMethodName().equals(methodsStr);
                    }

                    if (!hasMethod) {
                        DubboProtocol.this.logger.warn(new IllegalStateException("The methodName " + inv.getMethodName() + " not found in callback service interface ,invoke will be ignored." + " please update the api interface. url is:" + invoker.getUrl()) + " ,invocation is :" + inv);
                        return null;
                    }
                }

                RpcContext.getContext().setRemoteAddress(channel.getRemoteAddress());
                return invoker.invoke(inv);
            }
        }
}
```
