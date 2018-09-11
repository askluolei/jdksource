[TOC]   

# Spring 分析-AOP   

## AnnotationAwareAspectJAutoProxyCreator   
AOP 的注解核心配置类是 `AnnotationAwareAspectJAutoProxyCreator` 
现在不考虑 xml 的配置方式   

一直向上，找继承链到 `AbstractAutoProxyCreator`, 这个类实现 `SmartInstantiationAwareBeanPostProcessor` 接口 
而 `SmartInstantiationAwareBeanPostProcessor` 接口是 `BeanPostProcessor` 的子接口，作用是在 bean 实例化过程中做些扩展具体的在实例化里面可以看   
重点方法是 `postProcessAfterInitialization` 
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

protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
    if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
        return bean;
    }
    if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
        return bean;
    }
    if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
        this.advisedBeans.put(cacheKey, Boolean.FALSE);
        return bean;
    }

    // Create proxy if we have advice.
    Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
    if (specificInterceptors != DO_NOT_PROXY) {
        this.advisedBeans.put(cacheKey, Boolean.TRUE);
        Object proxy = createProxy(
                bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
        this.proxyTypes.put(cacheKey, proxy.getClass());
        return proxy;
    }

    this.advisedBeans.put(cacheKey, Boolean.FALSE);
    return bean;
}
``` 

第一步就是找到拦截器，也就是通知器，然后就是创建代理    

### 获取拦截 Advisor
```java
@Override
@Nullable
protected Object[] getAdvicesAndAdvisorsForBean(
        Class<?> beanClass, String beanName, @Nullable TargetSource targetSource) {

    List<Advisor> advisors = findEligibleAdvisors(beanClass, beanName);
    if (advisors.isEmpty()) {
        return DO_NOT_PROXY;
    }
    return advisors.toArray();
}

protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {
    // 找到所有的 Advisor，这里就是直接在容器里面找 Advisor 类型
    List<Advisor> candidateAdvisors = findCandidateAdvisors();
    // 过滤一下，找到匹配的 Advisor
    List<Advisor> eligibleAdvisors = findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);
    // 给子类扩展扩展一下匹配的集合
    extendAdvisors(eligibleAdvisors);
    if (!eligibleAdvisors.isEmpty()) {
        eligibleAdvisors = sortAdvisors(eligibleAdvisors);
    }
    return eligibleAdvisors;
}
``` 

先看看  `AnnotationAwareAspectJAutoProxyCreator.findCandidateAdvisors` 方法 
```java
@Override
protected List<Advisor> findCandidateAdvisors() {
    // Add all the Spring advisors found according to superclass rules.
    // 这里直接在容器里面找 Advisor 的实现
    List<Advisor> advisors = super.findCandidateAdvisors();
    // Build Advisors for all AspectJ aspects in the bean factory.
    // 不出意外，上面是空（spring boot 注解情况下）
    if (this.aspectJAdvisorsBuilder != null) {
        // 这里是看 @Aspect 注解的 bean
        advisors.addAll(this.aspectJAdvisorsBuilder.buildAspectJAdvisors());
    }
    return advisors;
}

public List<Advisor> buildAspectJAdvisors() {
    List<String> aspectNames = this.aspectBeanNames;

    if (aspectNames == null) {
        synchronized (this) {
            aspectNames = this.aspectBeanNames;
            if (aspectNames == null) {
                List<Advisor> advisors = new LinkedList<>();
                aspectNames = new LinkedList<>();
                // 遍历所有 beanNames
                String[] beanNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
                        this.beanFactory, Object.class, true, false);
                for (String beanName : beanNames) {
                    if (!isEligibleBean(beanName)) {
                        continue;
                    }
                    // We must be careful not to instantiate beans eagerly as in this case they
                    // would be cached by the Spring container but would not have been weaved.
                    Class<?> beanType = this.beanFactory.getType(beanName);
                    if (beanType == null) {
                        continue;
                    }
                    // 看是否有 @Aspect 注解，或者有字段是 aspect 类型,通常是 @Aspect，第二个条件无视
                    if (this.advisorFactory.isAspect(beanType)) {
                        aspectNames.add(beanName);
                        AspectMetadata amd = new AspectMetadata(beanType, beanName);
                        if (amd.getAjType().getPerClause().getKind() == PerClauseKind.SINGLETON) {
                            // 一般是这里
                            MetadataAwareAspectInstanceFactory factory =
                                    new BeanFactoryAspectInstanceFactory(this.beanFactory, beanName);
                            // 找到 advisor。。。，这里，看怎么从 Aspect 里面获取 Advisor
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
                }
                this.aspectBeanNames = aspectNames;
                return advisors;
            }
        }
    }

    if (aspectNames.isEmpty()) {
        return Collections.emptyList();
    }
    List<Advisor> advisors = new LinkedList<>();
    for (String aspectName : aspectNames) {
        List<Advisor> cachedAdvisors = this.advisorsCache.get(aspectName);
        if (cachedAdvisors != null) {
            advisors.addAll(cachedAdvisors);
        }
        else {
            MetadataAwareAspectInstanceFactory factory = this.aspectFactoryCache.get(aspectName);
            advisors.addAll(this.advisorFactory.getAdvisors(factory));
        }
    }
    return advisors;
}

@Override
public List<Advisor> getAdvisors(MetadataAwareAspectInstanceFactory aspectInstanceFactory) {
    Class<?> aspectClass = aspectInstanceFactory.getAspectMetadata().getAspectClass();
    String aspectName = aspectInstanceFactory.getAspectMetadata().getAspectName();
    validate(aspectClass);

    // We need to wrap the MetadataAwareAspectInstanceFactory with a decorator
    // so that it will only instantiate once.
    MetadataAwareAspectInstanceFactory lazySingletonAspectInstanceFactory =
            new LazySingletonAspectInstanceFactoryDecorator(aspectInstanceFactory);

    List<Advisor> advisors = new LinkedList<>();
    // 这里的方法 getAdvisorMethods 是找除了 @Pointcut 注解标记外的所有方法
    for (Method method : getAdvisorMethods(aspectClass)) {
        // 只要不是切入点，其他方法都可能是 Advice ，使用 这个method 构建 通知器 Advisor
        Advisor advisor = getAdvisor(method, lazySingletonAspectInstanceFactory, advisors.size(), aspectName);
        if (advisor != null) {
            advisors.add(advisor);
        }
    }

    // If it's a per target aspect, emit the dummy instantiating aspect.
    if (!advisors.isEmpty() && lazySingletonAspectInstanceFactory.getAspectMetadata().isLazilyInstantiated()) {
        Advisor instantiationAdvisor = new SyntheticInstantiationAdvisor(lazySingletonAspectInstanceFactory);
        advisors.add(0, instantiationAdvisor);
    }

    // Find introduction fields.
    // 字段有 @DeclareParents 注解
    for (Field field : aspectClass.getDeclaredFields()) {
        Advisor advisor = getDeclareParentsAdvisor(field);
        if (advisor != null) {
            advisors.add(advisor);
        }
    }

    return advisors;
}


@Override
@Nullable
public Advisor getAdvisor(Method candidateAdviceMethod, MetadataAwareAspectInstanceFactory aspectInstanceFactory,
        int declarationOrderInAspect, String aspectName) {

    validate(aspectInstanceFactory.getAspectMetadata().getAspectClass());

    AspectJExpressionPointcut expressionPointcut = getPointcut(
            candidateAdviceMethod, aspectInstanceFactory.getAspectMetadata().getAspectClass());
    if (expressionPointcut == null) {
        return null;
    }
    // 都是 InstantiationModelAwarePointcutAdvisorImpl 实例了
    return new InstantiationModelAwarePointcutAdvisorImpl(expressionPointcut, candidateAdviceMethod,
            this, aspectInstanceFactory, declarationOrderInAspect, aspectName);
}
``` 

### 创建代理   
创建代理的流程就是动态代理的过程，也就是网上 spring aop 原理讲的最多的 jdk 和 cglig 动态代理    

```java
protected Object createProxy(Class<?> beanClass, @Nullable String beanName,
        @Nullable Object[] specificInterceptors, TargetSource targetSource) {

    if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
        AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
    }

    // 代理工厂，正常的创建代理的流程
    ProxyFactory proxyFactory = new ProxyFactory();
    proxyFactory.copyFrom(this);

    // isProxyTargetClass 就是用来判断使用 cglib 还是 jdk 的动态代理
    if (!proxyFactory.isProxyTargetClass()) {
        if (shouldProxyTargetClass(beanClass, beanName)) {
            proxyFactory.setProxyTargetClass(true);
        }
        else {
            evaluateProxyInterfaces(beanClass, proxyFactory);
        }
    }

    // 获取 Advisor 通知器数组
    Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
    // 设置
    proxyFactory.addAdvisors(advisors);
    proxyFactory.setTargetSource(targetSource);
    customizeProxyFactory(proxyFactory);

    proxyFactory.setFrozen(this.freezeProxy);
    if (advisorsPreFiltered()) {
        proxyFactory.setPreFiltered(true);
    }
    // 创建代理
    return proxyFactory.getProxy(getProxyClassLoader());
}

public Object getProxy(@Nullable ClassLoader classLoader) {
    return createAopProxy().getProxy(classLoader);
}

protected final synchronized AopProxy createAopProxy() {
    if (!this.active) {
        activate();
    }
    // 其中 aopProxyFactory 方法获取的是 DefaultAopProxyFactory
    // createAopProxy 其实就用来判断使用的是哪个代理
    return getAopProxyFactory().createAopProxy(this);
}

@Override
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
    if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
        Class<?> targetClass = config.getTargetClass();
        if (targetClass == null) {
            throw new AopConfigException("TargetSource cannot determine target class: " +
                    "Either an interface or a target is required for proxy creation.");
        }
        if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
            return new JdkDynamicAopProxy(config);
        }
        return new ObjenesisCglibAopProxy(config);
    }
    else {
        return new JdkDynamicAopProxy(config);
    }
}
``` 

再看看创建代理方法  
```java
@Override
public Object getProxy(@Nullable ClassLoader classLoader) {
    if (logger.isDebugEnabled()) {
        logger.debug("Creating CGLIB proxy: target source is " + this.advised.getTargetSource());
    }

    try {
        Class<?> rootClass = this.advised.getTargetClass();
        Assert.state(rootClass != null, "Target class must be available for creating a CGLIB proxy");

        // 如果有实现接口，添加一下
        Class<?> proxySuperClass = rootClass;
        if (ClassUtils.isCglibProxyClass(rootClass)) {
            proxySuperClass = rootClass.getSuperclass();
            Class<?>[] additionalInterfaces = rootClass.getInterfaces();
            for (Class<?> additionalInterface : additionalInterfaces) {
                this.advised.addInterface(additionalInterface);
            }
        }

        // Validate the class, writing log messages as necessary.
        validateClassIfNecessary(proxySuperClass, classLoader);

        // cglib 过程了
        // Configure CGLIB Enhancer...
        Enhancer enhancer = createEnhancer();
        // 类加载器
        if (classLoader != null) {
            enhancer.setClassLoader(classLoader);
            if (classLoader instanceof SmartClassLoader &&
                    ((SmartClassLoader) classLoader).isClassReloadable(proxySuperClass)) {
                enhancer.setUseCache(false);
            }
        }
        // 要代理的对象
        enhancer.setSuperclass(proxySuperClass);
        // 要代理的接口
        enhancer.setInterfaces(AopProxyUtils.completeProxiedInterfaces(this.advised));
        // 生成的代理类名，使用自定义格式
        enhancer.setNamingPolicy(SpringNamingPolicy.INSTANCE);
        // 类加载器处理
        enhancer.setStrategy(new ClassLoaderAwareUndeclaredThrowableStrategy(classLoader));

        // 回调，也就是拦截处理器
        Callback[] callbacks = getCallbacks(rootClass);
        Class<?>[] types = new Class<?>[callbacks.length];
        for (int x = 0; x < types.length; x++) {
            types[x] = callbacks[x].getClass();
        }
        // fixedInterceptorMap only populated at this point, after getCallbacks call above
        // 回调过滤，会根据这个 filter 来选择使用哪个回调
        enhancer.setCallbackFilter(new ProxyCallbackFilter(
                this.advised.getConfigurationOnlyCopy(), this.fixedInterceptorMap, this.fixedInterceptorOffset));
        enhancer.setCallbackTypes(types);

        // 创建代理对象
        // Generate the proxy class and create a proxy instance.
        return createProxyClassAndInstance(enhancer, callbacks);
    }
    catch (CodeGenerationException | IllegalArgumentException ex) {
        throw new AopConfigException("Could not generate CGLIB subclass of " + this.advised.getTargetClass() +
                ": Common causes of this problem include using a final class or a non-visible class",
                ex);
    }
    catch (Throwable ex) {
        // TargetSource.getTarget() failed
        throw new AopConfigException("Unexpected AOP exception", ex);
    }
}
``` 

上面重点是 callback 的设置，spring 里面的callback有如下几个 
```java
// 这些是对应callback 在数组里面的索引，将根据 ProxyCallbackFilter 的返回值来判断使用哪个
// Constants for CGLIB callback array indices
// aop 回调
private static final int AOP_PROXY = 0;
private static final int INVOKE_TARGET = 1;
private static final int NO_OVERRIDE = 2;
private static final int DISPATCH_TARGET = 3;
private static final int DISPATCH_ADVISED = 4;
private static final int INVOKE_EQUALS = 5;
private static final int INVOKE_HASHCODE = 6;

// 根据这个方法的返回值，来判断使用哪个
public int accept(Method method) {
    if (AopUtils.isFinalizeMethod(method)) {
        logger.debug("Found finalize() method - using NO_OVERRIDE");
        return NO_OVERRIDE;
    }
    if (!this.advised.isOpaque() && method.getDeclaringClass().isInterface() &&
            method.getDeclaringClass().isAssignableFrom(Advised.class)) {
        if (logger.isDebugEnabled()) {
            logger.debug("Method is declared on Advised interface: " + method);
        }
        return DISPATCH_ADVISED;
    }
    // We must always proxy equals, to direct calls to this.
    if (AopUtils.isEqualsMethod(method)) {
        logger.debug("Found 'equals' method: " + method);
        return INVOKE_EQUALS;
    }
    // We must always calculate hashCode based on the proxy.
    if (AopUtils.isHashCodeMethod(method)) {
        logger.debug("Found 'hashCode' method: " + method);
        return INVOKE_HASHCODE;
    }
    Class<?> targetClass = this.advised.getTargetClass();
    // Proxy is not yet available, but that shouldn't matter.
    List<?> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
    boolean haveAdvice = !chain.isEmpty();
    boolean exposeProxy = this.advised.isExposeProxy();
    boolean isStatic = this.advised.getTargetSource().isStatic();
    boolean isFrozen = this.advised.isFrozen();
    if (haveAdvice || !isFrozen) {
        // If exposing the proxy, then AOP_PROXY must be used.
        if (exposeProxy) {
            if (logger.isDebugEnabled()) {
                logger.debug("Must expose proxy on advised method: " + method);
            }
            return AOP_PROXY;
        }
        String key = method.toString();
        // Check to see if we have fixed interceptor to serve this method.
        // Else use the AOP_PROXY.
        if (isStatic && isFrozen && this.fixedInterceptorMap.containsKey(key)) {
            if (logger.isDebugEnabled()) {
                logger.debug("Method has advice and optimizations are enabled: " + method);
            }
            // We know that we are optimizing so we can use the FixedStaticChainInterceptors.
            int index = this.fixedInterceptorMap.get(key);
            return (index + this.fixedInterceptorOffset);
        }
        else {
            if (logger.isDebugEnabled()) {
                logger.debug("Unable to apply any optimizations to advised method: " + method);
            }
            return AOP_PROXY;
        }
    }
    else {
        // See if the return type of the method is outside the class hierarchy of the target type.
        // If so we know it never needs to have return type massage and can use a dispatcher.
        // If the proxy is being exposed, then must use the interceptor the correct one is already
        // configured. If the target is not static, then we cannot use a dispatcher because the
        // target needs to be explicitly released after the invocation.
        if (exposeProxy || !isStatic) {
            return INVOKE_TARGET;
        }
        Class<?> returnType = method.getReturnType();
        if (targetClass != null && returnType.isAssignableFrom(targetClass)) {
            if (logger.isDebugEnabled()) {
                logger.debug("Method return type is assignable from target type and " +
                        "may therefore return 'this' - using INVOKE_TARGET: " + method);
            }
            return INVOKE_TARGET;
        }
        else {
            if (logger.isDebugEnabled()) {
                logger.debug("Method return type ensures 'this' cannot be returned - " +
                        "using DISPATCH_TARGET: " + method);
            }
            return DISPATCH_TARGET;
        }
    }
}

``` 

里面重点的是 AOP 的那个回调 
```java
private static class DynamicAdvisedInterceptor implements MethodInterceptor, Serializable {

    private final AdvisedSupport advised;

    public DynamicAdvisedInterceptor(AdvisedSupport advised) {
        this.advised = advised;
    }

    @Override
    @Nullable
    public Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
        Object oldProxy = null;
        boolean setProxyContext = false;
        Object target = null;
        TargetSource targetSource = this.advised.getTargetSource();
        try {
            // 这个默认是 false，不需要管
            if (this.advised.exposeProxy) {
                // Make invocation available if necessary.
                oldProxy = AopContext.setCurrentProxy(proxy);
                setProxyContext = true;
            }
            // Get as late as possible to minimize the time we "own" the target, in case it comes from a pool...
            target = targetSource.getTarget();
            Class<?> targetClass = (target != null ? target.getClass() : null);
            // 看拦截链，一个方法可以触发多个通知
            List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
            Object retVal;
            // Check whether we only have one InvokerInterceptor: that is,
            // no real advice, but just reflective invocation of the target.
            // 没有连接，直接调用
            if (chain.isEmpty() && Modifier.isPublic(method.getModifiers())) {
                // We can skip creating a MethodInvocation: just invoke the target directly.
                // Note that the final invoker must be an InvokerInterceptor, so we know
                // it does nothing but a reflective operation on the target, and no hot
                // swapping or fancy proxying.
                Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
                retVal = methodProxy.invoke(target, argsToUse);
            }
            else {
                // 创建一个 方法调用链
                // We need to create a method invocation...
                retVal = new CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy).proceed();
            }
            retVal = processReturnType(proxy, target, method, retVal);
            return retVal;
        }
        finally {
            if (target != null && !targetSource.isStatic()) {
                targetSource.releaseTarget(target);
            }
            if (setProxyContext) {
                // Restore old proxy.
                AopContext.setCurrentProxy(oldProxy);
            }
        }
    }

    @Override
    public boolean equals(Object other) {
        return (this == other ||
                (other instanceof DynamicAdvisedInterceptor &&
                        this.advised.equals(((DynamicAdvisedInterceptor) other).advised)));
    }

    /**
        * CGLIB uses this to drive proxy creation.
        */
    @Override
    public int hashCode() {
        return this.advised.hashCode();
    }
}
``` 


## 总结
上面的代码是spring对 aop 的处理流程，现在在梳理一遍 
aop 使用的原理是 动态代理，这句话没毛病，针对接口，使用 jdk 的动态代理，针对类，使用 cglib 的动态代理   
代理对象的创建，是使用spring 的扩展接口，在bean 实例化过程中切入的  
切入的过程是 找对有 @Aspect 注解的 bean，排除所有 @Pointcut 注解的方法，其他都可能通知点 Advice     
使用这些通知点构建通知器 Advisor，通知器作用就是返回 Advice     
然后根据条件，看该通知器是否跟对应的目标对象匹配，如果匹配，就可以创建代理了