---
layout:     post
title:      "多线程编程中的\"坑\""
subtitle:   "近期遇到的多线程bug总结"
date:       2016-03-10 12:00:00
author:     "Thh"
tags:
    - 多线程
---

最近工作中连续碰到几个涉及多线程方面的bug，在这总结梳理一下，就当提醒自己别犯同样的错误。

## Bug 1 - 狂转的CPU

同事的一个项目上线的时候，发现CPU占用率奇高，达到700%，而平常的时候，也就100%左右。用jstack查看线程栈，发现很多线程都卡在一个名为`waitUntilInited()`的方法里面。查看代码，发现这个方法是这样的：  

	private boolean inited = false;
	...
	void waitUntilInited() {
		while(!inited) {
			;
		}
	}

有一个线程会执行一些初始化操作，初始化完成会将inited变量赋值为true；而业务线程调用`waitUntilInited()`方法等待初始化完成才能执行操作。说到这里，bug已经很明显了。这是典型的没使用volatile导致的线程可见性bug。这个bug的情况比较简单，由于一直在做循环，比较容易定位到问题所在。但有时候由于可见性问题造成的bug会比这个诡异得多，因此在写多线程程序的时候要特别留心共享变量的可见性。

## Bug 2 - 忽隐忽现的地址已绑定异常

最近同事的项目在启动的时候，Dubbo服务打开端口偶尔会出现地址已经被绑定异常(`java.net.BindException`)。出现异常的代码在`com.alibaba.dubbo.rpc.protocol.dubbo.DubboProtocol`类里面，其中创建server的方法是这样的：

	public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
		URL url = invoker.getUrl();
		...
		openServer(url);
		return invoker;
	}
	
	private void openServer(URL url) {
        // find server.
        String key = url.getAddress();
        //client 也可以暴露一个只有server可以调用的服务。
        boolean isServer = url.getParameter(Constants.IS_SERVER_KEY,true);
        if (isServer) {
        	ExchangeServer server = serverMap.get(key);
        	if (server == null) {
        		serverMap.put(key, createServer(url));
        	} else {
        		//server支持reset,配合override功能使用
        		server.reset(url);
        	}
        }
    }

由于应用为服务配置了延迟暴露，而延迟暴露实现方式是另起一个线程，sleep一段时间，然后再暴露方法，这就导致会并发调用上面的`export()`方法，进而间接并发地调用`createServer()`，最终导致多次绑定同一个地址的异常。解决的办法很简单，为`openServer()`方法加上`synchronized`关键字即可；或者使用`synchronized`块，将锁的粒度减小。

这种bug比较隐蔽，因为`serverMap`是一个`ConcurrentHashMap`，很多人以为使用了`ConcurrentHashMap`就是线程安全的，而且在创建server之前先在map中查询了一次，如果没有才会创建，所以应该没有问题。但没有意识到`ConcurrentHashMap`保证的只是map内部的操作是同步的，不能一次get()操作和一次紧邻的put操作也是同步的，所以必须在外部加上同步措施。

## Bug 3 - 神出鬼没的CompileError

也是一个同事的项目，在启动的时候偶尔会出现下面的异常：

	Caused by: java.lang.RuntimeException: [source error] no such class: com.alibaba.dubbo.common.bytecode.proxy2
	at com.alibaba.dubbo.common.bytecode.ClassGenerator.toClass(ClassGenerator.java:354) ~[dubbo-yiji-2.5.13.jar:yiji-2.5.13]
	at com.alibaba.dubbo.common.bytecode.ClassGenerator.toClass(ClassGenerator.java:293) ~[dubbo-yiji-2.5.13.jar:yiji-2.5.13]
	at com.alibaba.dubbo.common.bytecode.Proxy.getProxy(Proxy.java:214) ~[dubbo-yiji-2.5.13.jar:yiji-2.5.13]
	at com.alibaba.dubbo.common.bytecode.Proxy.getProxy(Proxy.java:67) ~[dubbo-yiji-2.5.13.jar:yiji-2.5.13]
	at com.alibaba.dubbo.rpc.proxy.javassist.JavassistProxyFactory.getProxy(JavassistProxyFactory.java:35) ~[dubbo-yiji-2.5.13.jar:yiji-2.5.13]
	at com.alibaba.dubbo.rpc.proxy.AbstractProxyFactory.getProxy(AbstractProxyFactory.java:49) ~[dubbo-yiji-2.5.13.jar:yiji-2.5.13]
	at com.alibaba.dubbo.rpc.proxy.wrapper.StubProxyFactoryWrapper.getProxy(StubProxyFactoryWrapper.java:60) ~[dubbo-yiji-2.5.13.jar:yiji-2.5.13]
	at com.alibaba.dubbo.rpc.ProxyFactory$Adpative.getProxy(ProxyFactory$Adpative.java) ~[dubbo-yiji-2.5.13.jar:yiji-2.5.13]
	at com.alibaba.dubbo.config.ReferenceConfig.createProxy(ReferenceConfig.java:431) ~[dubbo-yiji-2.5.13.jar:yiji-2.5.13]
	at com.alibaba.dubbo.config.ReferenceConfig.init(ReferenceConfig.java:305) ~[dubbo-yiji-2.5.13.jar:yiji-2.5.13]
	at com.alibaba.dubbo.config.ReferenceConfig.get(ReferenceConfig.java:139) ~[dubbo-yiji-2.5.13.jar:yiji-2.5.13]
	at com.alibaba.dubbo.config.spring.AnnotationBean$2.call(AnnotationBean.java:296) ~[dubbo-yiji-2.5.13.jar:yiji-2.5.13]
	at com.alibaba.dubbo.config.spring.AnnotationBean$2.call(AnnotationBean.java:293) ~[dubbo-yiji-2.5.13.jar:yiji-2.5.13]
	... 4 common frames omitted
	
	Caused by: javassist.CannotCompileException: [source error] no such class: com.alibaba.dubbo.common.bytecode.proxy2
	at javassist.CtNewMethod.make(CtNewMethod.java:79) ~[javassist-3.18.1-GA.jar:na]
	at javassist.CtNewMethod.make(CtNewMethod.java:45) ~[javassist-3.18.1-GA.jar:na]
	at 	com.alibaba.dubbo.common.bytecode.ClassGenerator.toClass(ClassGenerator.java:322) ~[dubbo-yiji-2.5.13.jar:yiji-2.5.13]
	... 16 common frames omitted

	Caused by: javassist.compiler.CompileError: no such class: com.alibaba.dubbo.common.bytecode.proxy2
	at javassist.compiler.MemberResolver.searchImports(MemberResolver.java:468) ~[javassist-3.18.1-GA.jar:na]
	at javassist.compiler.MemberResolver.lookupClass(MemberResolver.java:412) ~[javassist-3.18.1-GA.jar:na]
	at javassist.compiler.MemberResolver.lookupClassByName(MemberResolver.java:315) ~[javassist-3.18.1-GA.jar:na]
	at javassist.compiler.TypeChecker.atNewExpr(TypeChecker.java:146) ~[javassist-3.18.1-GA.jar:na]
	at javassist.compiler.ast.NewExpr.accept(NewExpr.java:73) ~[javassist-3.18.1-GA.jar:na]
	at javassist.compiler.CodeGen.doTypeCheck(CodeGen.java:242) ~[javassist-3.18.1-GA.jar:na]
	at javassist.compiler.CodeGen.compileExpr(CodeGen.java:229) ~[javassist-3.18.1-GA.jar:na]
	at javassist.compiler.CodeGen.atReturnStmnt2(CodeGen.java:598) ~[javassist-3.18.1-GA.jar:na]
	at javassist.compiler.JvstCodeGen.atReturnStmnt(JvstCodeGen.java:425) ~[javassist-3.18.1-GA.jar:na]
	at javassist.compiler.CodeGen.atStmnt(CodeGen.java:363) ~[javassist-3.18.1-GA.jar:na]
	at javassist.compiler.ast.Stmnt.accept(Stmnt.java:50) ~[javassist-3.18.1-GA.jar:na]
	at javassist.compiler.CodeGen.atStmnt(CodeGen.java:351) ~[javassist-3.18.1-GA.jar:na]
	at javassist.compiler.ast.Stmnt.accept(Stmnt.java:50) ~[javassist-3.18.1-GA.jar:na]
	at javassist.compiler.CodeGen.atMethodBody(CodeGen.java:292) ~[javassist-3.18.1-GA.jar:na]
	at javassist.compiler.CodeGen.atMethodDecl(CodeGen.java:274) ~[javassist-3.18.1-GA.jar:na]
	at javassist.compiler.ast.MethodDecl.accept(MethodDecl.java:44) ~[javassist-3.18.1-GA.jar:na]
	at javassist.compiler.Javac.compileMethod(Javac.java:169) ~[javassist-3.18.1-GA.jar:na]
	at javassist.compiler.Javac.compile(Javac.java:95) ~[javassist-3.18.1-GA.jar:na]
	at javassist.CtNewMethod.make(CtNewMethod.java:74) ~[javassist-3.18.1-GA.jar:na]
	... 18 common frames omitted

由于异常不是每次启动都出现，所以推测可能和多线程有关。查看源码，发现`com.alibaba.dubbo.common.bytecode.ClassGenerator`类的`getClassPool()`方法有问题。

	private static final Map<ClassLoader, ClassPool> POOL_MAP = new ConcurrentHashMap<ClassLoader, ClassPool>();
	
	public static ClassGenerator newInstance()
	{
		return new ClassGenerator(getClassPool(Thread.currentThread().getContextClassLoader()));
	}
	
	public static ClassPool getClassPool(ClassLoader loader)
	{
		if( loader == null )
			return ClassPool.getDefault();

		ClassPool pool = POOL_MAP.get(loader);
		if( pool == null )
		{
			pool = new ClassPool(true);
			pool.appendClassPath(new LoaderClassPath(loader));
			POOL_MAP.put(loader, pool);
		}
		return pool;
	}
	
其中`getClassPool()`方法不是线程安全的。作者用一个`ConcurrentHashMap`保存每个ClassLoader对应的ClassPool。和Bug2情况类似，作者也是先get()一下，如果没有，就创建一个，然后再put()回去。这个过程没有加锁，如果第一个线程get()发现没有，紧接着第二个线程用同样的key也来get()，这时候还是没有，然后第一个线程创建ClassPool放进map, 第二个线程也新建一个ClassPool放进map，就会把第一个线程的ClassPool覆盖，造成第一个线程创建的proxy class找不到。

解决办法有多种，可以使用锁同步get()、put()操作，也可以在put的时候，使用`putIfAbsent()`方法，这样就不会覆盖已经创建好的ClassPool，然后get()到最新的value返回。

为什么之前没有发现这个bug呢？其实用官方的dubbo版本，不会出现问题，因为ReferenceBean的初始化是单线程的。最近公司内部维护的版本优化使用多线程来初始化
ReferenceBean，才导致上述bug暴露出来。所以有时候程序运行正常不代表没有bug，开发和测试的时候应该尽量覆盖更多的使用场景，尽量减少隐藏bug的可能性。



