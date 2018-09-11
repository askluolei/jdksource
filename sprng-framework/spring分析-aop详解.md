# spring-aop 分析

总所周知，aop 的核心是动态代理 ，但是 动态代理 是如何跟 ioc 容器结合，是什么时候生成代理对象，怎么生成代理对象，什么情况下才会有 aop ， advice ， advisor， pointcut 这些概念在代码里面的体现?
我们需要知道 spring 的 aop 在容器中的运行机制，而不是一句简单的动态代理

## 源码解析
核心类为 `AnnotationAwareAspectJAutoProxyCreator` ,为什么一上来就是这个类，可以看 springboot 里面 aop 的自动
配置类里面，就可以看到这个类了  
那么这个类是怎么起作用的，我们知道，aop 的核心就是动态代理，代理，肯定需要代理一个对象，  
我们也很清楚，被代理的是 spring 容器里面的 bean 实例，那么，需要使用代理对象替换容器里面的对象
需要使用 ioc 的扩展接口 `BeanPostProcessor` 及其子接口，我们往这个类的父类上看看，就可以
看到 `AbstractAutoProxyCreator` 这个类就是实现了 `SmartInstantiationAwareBeanPostProcessor` 接口

经过简单的测试和 debug 跟踪，发现，代理发生的方式是 `postProcessAfterInitialization` 这个方法是在
bean 实例化之后发生的 
```java
public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) throws BeansException {
	if (bean != null) {
		Object cacheKey = getCacheKey(bean.getClass(), beanName);
		if (!this.earlyProxyReferences.contains(cacheKey)) {
			return wrapIfNecessary(bean, beanName, cacheKey);
		}
	}
	return bean;
}
```	

重点在 `wrapIfNecessary` 方法，这个地方就是用来判断是否需要使用代理,贴出关键代码片段
```java
// Create proxy if we have advice.
// 这里就是找代理增强的动作
Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
if (specificInterceptors != DO_NOT_PROXY) {
	this.advisedBeans.put(cacheKey, Boolean.TRUE);
	// 创建代理
	Object proxy = createProxy(
			bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
	this.proxyTypes.put(cacheKey, proxy.getClass());
	return proxy;
}
```
可以看到，上面就是两个关键动作，找到需要增强的动作，然后创建代理  
我们先看看怎么找增强动作  
```java
protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {
	// 找出增强
	List<Advisor> candidateAdvisors = findCandidateAdvisors();
	// 是否可以用在本类型上
	List<Advisor> eligibleAdvisors = findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);
	// 这个留给子类玩
	extendAdvisors(eligibleAdvisors);
	if (!eligibleAdvisors.isEmpty()) {
	    // 排序
		eligibleAdvisors = sortAdvisors(eligibleAdvisors);
	}
	return eligibleAdvisors;
}

// 这个是在类 AnnotationAwareAspectJAutoProxyCreator 里面的实现
@Override
protected List<Advisor> findCandidateAdvisors() {
	// Add all the Spring advisors found according to superclass rules.
	List<Advisor> advisors = super.findCandidateAdvisors();
	// Build Advisors for all AspectJ aspects in the bean factory.
	if (this.aspectJAdvisorsBuilder != null) {
		advisors.addAll(this.aspectJAdvisorsBuilder.buildAspectJAdvisors());
	}
	return advisors;
}
```	

可以看到先调用 super 的，然后调用本类的，找 aspect 相关的  
实际上，这里处理的是两个类型的，一般增强的动作，概念上就是 Adivce ，而它下面可以分为两个分支  
一个是 aspect 的 before after around 这一批，另一个是 intercept 这一批  
第一批通常是我们自定义用的，第二批通常是框架用的，如果是框架用，通常直接就注册 advisor 了  
这里父类处理的就是 框架的，子类处理的，就是自定义的，我们详细看看  
先看父类处理
```java
if (advisorNames == null) {
	// Do not initialize FactoryBeans here: We need to leave all regular beans
	// uninitialized to let the auto-proxy creator apply to them!
	advisorNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
			this.beanFactory, Advisor.class, true, false);
	this.cachedAdvisorBeanNames = advisorNames;
}
// ...
advisors.add(this.beanFactory.getBean(name, Advisor.class));
```	
这里是在容器里面直接找 `Advisor` 类型的 bean，我们可以结合自己使用 aop 的方式看  
我们通常使用 aop 就是定义 aspect 使用 @Aspect 注解 ，定义 pointcut 和 advice  
基本确定，自己使用的 aop 肯定不是在这里处理的，那么我们再看看子类的处理(只有关键步骤) 
```java
String[] beanNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
		this.beanFactory, Object.class, true, false);
// ...
// We must be careful not to instantiate beans eagerly as in this case they
// would be cached by the Spring container but would not have been weaved.
Class<?> beanType = this.beanFactory.getType(beanName);
if (beanType == null) {
	continue;
}
if (this.advisorFactory.isAspect(beanType)) {
	aspectNames.add(beanName);
	AspectMetadata amd = new AspectMetadata(beanType, beanName);
	if (amd.getAjType().getPerClause().getKind() == PerClauseKind.SINGLETON) {
		MetadataAwareAspectInstanceFactory factory =
				new BeanFactoryAspectInstanceFactory(this.beanFactory, beanName);
		List<Advisor> classAdvisors = this.advisorFactory.getAdvisors(factory);
		if (this.beanFactory.isSingleton(beanName)) {
			this.advisorsCache.put(beanName, classAdvisors);
		}
		else {
			this.aspectFactoryCache.put(beanName, factory);
		}
		advisors.addAll(classAdvisors);
	}
	else {
		// Per target or per this.
		if (this.beanFactory.isSingleton(beanName)) {
			throw new IllegalArgumentException("Bean with name '" + beanName +
					"' is a singleton, but aspect instantiation model is not singleton");
		}
		MetadataAwareAspectInstanceFactory factory =
				new PrototypeAspectInstanceFactory(this.beanFactory, beanName);
		this.aspectFactoryCache.put(beanName, factory);
		advisors.addAll(this.advisorFactory.getAdvisors(factory));
	}
}
```	
一般走上面的判断，判断就是看是单例还是原型，重点逻辑是 
```java
MetadataAwareAspectInstanceFactory factory =
		new BeanFactoryAspectInstanceFactory(this.beanFactory, beanName);
List<Advisor> classAdvisors = this.advisorFactory.getAdvisors(factory);
```	

这里获取 Advisor ，但是之前说过了，我们自定使用的 aop 是没有实现 Advisor 的，我们使用的是注解 @Aspect @Pointcut @Before @Around 等
那么，里面肯定有一个转换过程，我们进去看看 getAdvisors 方法  
```java
List<Advisor> advisors = new LinkedList<>();
for (Method method : getAdvisorMethods(aspectClass)) {
	Advisor advisor = getAdvisor(method, lazySingletonAspectInstanceFactory, advisors.size(), aspectName);
	if (advisor != null) {
		advisors.add(advisor);
	}
}
	
// Find introduction fields.
for (Field field : aspectClass.getDeclaredFields()) {
	Advisor advisor = getDeclareParentsAdvisor(field);
	if (advisor != null) {
		advisors.add(advisor);
	}
}
```	

里面有两个重点，一个是处理方法，一个是处理字段  
自己可以进去看看，方法是没有 @Pointcut 注解的，都要处理  
处理的逻辑就是方法带 Before.class, Around.class, After.class, AfterReturning.class, AfterThrowing.class, Pointcut.class 注解的 
构造 InstantiationModelAwarePointcutAdvisorImpl 返回，我们清楚，这里就有了 Advice 增强动作，pointcut 哪里增强
事件上 beforeAdvice 也有哪里增强的逻辑（before），所以真正标识在哪里增强的，应该是 Advisor 实例，这个是 spring 内部帮忙处理的

在看看怎么处理字段的  
它处理的是带 @DeclareParents 注解的，返回 DeclareParentsAdvisor 实例，具体的，可以看看这个注解的用法，这里就不详看了  

到这里，找到 Advisor 哪里增强，就结束了，接下来就是创建代理的过程。  
```java
// 创建代理
Object proxy = createProxy(bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
```

传入的参数，class 类型，beanName，这里的 specificInterceptors 就是返回的 Advisor ，保证一下原始对象 
我们看看创建代理的详细过程  
```java
protected Object createProxy(Class<?> beanClass, @Nullable String beanName,
			@Nullable Object[] specificInterceptors, TargetSource targetSource) {

	if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
		AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
	}

	ProxyFactory proxyFactory = new ProxyFactory();
	proxyFactory.copyFrom(this);

	if (!proxyFactory.isProxyTargetClass()) {
		if (shouldProxyTargetClass(beanClass, beanName)) {
			proxyFactory.setProxyTargetClass(true);
		}
		else {
			evaluateProxyInterfaces(beanClass, proxyFactory);
		}
	}

	Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
	proxyFactory.addAdvisors(advisors);
	proxyFactory.setTargetSource(targetSource);
	customizeProxyFactory(proxyFactory);

	proxyFactory.setFrozen(this.freezeProxy);
	if (advisorsPreFiltered()) {
		proxyFactory.setPreFiltered(true);
	}

	return proxyFactory.getProxy(getProxyClassLoader());
}
```	

看上起很多，事件上逻辑很简单 
1. 判断使用 cglib 还是 java 的动态代理
2. buildAdvisors 其实就是返回之前的参数，里面会添加自定义的（commonInterceptors），但是通常是没有
3. 创建代理
重点在创建代理，我们已经知道了，只有两种情况，直接看就行了  
先看看 cglib 的，先要会 cglib 的基本 api 才行  
```java
// Configure CGLIB Enhancer...
Enhancer enhancer = createEnhancer();
if (classLoader != null) {
	enhancer.setClassLoader(classLoader);
	if (classLoader instanceof SmartClassLoader &&
			((SmartClassLoader) classLoader).isClassReloadable(proxySuperClass)) {
		enhancer.setUseCache(false);
	}
}
enhancer.setSuperclass(proxySuperClass);
enhancer.setInterfaces(AopProxyUtils.completeProxiedInterfaces(this.advised));
enhancer.setNamingPolicy(SpringNamingPolicy.INSTANCE);
enhancer.setStrategy(new ClassLoaderAwareUndeclaredThrowableStrategy(classLoader));

Callback[] callbacks = getCallbacks(rootClass);
Class<?>[] types = new Class<?>[callbacks.length];
for (int x = 0; x < types.length; x++) {
	types[x] = callbacks[x].getClass();
}
// fixedInterceptorMap only populated at this point, after getCallbacks call above
enhancer.setCallbackFilter(new ProxyCallbackFilter(
		this.advised.getConfigurationOnlyCopy(), this.fixedInterceptorMap, this.fixedInterceptorOffset));
enhancer.setCallbackTypes(types);

// Generate the proxy class and create a proxy instance.
return createProxyClassAndInstance(enhancer, callbacks);
```	

上面重点有两个，一个是设置 callback 一个是 setCallbackFilter 
callback 是一个数组 ProxyCallbackFilter 里面有一个方法，返回 int 类型，是 callback 数组的下标，对应的 callback 处理  
重点关注 aop 的 callback 就可以了 DynamicAdvisedInterceptor 使用的是这个 
里面重点就是一行，其他的可以自己看 
```java
retVal = new CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy).proceed();
```

里面的具体调用过程就是代理过程了  
```java
public Object proceed() throws Throwable {
//	We start with an index of -1 and increment early.
if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
	return invokeJoinpoint();
}

Object interceptorOrInterceptionAdvice =
		this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
	// Evaluate dynamic method matcher here: static part will already have
	// been evaluated and found to match.
	InterceptorAndDynamicMethodMatcher dm =
			(InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
	if (dm.methodMatcher.matches(this.method, this.targetClass, this.arguments)) {
		return dm.interceptor.invoke(this);
	}
	else {
		// Dynamic matching failed.
		// Skip this interceptor and invoke the next in the chain.
		return proceed();
	}
}
else {
	// It's an interceptor, so we just invoke it: The pointcut will have
	// been evaluated statically before this object was constructed.
	return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
}
}
```
这个调用模式可以学一学
到这里，cglib 的实现就看完了，其实大部分时候，就是用的 cglib 动态代理 
jdk 的动态代理，就是实现 `InvocationHandler` 然后看看 invoke 方法 
```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
		MethodInvocation invocation;
	Object oldProxy = null;
	boolean setProxyContext = false;

	TargetSource targetSource = this.advised.targetSource;
	Object target = null;

	try {
		if (!this.equalsDefined && AopUtils.isEqualsMethod(method)) {
			// The target does not implement the equals(Object) method itself.
			return equals(args[0]);
		}
		else if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {
			// The target does not implement the hashCode() method itself.
			return hashCode();
		}
		else if (method.getDeclaringClass() == DecoratingProxy.class) {
			// There is only getDecoratedClass() declared -> dispatch to proxy config.
			return AopProxyUtils.ultimateTargetClass(this.advised);
		}
		else if (!this.advised.opaque && method.getDeclaringClass().isInterface() &&
				method.getDeclaringClass().isAssignableFrom(Advised.class)) {
			// Service invocations on ProxyConfig with the proxy config...
			return AopUtils.invokeJoinpointUsingReflection(this.advised, method, args);
		}

		Object retVal;

		if (this.advised.exposeProxy) {
			// Make invocation available if necessary.
			oldProxy = AopContext.setCurrentProxy(proxy);
			setProxyContext = true;
		}

		// Get as late as possible to minimize the time we "own" the target,
		// in case it comes from a pool.
		target = targetSource.getTarget();
		Class<?> targetClass = (target != null ? target.getClass() : null);

		// Get the interception chain for this method.
		List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

		// Check whether we have any advice. If we don't, we can fallback on direct
		// reflective invocation of the target, and avoid creating a MethodInvocation.
		if (chain.isEmpty()) {
			// We can skip creating a MethodInvocation: just invoke the target directly
			// Note that the final invoker must be an InvokerInterceptor so we know it does
			// nothing but a reflective operation on the target, and no hot swapping or fancy proxying.
			Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
			retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
		}
		else {
			// We need to create a method invocation...
			invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
			// Proceed to the joinpoint through the interceptor chain.
			retVal = invocation.proceed();
		}

		// Massage return value if necessary.
		Class<?> returnType = method.getReturnType();
		if (retVal != null && retVal == target &&
				returnType != Object.class && returnType.isInstance(proxy) &&
				!RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
			// Special case: it returned "this" and the return type of the method
			// is type-compatible. Note that we can't help if the target sets
			// a reference to itself in another returned object.
			retVal = proxy;
		}
		else if (retVal == null && returnType != Void.TYPE && returnType.isPrimitive()) {
			throw new AopInvocationException(
					"Null return value from advice does not match primitive return type for: " + method);
		}
		return retVal;
	}
	finally {
		if (target != null && !targetSource.isStatic()) {
			// Must have come from TargetSource.
			targetSource.releaseTarget(target);
		}
		if (setProxyContext) {
			// Restore old proxy.
			AopContext.setCurrentProxy(oldProxy);
		}
	}
}
```
 一大坨代码，自己看了  
 到此，aop 流程就结束了 
 
