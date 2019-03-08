---
layout: default
title: dubbo源码解析-dubbo-rpc-api
---





dubbo源码解析-dubbo-rpc-api

<img src="https://qiuyuimg.oss-cn-beijing.aliyuncs.com/images/204734_9Xv8_113011.png" width="700px"> 



这是rpc模块的类图。是不是有点乱？别急我们慢慢整。

首先protocol是我们最核心的接口。主要功能是1:服务提供者暴露一个服务，2:消费者引用一个服务。

他下面有两个抽象类：AbstractProtocol和AbstractProxyProtocol；

Invoker这个暂时没想到怎么描述他。官方解释：它是 Dubbo 的核心模型，其它模型都向它靠扰，或转换成它，它代表一个可执行体，可向它发起 invoke 调用，它有可能是一个本地的实现，也可能是一个远程的实现，也可能一个集群实现。个人认为他是dubbo的主角，是服务的执行者，主要的服务执行方法invoke，参数是：Invocation服务的请求信息，结果对象Result；

ProtocolFilterWrapper是protocol装饰类。主要是对export和refer方法增加了一堆filter；



gx1、服务发布和消费端的转换过程：

	服务发布：

<img src="https://qiuyuimg.oss-cn-beijing.aliyuncs.com/images/dubbo_rpc_export.jpg" width="400px"> 

![16_23_54__03_08_2019](https://qiuyuimg.oss-cn-beijing.aliyuncs.com/images/16_23_54__03_08_2019.jpg)

	大致步骤就是 对外暴漏的rpc接口的实例 通过proxyfactory转换成invoker，在通过protocol把invoker转换成export。然后再将得到的export对外发布。

dubbo提供了两个ProxyFactory实现方式jdkproxy和javassist，默认的实现使用javassist，主要是因为jsvassist比jdkproxy效率快很多，具体参考：

http://javatar.iteye.com/blog/814426/

为了方便理解以JdkProxyFactory的代码为例

JdkProxyFactory：

```
public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) {
        return new AbstractProxyInvoker<T>(proxy, type, url) {
            @Override
            protected Object doInvoke(T proxy, String methodName,
                                      Class<?>[] parameterTypes,
                                      Object[] arguments) throws Throwable {
                Method method = proxy.getClass().getMethod(methodName, parameterTypes);
                //通过反射执行原方法
                return method.invoke(proxy, arguments);
            }
        };
    }
```

DubboProtocol：

```
public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
    URL url = invoker.getUrl();

    // export service.
    String key = serviceKey(url);
    DubboExporter<T> exporter = new DubboExporter<T>(invoker, key, exporterMap);
    //保存布过的serviceKey和Exporter（远程服务发布的引用）的映射表
    exporterMap.put(key, exporter);

    //export an stub service for dispaching event
    Boolean isStubSupportEvent = url.getParameter(Constants.STUB_EVENT_KEY, Constants.DEFAULT_STUB_EVENT);
    Boolean isCallbackservice = url.getParameter(Constants.IS_CALLBACK_SERVICE, false);
    if (isStubSupportEvent && !isCallbackservice) {
        String stubServiceMethods = url.getParameter(Constants.STUB_EVENT_METHODS_KEY);
        if (stubServiceMethods == null || stubServiceMethods.length() == 0) {
            if (logger.isWarnEnabled()) {
                logger.warn(new IllegalStateException("consumer [" + url.getParameter(Constants.INTERFACE_KEY) +
                        "], has set stubproxy support event ,but no stub methods founded."));
            }
        } else {
            stubServiceMethodsMap.put(url.getServiceKey(), stubServiceMethods);
        }
    }

    openServer(url);

    return exporter;
}
```



服务消费：

<img src="https://qiuyuimg.oss-cn-beijing.aliyuncs.com/images/dubbo_rpc_refer.jpg" width="400px"> 

![16_24_25__03_08_2019](https://qiuyuimg.oss-cn-beijing.aliyuncs.com/images/16_24_25__03_08_2019.jpg)

大致步骤就是 通过protocol把远程服务（简单理解成url）转换成invoker。然后再通过proxyfactory将invoker转成我们请求的接口。

DubboProtocol

```
public <T> Invoker<T> refer(Class<T> serviceType, URL url) throws RpcException {
    // create rpc invoker.
    DubboInvoker<T> invoker = new DubboInvoker<T>(serviceType, url, getClients(url), invokers);
    invokers.add(invoker);
    return invoker;
}
```

JdkProxyFactory

```
public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
    return (T) Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader(), interfaces, new InvokerInvocationHandler(invoker));
}
```

整体的请求过程大概就是这样：

<img src="https://qiuyuimg.oss-cn-beijing.aliyuncs.com/images/dubbo_rpc_invoke.jpg" width="600px"> 



下一步围绕Protocol和Invoker看看他们的具体的实现过程。

