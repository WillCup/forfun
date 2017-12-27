
title: JMX-与docker的问题
date: 2017-09-22 14:44:39
tags: [youdaonote]
---

docker容器依赖于外部某些服务，运行一段时间后网络不通

我的所有的采样jmx数据的容器全都是这样，开始的时候好好的，一段时间后就连不上目标主机了。然后重启又恢复正常



下面是使用prometheus/jmx_exporter自己制作docker镜像后产生的问题。
报错异常如下：
```
SEVERE: JMX scrape failed: java.io.IOException: Failed to retrieve RMIServer stub: javax.naming.ServiceUnavailableException [Root exception is java.rmi.ConnectException: Connection refused to host: 10.2.19.64; nested exception is: 
	java.net.ConnectException: Connection refused (Connection refused)]
	at javax.management.remote.rmi.RMIConnector.connect(RMIConnector.java:369)
	at javax.management.remote.JMXConnectorFactory.connect(JMXConnectorFactory.java:270)
	at io.prometheus.jmx.JmxScraper.doScrape(JmxScraper.java:106)
	at io.prometheus.jmx.JmxCollector.collect(JmxCollector.java:436)
	at io.prometheus.client.CollectorRegistry$MetricFamilySamplesEnumeration.findNextElement(CollectorRegistry.java:180)
	at io.prometheus.client.CollectorRegistry$MetricFamilySamplesEnumeration.nextElement(CollectorRegistry.java:213)
	at io.prometheus.client.CollectorRegistry$MetricFamilySamplesEnumeration.nextElement(CollectorRegistry.java:134)
	at io.prometheus.client.exporter.common.TextFormat.write004(TextFormat.java:22)
	at io.prometheus.client.exporter.HTTPServer$HTTPMetricHandler.handle(HTTPServer.java:43)
	at com.sun.net.httpserver.Filter$Chain.doFilter(Filter.java:79)
	at sun.net.httpserver.AuthFilter.doFilter(AuthFilter.java:83)
	at com.sun.net.httpserver.Filter$Chain.doFilter(Filter.java:82)
	at sun.net.httpserver.ServerImpl$Exchange$LinkHandler.handle(ServerImpl.java:675)
	at com.sun.net.httpserver.Filter$Chain.doFilter(Filter.java:79)
	at sun.net.httpserver.ServerImpl$Exchange.run(ServerImpl.java:647)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
	at java.lang.Thread.run(Thread.java:745)
Caused by: javax.naming.ServiceUnavailableException [Root exception is java.rmi.ConnectException: Connection refused to host: 10.2.19.64; nested exception is: 
	java.net.ConnectException: Connection refused (Connection refused)]
	at com.sun.jndi.rmi.registry.RegistryContext.lookup(RegistryContext.java:122)
	at com.sun.jndi.toolkit.url.GenericURLContext.lookup(GenericURLContext.java:205)
	at javax.naming.InitialContext.lookup(InitialContext.java:417)
	at javax.management.remote.rmi.RMIConnector.findRMIServerJNDI(RMIConnector.java:1955)
	at javax.management.remote.rmi.RMIConnector.findRMIServer(RMIConnector.java:1922)
	at javax.management.remote.rmi.RMIConnector.connect(RMIConnector.java:287)
	... 17 more
Caused by: java.rmi.ConnectException: Connection refused to host: 10.2.19.64; nested exception is: 
	java.net.ConnectException: Connection refused (Connection refused)
	at sun.rmi.transport.tcp.TCPEndpoint.newSocket(TCPEndpoint.java:619)
	at sun.rmi.transport.tcp.TCPChannel.createConnection(TCPChannel.java:216)
	at sun.rmi.transport.tcp.TCPChannel.newConnection(TCPChannel.java:202)
	at sun.rmi.server.UnicastRef.newCall(UnicastRef.java:342)
	at sun.rmi.registry.RegistryImpl_Stub.lookup(Unknown Source)
	at com.sun.jndi.rmi.registry.RegistryContext.lookup(RegistryContext.java:118)
	... 22 more
Caused by: java.net.ConnectException: Connection refused (Connection refused)
	at java.net.PlainSocketImpl.socketConnect(Native Method)
	at java.net.AbstractPlainSocketImpl.doConnect(AbstractPlainSocketImpl.java:350)
	at java.net.AbstractPlainSocketImpl.connectToAddress(AbstractPlainSocketImpl.java:206)
	at java.net.AbstractPlainSocketImpl.connect(AbstractPlainSocketImpl.java:188)
	at java.net.SocksSocketImpl.connect(SocksSocketImpl.java:392)
	at java.net.Socket.connect(Socket.java:589)
	at java.net.Socket.connect(Socket.java:538)
	at java.net.Socket.<init>(Socket.java:434)
	at java.net.Socket.<init>(Socket.java:211)
	at sun.rmi.transport.proxy.RMIDirectSocketFactory.createSocket(RMIDirectSocketFactory.java:40)
	at sun.rmi.transport.proxy.RMIMasterSocketFactory.createSocket(RMIMasterSocketFactory.java:148)
	at sun.rmi.transport.tcp.TCPEndpoint.newSocket(TCPEndpoint.java:613)
	... 27 more
```
