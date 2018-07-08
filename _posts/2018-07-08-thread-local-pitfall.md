---
layout:     post
title:      "慎用线程局部变量"
date:       2018-07-08 12:00:00
author:     "Thh"
tags:
    - 问题解决
---

最近项目中碰到一个bug，bug出现的原因跟线程局部变量有关，比较典型，这里记录一下。

## Bug场景

SpringBoot web应用，使用通用mapper [https://github.com/abel533/Mapper](https://github.com/abel533/Mapper "通用mapper") 以及PageHelper [https://github.com/pagehelper/Mybatis-PageHelper](https://github.com/pagehelper/Mybatis-PageHelper) (mapper-spring-boot-starter版本2.0.2，pagehelper-spring-boot-starter版本1.2.4)做DAL层，测试用例中使用h2内存数据库。

问题出现在单元测试用例中。项目有多个单元测试类，其中一个叫TaskManagerImplTest，用来做集成测试。报错的就是这个单元测试类。

	Caused by: org.h2.jdbc.JdbcSQLException: Feature not supported: "MVCC=TRUE && FOR UPDATE && GROUP"; SQL statement:
	SELECT count(0) FROM general_property WHERE (name = ?) FOR UPDATE [50100-197]
		at org.h2.message.DbException.getJdbcSQLException(DbException.java:357)
		at org.h2.message.DbException.get(DbException.java:179)
		at org.h2.message.DbException.get(DbException.java:155)
		at org.h2.message.DbException.getUnsupportedException(DbException.java:228)
		at org.h2.command.dml.Select.queryWithoutCache(Select.java:603)
		at org.h2.command.dml.Query.queryWithoutCacheLazyCheck(Query.java:114)
		at org.h2.command.dml.Query.query(Query.java:371)
		at org.h2.command.dml.Query.query(Query.java:333)
		at org.h2.command.CommandContainer.query(CommandContainer.java:114)
		at org.h2.command.Command.executeQuery(Command.java:202)
		at org.h2.jdbc.JdbcPreparedStatement.execute(JdbcPreparedStatement.java:242)
		at com.alibaba.druid.pool.DruidPooledPreparedStatement.execute(DruidPooledPreparedStatement.java:498)
		at sun.reflect.GeneratedMethodAccessor31.invoke(Unknown Source)
		at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
		at java.lang.reflect.Method.invoke(Method.java:498)
		at org.apache.ibatis.logging.jdbc.PreparedStatementLogger.invoke(PreparedStatementLogger.java:59)
		at com.sun.proxy.$Proxy62.execute(Unknown Source)
		at org.apache.ibatis.executor.statement.PreparedStatementHandler.query(PreparedStatementHandler.java:63)
		at org.apache.ibatis.executor.statement.RoutingStatementHandler.query(RoutingStatementHandler.java:79)
		at org.apache.ibatis.executor.SimpleExecutor.doQuery(SimpleExecutor.java:63)
		at org.apache.ibatis.executor.BaseExecutor.queryFromDatabase(BaseExecutor.java:326)
		at org.apache.ibatis.executor.BaseExecutor.query(BaseExecutor.java:156)
		at org.apache.ibatis.executor.CachingExecutor.query(CachingExecutor.java:109)
		at com.github.pagehelper.PageInterceptor.executeAutoCount(PageInterceptor.java:201)
		at com.github.pagehelper.PageInterceptor.intercept(PageInterceptor.java:113)
		at org.apache.ibatis.plugin.Plugin.invoke(Plugin.java:61)
		at com.sun.proxy.$Proxy136.query(Unknown Source)
		at org.apache.ibatis.session.defaults.DefaultSqlSession.selectList(DefaultSqlSession.java:148)
		at org.apache.ibatis.session.defaults.DefaultSqlSession.selectList(DefaultSqlSession.java:141)
		at org.apache.ibatis.session.defaults.DefaultSqlSession.selectOne(DefaultSqlSession.java:77)

奇怪的是当所有测用例一起执行时，TaskManagerImplTest会出现上面的异常，但单独执行 TaskManagerImplTest又没有问题。

## 解决思路及过程

第一反应可能跟h2的配置有关，查资料发现h2有个配置是控制是否启用MVCC，尝试关闭该配置

	jdbc:h2:mem:manualtask;MODE=MYSQL;MVCC=false

无效，异常依然出现。

接着尝试在网上查该问题，没有发现靠谱的解决方案。于是决定自己分析，跟踪代码，查原因。

由于该测试用例单独执行没有问题，所有测试用例一起执行有问题，说明应该是其他测试用例对该测试用例产生了影响。通过二分法排除，很快找到引起问题的测试用例，TaskMapperTest类，并通过同样的方法定位到引起问题方法：

	@Test
	public void  testSelectByExample() {
	    TaskEntity taskModel = new TaskEntity();
	    String id = TestUtils.generateUUID();
	    String procInstId = TestUtils.generateUUID();
	    taskModel.setId(id);
	    taskModel.setBusinessKey(TestUtils.generateUUID());
	    taskModel.setProcessInstanceId(procInstId);
	    taskMapper.insert(taskModel);
	    PageHelper.startPage(0, 1);
	    Example example = Example.builder(TaskEntity.class).select("source").where(Sqls.custom().andEqualTo("processInstanceId", procInstId)).build();
	    List<TaskEntity> taskEntities = taskMapper.selectByExample(example);
	    assertTrue(taskEntities.size() > 0);
	}

这个方法并不复杂，只是做常规的insert和select测试。其中有一句PageHelper.startPage(0, 1)设置分页参数，把这一句去掉，测试用例就可以正常运行。

到这里问题已经解决，但解决完问题，事情还没完。为什么加了分页，会影响后面其他的测试用例执行呢? 根据PageHelper官方文档，PageHelper.startPage设置的的线程局部变量，会被后面的第一条查询消费掉，为啥这里没有消费呢？于是接下来决定跟踪代码，一探究竟。

查看抛出异常的源码（`org.h2.command.dml.Select`类的queryWithoutCache方法）：

	if (isForUpdateMvcc) {
	    if (isGroupQuery) {
	        throw DbException.getUnsupportedException(
	                "MVCC=TRUE && FOR UPDATE && GROUP");
	    } else if (distinct) {
	        throw DbException.getUnsupportedException(
	                "MVCC=TRUE && FOR UPDATE && DISTINCT");
	    } else if (isQuickAggregateQuery) {
	        throw DbException.getUnsupportedException(
	                "MVCC=TRUE && FOR UPDATE && AGGREGATE");
	    } else if (topTableFilter.getJoin() != null) {
	        throw DbException.getUnsupportedException(
	                "MVCC=TRUE && FOR UPDATE && JOIN");
	    }
	}

当isForUpdateMvcc和isGroupQuery同时为true时，就抛出上面的异常，那现在的问题就变成这2个变量是在哪设置为true的呢？

继续跟，找到了设置isForUpdateMvcc的地方：

	this.isForUpdate = b;
	    if (session.getDatabase().getSettings().selectForUpdateMvcc &&
	            session.getDatabase().isMultiVersion()) {
	        isForUpdateMvcc = b;
	    }
	}

这段代码大意就是如果当前查询有for update，同时数据库支持mvcc，而且配置了select for update使用mvcc的话，就设置isForUpdateMvcc为true。
而isGroupQuery会在多个地方设置，这个场景下，因为有select count，所以为true。

出问题的代码生成的是一句`select name, value from general_property where name=? for update`语句，但异常信息中报出来有问题的sql是: `SELECT count(0) FROM general_property WHERE (name = ?) FOR UPDATE`

代码中并没有这样的count语句，那么这条sql是哪来的呢？

继续分析，发现该sql来自于PageInterceptor类的executeAutoCount方法。

	@Override
	public Object intercept(Invocation invocation) throws Throwable {
	    try {
	         …       
			//调用方法判断是否需要进行分页，如果不需要，直接返回结果
	        if (!dialect.skip(ms, parameter, rowBounds)) {
	            …
				if (dialect.beforeCount(ms, parameter, rowBounds)) {
			    String countMsId = msId + countSuffix;
			    Long count;
			    //先判断是否存在手写的 count 查询
			    MappedStatement countMs = getExistedMappedStatement(configuration, countMsId);
			    if(countMs != null){
			        count = executeManualCount(executor, countMs, parameter, boundSql, resultHandler);
			    } else {
			        countMs = msCountMap.get(countMsId);
			        //自动创建
			        if (countMs == null) {
			            //根据当前的 ms 创建一个返回值为 Long 类型的 ms
			            countMs = MSUtils.newCountMappedStatement(ms, countMsId);
			            msCountMap.put(countMsId, countMs);
			        }
			        count = executeAutoCount(executor, countMs, parameter, boundSql, rowBounds, resultHandler);
			    }
			    //处理查询总数
			    //返回 true 时继续分页查询，false 时直接返回
			    if (!dialect.afterCount(count, parameter, rowBounds)) {
			        //当查询总数为 0 时，直接返回空的结果
			        return dialect.afterPage(new ArrayList(), parameter, rowBounds);
			    }
				}
				…
	
		    } else {
		            //rowBounds用参数值，不使用分页插件处理时，仍然支持默认的内存分页
		            resultList = executor.query(ms, parameter, rowBounds, resultHandler, cacheKey, boundSql);
		    }
	        return dialect.afterPage(resultList, parameter, rowBounds);
	    } finally {
	        dialect.afterAll();
	    }
	}

分析代码，发现该查询触发了分页，也就是

	if (!dialect.skip(ms, parameter, rowBounds)) {

这个判断为true，没有跳过分页。

继续跟进，发现该方法调用了PageHelper的skip方法。

	@Override
	public boolean skip(MappedStatement ms, Object parameterObject, RowBounds rowBounds) {
	    if(ms.getId().endsWith(MSUtils.COUNT)){
	        throw new RuntimeException("在系统中发现了多个分页插件，请检查系统配置!");
	    }
	    Page page = pageParams.getPage(parameterObject, rowBounds);
	    if (page == null) {
	        return true;
	    } else {
	        //设置默认的 count 列
	        if(StringUtil.isEmpty(page.getCountColumn())){
	            page.setCountColumn(pageParams.getCountColumn());
	        }
	        autoDialect.initDelegateDialect(ms);
	        return false;
	    }
	}

其中Page page = pageParams.getPage(parameterObject, rowBounds); 返回的page不为null，导致触发分页。

进入Pageparms类的getPage方法，发现

	public Page getPage(Object parameterObject, RowBounds rowBounds) {
    Page page = PageHelper.getLocalPage();
    if (page == null) {
        if (rowBounds != RowBounds.DEFAULT) {
	…

其中Page page = PageHelper.getLocalPage(); 返回的page不为null，而getLocalPage()方法是从线程局部变量中获取的。 而调用`PageHelper.startPage(int, int)`方法就会设置该线程局部变量。


	@Test
	public void  testSelectByExample() {
	    TaskEntity taskModel = new TaskEntity();
	    String id = TestUtils.generateUUID();
	    String procInstId = TestUtils.generateUUID();
	    taskModel.setId(id);
	    taskModel.setBusinessKey(TestUtils.generateUUID());
	    taskModel.setProcessInstanceId(procInstId);
	    taskMapper.insert(taskModel);
	    PageHelper.startPage(0, 1);
	    Example example = Example.builder(TaskEntity.class).select("source").where(Sqls.custom().andEqualTo("processInstanceId", procInstId)).build();
	    List<TaskEntity> taskEntities = taskMapper.selectByExample(example);
	    assertTrue(taskEntities.size() > 0);
	}

到这里，也就找到了为什么加了分页，会影响后面其他的测试用例的原因。前面提出的第二个问题还没有找到原因，根据PageHelper官方文档，PageHelper.startPage设置的的线程局部变量，会被后面的第一条查询消费掉，为啥这里没有消费/清除呢？那首先就要看看这个线程局部变量是如何被消费或清除的。

继续跟踪代码，发现这个线程局部变量是在com.github.pagehelper.PageInterceptor的intercept方法中清除的

	@Override
	public Object intercept(Invocation invocation) throws Throwable {
	    try {
	        …
	    } finally {
	        dialect.afterAll();
	    }
	}

	@Override
	public void afterAll() {
	    //这个方法即使不分页也会被执行，所以要判断 null
	    AbstractHelperDialect delegate = autoDialect.getDelegate();
	    if (delegate != null) {
	        delegate.afterAll();
	        autoDialect.clearDelegate();
	    }
	    clearPage();
	}
	
	/**
	 * 移除本地变量
	 */
	public static void clearPage() {
    	LOCAL_PAGE.remove();
	}

该线程局部变量是在finally块中执行，应该不会出现不执行的情况。所以唯一的可能是整个intercept方法没执行。

最后，根本原因终于浮出水面。原来，TaskMapperTest配置的spring context只初始化了mapper和数据源，并没有配置PageHelper插件。因此，TaskMapperTest执行的时候，并没有执行PageInterceptor的intercept方法；而TaskManagerImplTest测试用例的spring context配置了整个application，启用了PageInterceptor。这样就导致TaskMapperTest中设置的线程局部变量没有被清理掉，TaskManagerImplTest在执行的时候正好取到了，导致本来不需要分页的请求执行了分页，才有了select count那句sql。这样也就解释了TaskManagerImplTest单独执行没有问题，所有测试用例一起执行就有问题的原因。

## 总结

这个问题一开始看似跟H2有关，但最后的真相跟H2没有半毛钱关系。该问题的直接原因是PageHelper的使用方法有问题，但从框架的设计角度来说，个人认为通过PageHelper静态方法设置线程局部变量，然后在`PageInterceptor`拦截器中使用并不是一种优雅的设计方式，因为这样的用法并不安全。PageHelper的作者本人也承认这一点，但还是提供了这样的用法。

线程局部变量在某些情况下可以解决上下文传递的麻烦，但使用需要谨慎，尽量避免出现这种在一个地方设置，然后在另一个类的某个地方去清理，而应该尽量采用在一个方法块中完成变量的设置和清理，就像这样

	try {
		…
		threadLocal.set(value)
		…
	} finally {
		threadLocal.remove();
	}


