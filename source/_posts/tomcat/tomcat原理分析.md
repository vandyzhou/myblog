---
layout: tomcat
title: tomcat原理分析
date: 2021-06-23 15:03:04
tags: 源码分析
catogories: tomcat
---

## 整体架构

![tomcat整体架构-1](/imgs/tomcat/tomcat-core/tomcat-core-1.png)

![tomcat整体架构-2](/imgs/tomcat/tomcat-core/tomcat-core-2.png)

从上图我们看出，最核心的两个组件–连接器（Connector）和容器（Container）起到心脏的作用，他们至关重要！下面我们逐一来分析其功能：

* Server表示服务器，提供了一种优雅的方式来启动和停止整个系统，不必单独启停连接器和容器
* Service表示服务，Server可以运行多个服务。比如一个Tomcat里面可运行订单服务、支付服务、用户服务等等
* 每个Service可包含多个Connector和一个Container。因为每个服务允许同时支持多种协议，但是每种协议最终执行的Servlet却是相同的
* Connector表示连接器，比如一个服务可以同时支持AJP协议、Http协议和Https协议，每种协议可使用一种连接器来支持
* Container表示容器，可以看做Servlet容器
	- Engine – 引擎
	- Host – 主机
	- Context – 上下文
	- Wrapper – 包装器
* Service服务之下还有各种支撑组件，下面简单罗列一下这些组件
	- Manager – 管理器，用于管理会话Session
	- Logger – 日志器，用于管理日志
	- Loader – 加载器，和类加载有关，只会开放给Context所使用
	- Pipeline – 管道组件，配合Valve实现过滤器功能
	- Valve – 阀门组件，配合Pipeline实现过滤器功能
	- Realm – 认证授权组件

除了连接器和容器，管道组件和阀门组件也很关键，我们通过一张图来看看这两个组件

![组件](/imgs/tomcat/tomcat-core/tomcat-core-3.png)

## 启动

catalina.sh文件下的start ---> 执行Bootstrap.start

``` shell
eval $_NOHUP "\"$_RUNJAVA\"" "\"$CATALINA_LOGGING_CONFIG\"" $LOGGING_MANAGER "$JAVA_OPTS" "$CATALINA_OPTS" \
      -D$ENDORSED_PROP="\"$JAVA_ENDORSED_DIRS\"" \
      -classpath "\"$CLASSPATH\"" \
      -Djava.security.manager \
      -Djava.security.policy=="\"$CATALINA_BASE/conf/catalina.policy\"" \
      -Dcatalina.base="\"$CATALINA_BASE\"" \
      -Dcatalina.home="\"$CATALINA_HOME\"" \
      -Djava.io.tmpdir="\"$CATALINA_TMPDIR\"" \
      org.apache.catalina.startup.Bootstrap "$@" start \
      >> "$CATALINA_OUT" 2>&1 "&"
```

## 类图

![类图](/imgs/tomcat/tomcat-core/tomcat-core-4.png)

### Bootstrap.start()

* init()
	- createClassLoader
		common classloader：通过环境变量common.loader配置，默认：${catalina.base}/lib","${catalina.base}/lib/*.jar","${catalina.home}/lib","${catalina.home}/lib/*.jar；指定的parent为null；
		server classloader：通过环境变量server.loader配置，默认：空；parent为common classloader，命名为catalinaLoader
		shared classloader：通过环境变量shared.loader配置，默认：空；parent为common classloader
	- 设置当前线程的classloader为catalinaLoader
		Thread.currentThread().setContextClassLoader(catalinaLoader);
	- pre load security class
		SecurityClassLoad.securityClassLoad(catalinaLoader);
	- 初始化Catalina.class，并反射调用setParentClassLoader将parentClassLoader设置为sharedLoader
	- 反射调用Catalina.load()方法
* 反射调用Catalina.start()方法

* Catalina.load()
	- initNaming()
	- 解析server.xml获取到Server
		使用Digester进行server.xml解析，并将对象赋值给Server对象；Tomcat Digester使用
	- 执行Server.init()
		![uml](/imgs/tomcat/tomcat-core/tomcat-core-5.png)
		![调用关系](/imgs/tomcat/tomcat-core/tomcat-core-6.png)
	- 做了如下几件事：
		Server的New-->INITIALIZED的变更&事件的监听者的触发；
		NamingResource的New-->INITIALIZED的变更&事件的监听者的触发；
		Service的初始化；
	- 执行service.init()
		+ engine.init()
			1. getRealm()：server.xml里面配置的engine的realm是：LockOutRealm、UserDatabaseRealm（内存）；Realm主要是做一些权限认证；
			2. engine的状态New-->INITIALIZED的变更&事件的监听者的触发；
			3. 注册MBean；type=Executor,name=name
		+ executors.init()
			1. executor的状态New-->INITIALIZED的变更&事件的监听者的触发；
			2. 注册MBean；type=Engine
		+ mapperListener.init()
			1. mapperListener的状态New-->INITIALIZED的变更&事件的监听者的触发；
			2. 注册MBean；type=Mapper；
		+ connectors.init()
			1. 注册MBean；type=Connector,port=,address=；
			2. protocalHandler.init()
				- endpoint.init()
* Catalina.start()
	- server.start()
		* service.start()
			1. engine.start()
				- cluster.start()
				- realm.start()
				- (Container)child.start()：异步
				- pipeline.start()
				- utilityExecutorWrapper周期执行monitor；执行container下面children的backgroundProcess()方法；Context使用WebappClassLoader进行加载
			2. executor.start()
				- 初始化线程池：prefix=tomcat-exec、core=25、max=200、keepAliveTime=60000ms、queueSize = Integer.Max_Value；
			3. mapperListener.start()
			4. connector.start()
				- protocolHandler.start()
					* endpoint.start()
						+ 创建SynchronizedStack<SocketProcessorBase<S\>\> processorCache
						+ 创建SynchronizedStack<PollerEvent\> eventCache
						+ 创建SynchronizedStack<NioChannel\> nioChannels
						+ 创建ThreadPoolExecutor，prefix=TP\-exec\-、core=10、max=200、keepAliveTime=60s、queueSize = Integer.Max_Value
						+ 创建LimitLatch，size=8\*1024、类似countdownLatch
						+ 创建poller对象；是个runnable，线程名TP\-Poller，后台运行；就是socket编程里面的selector，维护着socket的链接
						+ 创建acceptor对象；也是个runnable，线程名TP-Acceptor，后台运行；就是socket编程里面的acceptor
		* utilityExecutorWrapper周期执行发送event = periodic 事件
	- generateCode = true; 默认为false，会动态生成类DigesterGeneratedCodeLoader
	- add Shutdown hook：关闭时会触发Catalina.stop()
	- await=true；默认false；
		+ server.await()
		+ server.stop() & server.destroy()

[参考链接](https://juconcurrent.com/2018/11/22/deep-understand-tomcat-008-container/)














