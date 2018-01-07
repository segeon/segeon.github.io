---
layout:     post
title:      "如何实现一个可用的javaagent"
date:       2016-01-23 12:00:00
author:     "Thh"
tags:
    - javaagent
---

最近做了一个项目需要用javaagent方式对应用常用的组件（比如httpclient, 数据库连接池等）进行调用追踪和监控，并结合公司的分布式追踪组件，将所有java应用的外部调用情况收集起来方便做系统分析和问题定位。项目定位和开源项目[pinpoint](https://github.com/naver/pinpoint)比较像，但了解过pinpoint实现以后，发现其分布式追踪和组件监控的逻辑耦合太过紧密，而且整个项目比较重，实现繁杂，不容易和公司的分布式追踪组件结合起来，所以决定自己搞。这里暂且把这个项目取名叫dagent。

网上其实有很多文章介绍如何编写javaagent，但往往介绍得非常简单，只介绍premain的启动机制，manifest如何编写，但这类文章都没有说明简单实现的javaagent能否实际发挥作用，在实际的项目中可能会有哪些坑。所以，我想把这次项目过程中踩过的坑记录下来，分享给需要的人。

## ClassLoader之殇
首先得从spring boot的uber jar说起。所谓uber jar，就是一个all in one的可执行jar包。jar包中包含了Java应用运行所需要的代码、资源以及依赖的jar包，直接执行`java -jar xxx.jar`即可启动。对于web应用，spring boot还提供了嵌入式的web容器，无需部署tomcat服务器，应用部署运行特别方便。所以最近公司开始采用spring boot。

在测试dagent时发现，对于使用spring boot框架的应用，直接在IDE里面执行main方法运行dagent工作ok，一旦打成uber jar方式后加上dagent启动，就会出现dagent中引用的第三方包中的类报`ClassNotFoundException`。

查看spring boot源码，发现了原因所在。原来，由于uber jar将应用依赖的jar包以nested jar的方式打进包内，为了实现不解压缩就启动，spring boot使用自己的main类`org.springframework.boot.loader.JarLauncher`启动应用，并自定义一个`LaunchedURLClassLoader`，再由它加载应用的main类。而`LaunchedURLClassLoader`在初始化classpath搜索路径时特意把javaagent jar包排除在外，所以javaagent jar包中的类是不能被`LaunchedURLClassLoader`定义的，所以javaagent中的辅助类如果引用了某个第三方包中的类，而这个类是被LaunchedURLClassLoader定义的，简单引用就会出现`ClassNotFoundException`。

所以，必须让javaagent中用到的辅助类也由定义当前正在增强的Class的`LaunchedURLClassLoader`定义。  

	public interface ClassFileTransformer {
		byte[] transform(  
		         ClassLoader         loader,
                String              className,
                Class<?>            classBeingRedefined,
                ProtectionDomain    protectionDomain,
                byte[]              classfileBuffer)
       throws IllegalClassFormatException;
	}

更确切地说，应该让每一个被`ClassFileTransformer`修改的类所用到的自定义以及第三方辅助类都由`ClassFileTransformer#transform()`方法第一个参数的ClassLoader定义。这样，不管应用的ClassLoader采用了何种类查找策略，都可以保证辅助类可以正常加载到。

## ClassLoader注入
那么，如何做到让增强类所用到的辅助类都被同一个类定义呢。一种办法是显示地用对应的ClassLoader定义所有用到的辅助类，这样需要手动注入所有辅助类，比较繁琐；另一种办法是将一组功能相关的辅助类打成jar包，注入到对应的ClassLoader中，这样就不需要一个一个类手动注入。dagent采用的是第二种方案，将一组功能相关的增强辅助类做成一个插件，并打成一个jar包，然后在增强类的时候，将对应的jar包注入到当前执行增强的ClassLoader中。  

![dagent工作原理](/img/in-post/dagent-workflow.png)

不同类型的ClassLoader的注入方式有所不同，方法如下：  

	public class ClassInjector {
		private static Method DEFINE_CLASS;
    	private static Method ADD_URL;

    	static {
	        try {
	            DEFINE_CLASS = ClassLoader.class.getDeclaredMethod("defineClass", String.class, byte[].class, int.class, int.class);
	            DEFINE_CLASS.setAccessible(true);

	            ADD_URL = URLClassLoader.class.getDeclaredMethod("addURL", URL.class);
	            ADD_URL.setAccessible(true);
	        } catch (NoSuchMethodException e) {
	            throw new IllegalStateException(e);
	        }
    	}

	    /**
	     * 注入到非URLClassLoader的非引导类ClassLoader
	     * @param classLoader
	     * @param className
	     * @param bytes 类定义
	     * @throws InvocationTargetException
	     * @throws IllegalAccessException
	     */
	    public static void defineClass(ClassLoader classLoader, String className, byte[] bytes) throws InvocationTargetException, IllegalAccessException {
	        if (classLoader != null) {
	            DEFINE_CLASS.invoke(classLoader, className, bytes, 0, bytes.length);
	        }
	    }

	    /**
	     * 注入到URLClassLoader类加载器
	     * @param classLoader
	     * @param url
	     * @throws InvocationTargetException
	     * @throws IllegalAccessException
	     */
	    public static void addURL(URLClassLoader classLoader, URL url) throws InvocationTargetException, IllegalAccessException {
	        ADD_URL.invoke(classLoader, url);
	    }

	    /**
	     * 注入到引导类加载器
	     * @param instrumentation
	     * @param jarFile
	     */
	    public static void addURL(Instrumentation instrumentation, JarFile jarFile) {
	        instrumentation.appendToBootstrapClassLoaderSearch(jarFile);
	    }
	}
	
## 打包javaagent

采用上面的思路将javaagent用插件的方式实现，会导致每个插件都是一个jar包，不方便部署。可以用`maven-assembly-plugin`将javaagent核心代码和插件打成一个jar包，agent加载时，再将jar包解压，取出内嵌的插件包。解压后的dagent包结构如下：
![javaagent包结构](/img/in-post/dagent-jar-structure.png)
或者，也可以借鉴spring-boot uber jar的处理方式，自定义Jar包结构和URL handler，这样就可以直接加载内嵌的jar包，不需要先解压。









