---
layout: post
title: "多线程-如何确定线程池大小"
subtitle: '如何确定线程池大小'
author: "Kang"
date: 2019-09-17 22:58:19
header-img: "img/post-head-img/post-bg-unix-linux.jpg"
catalog: true
tags:
  - 多线程
---

”并发线程池到底设置多大呢？”   
通常有点年纪的程序员或许都听说这样一个说法 （其中 N 代表 CPU 的个数）：
>
1. CPU 密集型应用，线程池大小设置为 N + 1
2. IO 密集型应用，线程池大小设置为 2N

如果一台服务器上只部署这一个应用并且只有这一个线程池，那么这种估算或许合理，但是在实际应用中是不可能的。  

### 怎么设置线程池大小呢
在网上相关博客介绍到“服务器性能IO优化”中存在一个估算公式：   
>最佳线程数目 = （（线程等待时间+线程CPU时间）/线程CPU时间 ）* CPU数目

比如平均每个线程CPU运行时间为0.5s，而线程等待时间（非CPU运行时间，比如IO）为1.5s，CPU核心数为8，那么根据上面这个公式估算得到：((0.5+1.5)/0.5)*8=32。这个公式进一步转化为：  
>最佳线程数目 = （线程等待时间与线程CPU时间之比 + 1）* CPU数目

可以看出与前面一般说法相同的结论：
**线程等待时间所占比例越高，需要越多线程。线程CPU时间所占比例越高，需要越少线程。**  
&emsp;&emsp;一个系统最快的部分是CPU，所以决定一个系统吞吐量上限的是CPU。增强CPU处理能力，可以提高系统吞吐量上限。但根据短板效应，真实的系统吞吐量并不能单纯根据CPU来计算。那要提高系统吞吐量，就需要从“系统短板”（比如网络延迟、IO）着手：  
→ 尽量提高短板操作的并行化比率，比如多线程下载技术  
→ 增强短板能力，比如用NIO替代IO  
#### 最佳实践
&emsp;&emsp;通过公式，我们了解到需要 3 个具体数值
1. 一个请求所消耗的时间 (线程等待时间 + 线程CPU时间)
2. 该请求计算时间 （线程CPU时间）
3. CPU 数目

##### 请求消耗时间
&emsp;&emsp;我们可以通过拦截器或者相关的测试代码来统计一个请求从进入到得到结果的整个时长，下面以一个Web请求过程为例：  
```java 
public class MoniterFilter implements Filter {

    private static final Logger logger = LoggerFactory.getLogger(MoniterFilter.class);

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException,
            ServletException {
        long start = System.currentTimeMillis();

        HttpServletRequest httpRequest = (HttpServletRequest) request;
        HttpServletResponse httpResponse = (HttpServletResponse) response;
        String uri = httpRequest.getRequestURI();
        String params = getQueryString(httpRequest);

        try {
            chain.doFilter(httpRequest, httpResponse);
        } finally {
            long cost = System.currentTimeMillis() - start;
            logger.info("access url [{}{}], cost time [{}] ms )", uri, params, cost);
        }

    private String getQueryString(HttpServletRequest req) {
        StringBuilder buffer = new StringBuilder("?");
        Enumeration<String> emParams = req.getParameterNames();
        try {
            while (emParams.hasMoreElements()) {
                String sParam = emParams.nextElement();
                String sValues = req.getParameter(sParam);
                buffer.append(sParam).append("=").append(sValues).append("&");
            }
            return buffer.substring(0, buffer.length() - 1);
        } catch (Exception e) {
            logger.error("get post arguments error", buffer.toString());
        }
        return "";
    }
}
```

##### CPU 计算时间
>CPU 计算时间 = 请求总耗时 - CPU IO time  

&emsp;&emsp;假设该请求有一个查询 DB 的操作，只要知道这个查询 DB 的耗时（CPU IO time），计算的时间不就出来了嘛，我们看一下怎么才能简洁，明了的记录 DB 查询的耗时。通过（JDK 动态代理/ CGLIB）的方式添加 AOP 切面，来获取线程 IO 耗时。代码如下，请参考:    
```java
public class DaoInterceptor implements MethodInterceptor {

    private static final Logger logger = LoggerFactory.getLogger(DaoInterceptor.class);

    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        StopWatch watch = new StopWatch();
        watch.start();
        Object result = null;
        Throwable t = null;
        try {
            result = invocation.proceed();
        } catch (Throwable e) {
            t = e == null ? null : e.getCause();
            throw e;
        } finally {
            watch.stop();
            logger.info("({}ms)", watch.getTotalTimeMillis());

        }

        return result;
    }

}
```
##### CPU 数目
&emsp;&emsp;逻辑 CPU 个数 ，设置线程池大小的时候参考的 CPU 个数
```shell
cat /proc/cpuinfo| grep "processor"| wc -l
```

## 总结
&emsp;&emsp;合适的配置线程池大小其实很不容易，但是通过上述的公式和具体代码，我们就能快速、落地的算出这个线程池该设置的多大。不过最后的最后，我们还是需要通过压力测试来进行微调，只有经过压测测试的检验，我们才能最终保证的配置大小是准确的。
  
