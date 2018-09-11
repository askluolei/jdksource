# spring-tx 分析
什么是事务？

## 详细分析 
稍微想一下，大概就可以猜到，事务是通过 aop 实现的，在前后增强，类似 @Around  
但是，在 aop 里面就说过了，我们自定义的 aop 才使用 @Around ，框架内部使用的是直接的 Advisor 
tx 的配置类是 `ProxyTransactionManagementConfiguration` 里面注册了3个 bean  
`BeanFactoryTransactionAttributeSourceAdvisor`,  这个后面要详细说，aop 找的就是这个类，这个类使用了下面两个  
`TransactionAttributeSource`, 这个是用来判断是否有事务的  
`TransactionInterceptor` 具体做事的地方，Advice 的另一个分支 intercept 的实现，也使用了 `TransactionAttributeSource`  

我们先看看 `TransactionAttributeSource` 是干啥的  
```java
public interface TransactionAttributeSource {

	@Nullable
	TransactionAttribute getTransactionAttribute(Method method, @Nullable Class<?> targetClass);

}
```
看一眼就知道，从方法和类型里面读取事务信息的，估计如果没有读取到，那就不走事务，没有事务 aop  
使用的具体实现是  `AnnotationTransactionAttributeSource` 可以进去看看，分为3中类型，ejb，jta，springtx，现在只关注最有一个  
spring 的使用 `@Transactional` 注解 

然后看看 `TransactionInterceptor` 这里是具体做事的地方，细节很复杂，详细看看，对以后配置事务，数据源有帮助  
实际类型是 `TransactionInterceptor`  实现了 `MethodInterceptor` 接口，这个接口是 `Advice` 的扩展接口  
里面，invoke 方法直接调用父类的 `invokeWithinTransaction` 方法  
```java
if (txAttr == null || !(tm instanceof CallbackPreferringPlatformTransactionManager)) {
	// Standard transaction demarcation with getTransaction and commit/rollback calls.
	TransactionInfo txInfo = createTransactionIfNecessary(tm, txAttr, joinpointIdentification);
	Object retVal = null;
	try {
		// This is an around advice: Invoke the next interceptor in the chain.
		// This will normally result in a target object being invoked.
		retVal = invocation.proceedWithInvocation();
	}
	catch (Throwable ex) {
		// target invocation exception
		completeTransactionAfterThrowing(txInfo, ex);
		throw ex;
	}
	finally {
		cleanupTransactionInfo(txInfo);
	}
	commitTransactionAfterReturning(txInfo);
	return retVal;
}
```	
虽然有 else 的逻辑，但是，大部分走的是这里，逻辑很清晰，这里就不多讲了，直接看细节。  
首先是创建事务，这里可能创建新事务，具体需要看配置（spring 的事务传播级别）
```java
protected TransactionInfo createTransactionIfNecessary(@Nullable PlatformTransactionManager tm,
		@Nullable TransactionAttribute txAttr, final String joinpointIdentification) {

	// If no name specified, apply method identification as transaction name.
	if (txAttr != null && txAttr.getName() == null) {
		txAttr = new DelegatingTransactionAttribute(txAttr) {
			@Override
			public String getName() {
				return joinpointIdentification;
			}
		};
	}

	TransactionStatus status = null;
	if (txAttr != null) {
		if (tm != null) {
			status = tm.getTransaction(txAttr);
		}
		else {
			if (logger.isDebugEnabled()) {
				logger.debug("Skipping transactional joinpoint [" + joinpointIdentification +
						"] because no transaction manager has been configured");
			}
		}
	}
	return prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);
}
```	

其实，这里只是调用了 `PlatformTransactionManager` 的 getTransaction 方法，在这里面处理了传播级别，我们过会看里面的细节， 
还是看事务的拦截器那里，获取事务，方法调用，异常处理，事务完成  
我们接下来看看异常处理  
```java
protected void completeTransactionAfterThrowing(@Nullable TransactionInfo txInfo, Throwable ex) {
	if (txInfo != null && txInfo.getTransactionStatus() != null) {
		if (logger.isTraceEnabled()) {
			logger.trace("Completing transaction for [" + txInfo.getJoinpointIdentification() +
					"] after exception: " + ex);
		}
		if (txInfo.transactionAttribute != null && txInfo.transactionAttribute.rollbackOn(ex)) {
			try {
				txInfo.getTransactionManager().rollback(txInfo.getTransactionStatus());
			}
			catch (TransactionSystemException ex2) {
				logger.error("Application exception overridden by rollback exception", ex);
				ex2.initApplicationException(ex);
				throw ex2;
			}
			catch (RuntimeException | Error ex2) {
				logger.error("Application exception overridden by rollback exception", ex);
				throw ex2;
			}
		}
		else {
			// We don't roll back on this exception.
			// Will still roll back if TransactionStatus.isRollbackOnly() is true.
			try {
				txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
			}
			catch (TransactionSystemException ex2) {
				logger.error("Application exception overridden by commit exception", ex);
				ex2.initApplicationException(ex);
				throw ex2;
			}
			catch (RuntimeException | Error ex2) {
				logger.error("Application exception overridden by commit exception", ex);
				throw ex2;
			}
		}
	}
}
```
一大坨代码，实际上做了两件事，如果抛出的异常是需要回滚的，那就回滚，否则提交，还是使用 `PlatformTransactionManager` 的方法  
提交事务一样 
```java
protected void commitTransactionAfterReturning(@Nullable TransactionInfo txInfo) {
	if (txInfo != null && txInfo.getTransactionStatus() != null) {
		if (logger.isTraceEnabled()) {
			logger.trace("Completing transaction for [" + txInfo.getJoinpointIdentification() + "]");
		}
		txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
	}
}
```

那么，我们就可以把注意力转移到 `PlatformTransactionManager` 的实现了，我们先看看这个接口，也就3个方法  
```java
public interface PlatformTransactionManager {

	TransactionStatus getTransaction(@Nullable TransactionDefinition definition) throws TransactionException;

	void commit(TransactionStatus status) throws TransactionException;

	void rollback(TransactionStatus status) throws TransactionException;

}
```	

外部也只调用了这3个方法  
下面的继承关系也很简单，只有一个抽象类 `AbstractPlatformTransactionManager` ，然后下面的都是具体实现了  
先看  `getTransaction` 方法,先简单说一下逻辑，代码贴的太多了，
首先调用 `doGetTransaction` 方法获取事务对象，事务对象的具体类型是子类确定的  
判断事务是否已经存在（方法外部已经有事务了） `isExistingTransaction` 这个子类实现，
如果存在了，那就调用 `handleExistingTransaction` 方法返回  
这里就需要判断事务传播级别了，这里先说说传播级别
1. PROPAGATION_REQUIRED  支持当前事务，如果没有，就新建事务，这个是默认的
2. PROPAGATION_SUPPORTS  支持当前事务，如果没有，那本方法不走业务
3. PROPAGATION_MANDATORY  支持当前事务，如果没有，就抛异常
4. PROPAGATION_REQUIRES_NEW  本方法使用新事务，如果当前存在事务了，就暂停当前的事务
5. PROPAGATION_NOT_SUPPORTED  不支持当前事务，总是以非事务运行
6. PROPAGATION_NEVER  不支持当前事务，如果存在，抛异常
7. PROPAGATION_NESTED 如果存在当前事务，本方法就以内部事务运行

清楚传播级别后，代码可以自己细看了  
这里说说新建事务相关的方法 
1. newTransactionStatus 新建事务，这方方法的返回值，就是 getTransaction 的返回值
2. doBegin 这里才是开始事务，如果没有执行这个方法，实际上是使用当前事务，调用了这个，就是新建事务
3. prepareSynchronization 这个方法应该是在 doBegin 后面执行的，设置一些线程上下文
4. prepareTransactionStatus 这个方法包含 newTransactionStatus 和 prepareSynchronization

如果是使用当前事务，那么新建的事务对象里面 `newTransaction` 属性是 false 
再说 commit 方法，只有当当前的事务对象 `newTransaction` 是 true 的时候，才调用 `doCommit` 方法做真正的提交  
rollback 方法，同样，只有当当前的事务对象 `newTransaction` 是 true 的时候，才调用 `doRollback` 方法做回滚  
除了上面的关键逻辑，还有一些其他的，譬如事务释放只回滚（test的时候用的）,触发一些事件  

那么子类需要实现的方法就有如下方法了  
1. doGetTransaction 获取事务对象，返回的对象是自己定义的类型
2. isExistingTransaction  当前事务是否存在，参数就是事务对象，自己定义的对象要能够分辨出来事务是否存在
3. doBegin 开始事务，通常，调用了这个方法，事务才被激活，才能说事务存在
4. doSuspend 暂停当前事务，返回值会传递给恢复方法 doResume
5. doResume 恢复事务
6. doCommit 实际的事务提交
7. doRollback 实际的事务回滚
8. doCleanupAfterCompletion 事务结束后的清尾操作

上面是必须的，其他的方法，可以覆盖父类，或者自定义方法  

需要看看 `DataSourceTransactionManager` 管理 jdbc 事务  
所谓事务，就是当前线程中，执行的方法，要使用同一个数据库连接对象 `Connection`  
很自然的想要，要使用线程对象来存储，spring 也是这样做的，在 doBegin 的时候，才需要获取数据库连接，存到线程对象中  
忽略其他细节
```java
TransactionSynchronizationManager.bindResource(obtainDataSource(), txObject.getConnectionHolder());
```
详细的可以看看 `TransactionSynchronizationManager` 这个类，里面有一堆线程对象  
顺便说一句，我们可以看到 `bindResource` 方法第一个对象是key 第二个对象是value，key 是数据库连接池对象  
我们可以使用 aop 在运行过程中确定使用哪个数据库连接池(数据库),再设置一个默认的数据库连接池，就可以使用多数据库了，不过不支持分布式事务，但是可以做读写分离  
具体的可以使用 `AbstractRoutingDataSource`  
事务和连接池的就分析到这里了
