---
layout:     post
title:      "Dubbo中的设计模式"
date:       2016-03-20 10:00:00
author:     "Thh"
tags:
    - 设计模式
---

最近在看阿里开源RPC框架Dubbo的源码，顺带梳理了一下其中用到的设计模式。下面将逐个列举其中的设计模式，并根据自己的理解分析这样设计的原因和优劣。

## 责任链模式

责任链模式在Dubbo中发挥的作用举足轻重，就像是Dubbo框架的骨架。Dubbo的调用链组织是用责任链模式串连起来的。责任链中的每个节点实现`Filter`接口，然后由`ProtocolFilterWrapper`，将所有`Filter`串连起来。Dubbo的许多功能都是通过`Filter`扩展实现的，比如监控、日志、缓存、安全、telnet以及RPC本身都是。如果把Dubbo比作一列火车，责任链就像是火车的各车厢，每个车厢的功能不同。如果需要加入新的功能，增加车厢就可以了，非常容易扩展。

## 观察者模式
Dubbo中使用观察者模式最典型的例子是`RegistryService`。消费者在初始化的时候回调用subscribe方法，注册一个观察者，如果观察者引用的服务地址列表发生改变，就会通过`NotifyListener`通知消费者。此外，Dubbo的`InvokerListener`、`ExporterListener` 也实现了观察者模式，只要实现该接口，并注册，就可以接收到consumer端调用refer和provider端调用export的通知。Dubbo的注册/订阅模型和观察者模式就是天生一对。

## 修饰器模式
Dubbo中还大量用到了修饰器模式。比如`ProtocolFilterWrapper`类是对Protocol类的修饰。在export和refer方法中，配合责任链模式，把Filter组装成责任链，实现对Protocol功能的修饰。其他还有`ProtocolListenerWrapper`、 `ListenerInvokerWrapper`、`InvokerWrapper`等。个人感觉，修饰器模式是一把双刃剑，一方面用它可以方便地扩展类的功能，而且对用户无感，但另一方面，过多地使用修饰器模式不利于理解，因为一个类可能经过层层修饰，最终的行为已经和原始行为偏离较大。

## 工厂方法模式
`CacheFactory`的实现采用的是工厂方法模式。`CacheFactory`接口定义getCache方法，然后定义一个`AbstractCacheFactory`抽象类实现`CacheFactory`，并将实际创建cache的createCache方法分离出来，并设置为抽象方法。这样具体cache的创建工作就留给具体的子类去完成。

## 抽象工厂模式
`ProxyFactory`及其子类是Dubbo中使用抽象工厂模式的典型例子。`ProxyFactory`提供两个方法，分别用来生产`Proxy`和`Invoker`（这两个方法签名看起来有些矛盾，因为getProxy方法需要传入一个Invoker对象，而getInvoker方法需要传入一个`Proxy`对象，看起来会形成循环依赖，但其实两个方式使用的场景不一样）。`AbstractProxyFactory`实现了`ProxyFactory`接口，作为具体实现类的抽象父类。然后定义了`JdkProxyFactory`和`JavassistProxyFactory`两个具体类，分别用来生产基于jdk代理机制和基于javassist代理机制的`Proxy`和`Invoker`。

## 适配器模式
为了让用户根据自己的需求选择日志组件，Dubbo自定义了自己的Logger接口，并为常见的日志组件（包括jcl, jdk, log4j, slf4j）提供相应的适配器。并且利用简单工厂模式提供一个`LoggerFactory`，客户可以创建抽象的Dubbo自定义`Logger`，而无需关心实际使用的日志组件类型。在LoggerFactory初始化时，客户通过设置系统变量的方式选择自己所用的日志组件，这样提供了很大的灵活性。


## 代理模式
Dubbo consumer使用`Proxy`类创建远程服务的本地代理，本地代理实现和远程服务一样的接口，并且屏蔽了网络通信的细节，使得用户在使用本地代理的时候，感觉和使用本地服务一样。
