# spring 分析

前面的分析，没有分析 bean 实例化的具体流程
这里具体分析一下

```java
// 开始实例化 singletons 类型的 bean 了
// Instantiate all remaining (non-lazy-init) singletons.
beanFactory.preInstantiateSingletons();
``` 

这里的 beanFactory 是 DefaultListableBeanFactory 实例

```java
public void preInstantiateSingletons() throws BeansException {
    if (this.logger.isDebugEnabled()) {
        this.logger.debug("Pre-instantiating singletons in " + this);
    }

    // 拿到所有 BeanDefinition 的name 复制
    // Iterate over a copy to allow for init methods which in turn register new bean definitions.
    // While this may not be part of the regular factory bootstrap, it does otherwise work fine.
    List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

    // 遍历一遍，初始化所有非 lazy 的 单例
    // Trigger initialization of all non-lazy singleton beans...
    for (String beanName : beanNames) {
        // 获得 BeanDefinition 定义，
        // getMergedLocalBeanDefinition 方法主要是处理 BeanDefinition 有继承其他 BeanDefinition 的情况，那就复制父 BeanDefinition 的信息，然后用本 BeanDefinition 覆盖
        RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
        // 非抽象，单例，非延时，才初始化
        if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
            // 如果是 FactoryBean
            if (isFactoryBean(beanName)) {
                // 初始化 FactoryBean
                Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
                if (bean instanceof FactoryBean) {
                    final FactoryBean<?> factory = (FactoryBean<?>) bean;
                    boolean isEagerInit;
                    if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
                        isEagerInit = AccessController.doPrivileged((PrivilegedAction<Boolean>)
                                        ((SmartFactoryBean<?>) factory)::isEagerInit,
                                getAccessControlContext());
                    }
                    else {
                        isEagerInit = (factory instanceof SmartFactoryBean &&
                                ((SmartFactoryBean<?>) factory).isEagerInit());
                    }
                    // 判断 FactoryBean 是否需要及时初始化，如果需要，就初始化
                    if (isEagerInit) {
                        getBean(beanName);
                    }
                }
            }
            else {
                // 不是 FactoryBean 正常初始化
                getBean(beanName);
            }
        }
    }

    // Trigger post-initialization callback for all applicable beans...
    // 这里判断 bean 实例是否实现类 SmartInitializingSingleton ，如果实现类，就调用 afterSingletonsInstantiated 方法
    for (String beanName : beanNames) {
        // 获取 单例
        Object singletonInstance = getSingleton(beanName);
        if (singletonInstance instanceof SmartInitializingSingleton) {
            final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
            if (System.getSecurityManager() != null) {
                AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
                    smartSingleton.afterSingletonsInstantiated();
                    return null;
                }, getAccessControlContext());
            }
            else {
                smartSingleton.afterSingletonsInstantiated();
            }
        }
    }
}
``` 

实例化的实际方式是 getBean 方法
```java
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
        @Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {

    // 获取 beanName
    final String beanName = transformedBeanName(name);
    Object bean;

    // 获取给定 beanName 的单例对象
    // Eagerly check singleton cache for manually registered singletons.
    // 这个方法获取步骤
    // 1. 从已经创建的map中获取 singletonObjects
    // 2. 没有获取到，从早期缓存获取 earlySingletonObjects
    // 3. 如果还没有获取到，则尝试获取该 bean 的对象工厂,后面如果需要支持循环依赖，会丢一个创建对象工厂
    // 4. 如果获取到工厂，就创建对象，丢到 earlySingletonObjects
    // 5. 没有对象工厂就返回 null
    Object sharedInstance = getSingleton(beanName);
    if (sharedInstance != null && args == null) {
        if (logger.isDebugEnabled()) {
            if (isSingletonCurrentlyInCreation(beanName)) {
                logger.debug("Returning eagerly cached instance of singleton bean '" + beanName +
                        "' that is not fully initialized yet - a consequence of a circular reference");
            }
            else {
                logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
            }
        }
        // getObjectForBeanInstance 方法是处理 FactoryBean 的情况
        bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
    }

    else {
        // prototype 类型的bean在创建中，提前异常出去
        // Fail if we're already creating this bean instance:
        // We're assumably within a circular reference.
        if (isPrototypeCurrentlyInCreation(beanName)) {
            throw new BeanCurrentlyInCreationException(beanName);
        }

        // 检查 BeanDefinition 在不在本 factory 中，如果不在，就尝试从 parent getBean 完成初始化
        // Check if bean definition exists in this factory.
        BeanFactory parentBeanFactory = getParentBeanFactory();
        if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
            // Not found -> check parent.
            String nameToLookup = originalBeanName(name);
            if (parentBeanFactory instanceof AbstractBeanFactory) {
                return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
                        nameToLookup, requiredType, args, typeCheckOnly);
            }
            else if (args != null) {
                // Delegation to parent with explicit args.
                return (T) parentBeanFactory.getBean(nameToLookup, args);
            }
            else {
                // No args -> delegate to standard getBean method.
                return parentBeanFactory.getBean(nameToLookup, requiredType);
            }
        }

        // 不做类型检查，标记 beanName 为已经创建，alreadyCreated。add
        if (!typeCheckOnly) {
            markBeanAsCreated(beanName);
        }

        try {
            // 获取 BeanDefinition，前面说过了，getMergedLocalBeanDefinition 方法是用来合并继承过来的 BeanDefinition
            final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
            // 非抽象
            checkMergedBeanDefinition(mbd, beanName, args);

            // 这里的 dependsOn 不是 bean 需要注入啥，而是使用 @DependsOn 注解标记的，代表需要提前初始化的
            // Guarantee initialization of beans that the current bean depends on.
            String[] dependsOn = mbd.getDependsOn();
            if (dependsOn != null) {
                for (String dep : dependsOn) {
                    // 检查 beanName 是否依赖 dep，直接，间接依赖都算
                    // 这里是以 beanName 为key 获取，如果有 dep，那就是循环依赖了
                    if (isDependent(beanName, dep)) {
                        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
                    }
                    // 这里注册依赖关系 key:dep  value:[...,beanName]
                    // 所以存的含义是 value 为 依赖 key 的 beanName 集合
                    registerDependentBean(dep, beanName);
                    try {
                        // 先初始化依赖的 bean
                        getBean(dep);
                    }
                    catch (NoSuchBeanDefinitionException ex) {
                        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                "'" + beanName + "' depends on missing bean '" + dep + "'", ex);
                    }
                }
            }
            // 如果是 单例
            // Create bean instance.
            if (mbd.isSingleton()) {
                // 这里传类一个 lambda，是一个 ObjectFactory，对象创建逻辑就是 createBean 方法
                sharedInstance = getSingleton(beanName, () -> {
                    try {
                        // 需要注意的是，通常来说，调用getBean 的时候很少传参数进来的，因此 args 大部分时候都是为null 的
						// spring 初始化的时候也是为 null 的
                        return createBean(beanName, mbd, args);
                    }
                    catch (BeansException ex) {
                        // Explicitly remove instance from singleton cache: It might have been put there
                        // eagerly by the creation process, to allow for circular reference resolution.
                        // Also remove any beans that received a temporary reference to the bean.
                        destroySingleton(beanName);
                        throw ex;
                    }
                });
                // getObjectForBeanInstance 方法是处理 FactoryBean 的情况
                bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
            }
            // 原型
            else if (mbd.isPrototype()) {
                // It's a prototype -> create a new instance.
                Object prototypeInstance = null;
                try {
                    beforePrototypeCreation(beanName);
                    prototypeInstance = createBean(beanName, mbd, args);
                }
                finally {
                    afterPrototypeCreation(beanName);
                }
                bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
            }
            // 其他作用域类型，扩展的 譬如 request session
            else {
                String scopeName = mbd.getScope();
                final Scope scope = this.scopes.get(scopeName);
                if (scope == null) {
                    throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
                }
                try {
                    Object scopedInstance = scope.get(beanName, () -> {
                        beforePrototypeCreation(beanName);
                        try {
                            return createBean(beanName, mbd, args);
                        }
                        finally {
                            afterPrototypeCreation(beanName);
                        }
                    });
                    bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
                }
                catch (IllegalStateException ex) {
                    throw new BeanCreationException(beanName,
                            "Scope '" + scopeName + "' is not active for the current thread; consider " +
                            "defining a scoped proxy for this bean if you intend to refer to it from a singleton",
                            ex);
                }
            }
        }
        catch (BeansException ex) {
            cleanupAfterBeanCreationFailure(beanName);
            throw ex;
        }
    }

    // 检查类型，如果需要，则进行类型转换
    // Check if required type matches the type of the actual bean instance.
    if (requiredType != null && !requiredType.isInstance(bean)) {
        try {
            T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
            if (convertedBean == null) {
                throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
            }
            return convertedBean;
        }
        catch (TypeMismatchException ex) {
            if (logger.isDebugEnabled()) {
                logger.debug("Failed to convert bean '" + name + "' to required type '" +
                        ClassUtils.getQualifiedName(requiredType) + "'", ex);
            }
            throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
        }
    }
    return (T) bean;
}
``` 

我们先只关心单例的创建。重要的方法有 getSingleton，getObjectForBeanInstance，createBean 方法    
```java
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    // 先从已经生成好的对象里面获取
    Object singletonObject = this.singletonObjects.get(beanName);
    // 如果没有，并且在创建中
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        // 先上锁
        synchronized (this.singletonObjects) {
            // 从更早的缓存获取,早期缓存是为了处理循环依赖用的
            singletonObject = this.earlySingletonObjects.get(beanName);
            // 如果还没有，并且允许早期引用
            if (singletonObject == null && allowEarlyReference) {
                // 获取对象工厂
                ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                // 如果存在
                if (singletonFactory != null) {
                    // 创建对象
                    singletonObject = singletonFactory.getObject();
                    // 丢到早期缓存里面
                    this.earlySingletonObjects.put(beanName, singletonObject);
                    // 移除对象工厂（单例），不再需要工厂创建了
                    this.singletonFactories.remove(beanName);
                }
            }
        }
    }
    return singletonObject;
}   

protected Object getObjectForBeanInstance(
        Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {

    // 看是不是 FactoryBean 的 beanName（&开头的）
    // Don't let calling code try to dereference the factory if the bean isn't a factory.
    if (BeanFactoryUtils.isFactoryDereference(name)) {
        if (beanInstance instanceof NullBean) {
            return beanInstance;
        }
        if (!(beanInstance instanceof FactoryBean)) {
            throw new BeanIsNotAFactoryException(transformedBeanName(name), beanInstance.getClass());
        }
    }

    // 如果不是，直接返回
    // Now we have the bean instance, which may be a normal bean or a FactoryBean.
    // If it's a FactoryBean, we use it to create a bean instance, unless the
    // caller actually wants a reference to the factory.
    if (!(beanInstance instanceof FactoryBean) || BeanFactoryUtils.isFactoryDereference(name)) {
        return beanInstance;
    }

    Object object = null;
    // 先从缓存里面获取
    if (mbd == null) {
        object = getCachedObjectForFactoryBean(beanName);
    }
    // 没有，就根据 FactoryBean 的方法创建
    if (object == null) {
        // Return bean instance from factory.
        FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
        // Caches object obtained from FactoryBean if it is a singleton.
        if (mbd == null && containsBeanDefinition(beanName)) {
            mbd = getMergedLocalBeanDefinition(beanName);
        }
        boolean synthetic = (mbd != null && mbd.isSynthetic());
        // 老复杂的方法了，后面细看
        object = getObjectFromFactoryBean(factory, beanName, !synthetic);
    }
    return object;
}

protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
        throws BeanCreationException {

    if (logger.isDebugEnabled()) {
        logger.debug("Creating instance of bean '" + beanName + "'");
    }
    RootBeanDefinition mbdToUse = mbd;

    // Make sure bean class is actually resolved at this point, and
    // clone the bean definition in case of a dynamically resolved Class
    // which cannot be stored in the shared merged bean definition.
    // 获取 beanClass
    Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
    if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
        // 如果是从 beanClassName 解析出来的 class，重新生成 BeanDefinition
        mbdToUse = new RootBeanDefinition(mbd);
        mbdToUse.setBeanClass(resolvedClass);
    }

    // Prepare method overrides.
    try {
        // 检查 overrides
        // 不太清楚是干啥的
        mbdToUse.prepareMethodOverrides();
    }
    catch (BeanDefinitionValidationException ex) {
        throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
                beanName, "Validation of method overrides failed", ex);
    }

    try {
        // 在生成实例之前，这里有一个机会，给对应 BeanPostProcessor 直接生成实例的地方，如果没人处理，就走 spring 下面的创建流程
        // 只有有实现 InstantiationAwareBeanPostProcessor （ BeanPostProcessor 的子接口），才进行处理
        // Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
        Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
        if (bean != null) {
            return bean;
        }
    }
    catch (Throwable ex) {
        throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
                "BeanPostProcessor before instantiation of bean failed", ex);
    }

    try {
        // 这里就是 spring 默认的创建流程，如果一个类型的bean，没有被 BeanProcessor 提前处理，那就会到这里
        Object beanInstance = doCreateBean(beanName, mbdToUse, args);
        if (logger.isDebugEnabled()) {
            logger.debug("Finished creating instance of bean '" + beanName + "'");
        }
        return beanInstance;
    }
    catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
        // A previously detected exception with proper bean creation context already,
        // or illegal singleton state to be communicated up to DefaultSingletonBeanRegistry.
        throw ex;
    }
    catch (Throwable ex) {
        throw new BeanCreationException(
                mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
    }
}


``` 

上面中，重要的方法是 doCreateBean，这个方法是创建的正常流程
```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
        throws BeanCreationException {
    // 这里再注释一遍，args 大部分时候为 null
    // Instantiate the bean.
    BeanWrapper instanceWrapper = null;
    // factoryBeanInstanceCache 的注释，如果 beanName 对应的不是 FactoryBean 忽略这里
    // Cache of unfinished FactoryBean instances: FactoryBean name --> BeanWrapper
    if (mbd.isSingleton()) {
        instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
    }
    // 如果不是 FactoryBean ，或者 FactoryBean 是已经完成创建的
    if (instanceWrapper == null) {
        // 这里创建实例，就是构建一个对象出来，不过有 构造方法注入，所以需要处理，这个后面再看
        // 实例可以 factoryBean ，factoryMethod，构造方法
        // 创建过程老复杂了，后面可以详细看看
        instanceWrapper = createBeanInstance(beanName, mbd, args);
    }
    // 这里先获取一些基本信息，设置解析出来的类型
    final Object bean = instanceWrapper.getWrappedInstance();
    Class<?> beanType = instanceWrapper.getWrappedClass();
    if (beanType != NullBean.class) {
        mbd.resolvedTargetType = beanType;
    }

    // 开始要执行 BeanPostProcessor 了，先上锁
    // Allow post-processors to modify the merged bean definition.
    synchronized (mbd.postProcessingLock) {
        // 判断是否处理过了
        if (!mbd.postProcessed) {
            try {
                // 处理 MergedBeanDefinitionPostProcessor （BeanPostProcessor 的子接口）的 postProcessMergedBeanDefinition 方法
                // 这里给个机会修改一下 BeanDefinition，但是 beanType 没法变了，实例已经创建好了，后面挺多设置一下 properties 或者依赖设置吧
                applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
            }
            catch (Throwable ex) {
                throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                        "Post-processing of merged bean definition failed", ex);
            }
            // 标记处理过类
            mbd.postProcessed = true;
        }
    }

    // 这里是可以运行循环依赖的处理
    // 当 A 创建一半，发现需要先创建 B，然后去创建 B，然而 B 也依赖 A了，这个时候，虽然 A 还没创建完
    // 这里就允许这种情况，因为 A 的实例已经创建好了，只是一些属性还没设置完全，依赖只是有一个引用
    // Eagerly cache singletons to be able to resolve circular references
    // even when triggered by lifecycle interfaces like BeanFactoryAware.
    boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
            isSingletonCurrentlyInCreation(beanName));
    if (earlySingletonExposure) {
        if (logger.isDebugEnabled()) {
            logger.debug("Eagerly caching bean '" + beanName +
                    "' to allow for resolving potential circular references");
        }
        // 添加 对象工厂
        // 处理 SmartInstantiationAwareBeanPostProcessor 接口（getEarlyBeanReference 方法）
        // 当有循环依赖，需要获取早期引用的时候，会调用 getEarlyBeanReference 方法（SmartInstantiationAwareBeanPostProcessor 的方法）
        addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
    }

    // 初始化
    // Initialize the bean instance.
    Object exposedObject = bean;
    try {
        // 这里依赖注入
        // 老复杂了
        populateBean(beanName, mbd, instanceWrapper);
        // 初始化方法，BeanPostProcessor 接口
        exposedObject = initializeBean(beanName, exposedObject, mbd);
    }
    catch (Throwable ex) {
        if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
            throw (BeanCreationException) ex;
        }
        else {
            throw new BeanCreationException(
                    mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
        }
    }

    // 需要提前暴露出去
    if (earlySingletonExposure) {
        Object earlySingletonReference = getSingleton(beanName, false);
        // 不为 null，才处理，因为不为 null，代表还没有其他对象引用
        if (earlySingletonReference != null) {
            // 这里是代表，经过 BeanPostProcessor 的处理，bean 的引用地址没有改变
            // 也就是说，如果 BeanPostProcessor 没有处理，或者处理的时候 new 新对象，是直接用之前的 bean = new ，而不是直接返回 new，那引用就没有变化
            if (exposedObject == bean) {
                exposedObject = earlySingletonReference;
            }
            // 下面检查是否有依赖了旧版本的 bean
            else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
                String[] dependentBeans = getDependentBeans(beanName);
                Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
                for (String dependentBean : dependentBeans) {
                    if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
                        actualDependentBeans.add(dependentBean);
                    }
                }
                if (!actualDependentBeans.isEmpty()) {
                    throw new BeanCurrentlyInCreationException(beanName,
                            "Bean with name '" + beanName + "' has been injected into other beans [" +
                            StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
                            "] in its raw version as part of a circular reference, but has eventually been " +
                            "wrapped. This means that said other beans do not use the final version of the " +
                            "bean. This is often the result of over-eager type matching - consider using " +
                            "'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
                }
            }
        }
    }

    // 注册 disposable bean，这样，当 容器关闭之前，先调用这些bean 的 destroy方法
    // Register bean as disposable.
    try {
        registerDisposableBeanIfNecessary(beanName, bean, mbd);
    }
    catch (BeanDefinitionValidationException ex) {
        throw new BeanCreationException(
                mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
    }

    return exposedObject;
}
``` 

里面有两个老复杂的方法，后面先不看了
1. createBeanInstance 这个方法，如果是构造方法创建，那么会有构造注入
2. populateBean 这个方法处理依赖注入

```java
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
    // Make sure bean class is actually resolved at this point.
    // 确保 class 已经解析了
    Class<?> beanClass = resolveBeanClass(mbd, beanName);

    if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                "Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
    }

    // 对象提供接口
    Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
    if (instanceSupplier != null) {
        return obtainFromSupplier(instanceSupplier, beanName);
    }

    // factory method 方法获取对象
    if (mbd.getFactoryMethodName() != null)  {
        return instantiateUsingFactoryMethod(beanName, mbd, args);
    }

    // 构造方法
    // Shortcut when re-creating the same bean...
    // 判断一下是否处理过了，就是解析出来要用的 构造方法和参数没
    boolean resolved = false;
    boolean autowireNecessary = false;
    if (args == null) {
        synchronized (mbd.constructorArgumentLock) {
            if (mbd.resolvedConstructorOrFactoryMethod != null) {
                resolved = true;
                autowireNecessary = mbd.constructorArgumentsResolved;
            }
        }
    }
    // 后面就是用解析出来的使用的构造方法 或者默认构造方法构建实例了
    // 解析出来的构造方法，需要注入
    if (resolved) {
        if (autowireNecessary) {
            return autowireConstructor(beanName, mbd, null, null);
        }
        else {
            return instantiateBean(beanName, mbd);
        }
    }

    // Need to determine the constructor...
    // 解析要用的构造方法
    // 解析是用 BeanPostProcessor 接口的子接口 SmartInstantiationAwareBeanPostProcessor 的来扩展的
    // 注入相关的是 AutowiredAnnotationBeanPostProcessor 处理
    Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
    if (ctors != null ||
            mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_CONSTRUCTOR ||
            mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args))  {
        // 构造方法依赖注入
        return autowireConstructor(beanName, mbd, ctors, args);
    }

    // 不需要特殊处理，使用默认的构造方法
    // No special handling: simply use no-arg constructor.
    return instantiateBean(beanName, mbd);
}

// 里面老复杂，流程就是找构造方法，选择使用的构造方法，根据构造方法的参数，选择注入，看是 value 还是 需要注入 bean，如果需要注入bean 那就getBean 初始化
protected BeanWrapper autowireConstructor(
        String beanName, RootBeanDefinition mbd, @Nullable Constructor<?>[] ctors, @Nullable Object[] explicitArgs) {

    return new ConstructorResolver(this).autowireConstructor(beanName, mbd, ctors, explicitArgs);
}

// 这里是根据 name 和 type 注入的地方，就不细看了
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
    if (bw == null) {
        if (mbd.hasPropertyValues()) {
            throw new BeanCreationException(
                    mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
        }
        else {
            // Skip property population phase for null instance.
            return;
        }
    }

    // Give any InstantiationAwareBeanPostProcessors the opportunity to modify the
    // state of the bean before properties are set. This can be used, for example,
    // to support styles of field injection.
    boolean continueWithPropertyPopulation = true;

    if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
        for (BeanPostProcessor bp : getBeanPostProcessors()) {
            if (bp instanceof InstantiationAwareBeanPostProcessor) {
                InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
                    continueWithPropertyPopulation = false;
                    break;
                }
            }
        }
    }

    if (!continueWithPropertyPopulation) {
        return;
    }

    PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);

    if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME ||
            mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
        MutablePropertyValues newPvs = new MutablePropertyValues(pvs);

        // Add property values based on autowire by name if applicable.
        if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME) {
            // 根据 beanName 注入
            autowireByName(beanName, mbd, bw, newPvs);
        }

        // Add property values based on autowire by type if applicable.
        if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
            // 类型注入
            autowireByType(beanName, mbd, bw, newPvs);
        }

        pvs = newPvs;
    }

    boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
    boolean needsDepCheck = (mbd.getDependencyCheck() != RootBeanDefinition.DEPENDENCY_CHECK_NONE);

    if (hasInstAwareBpps || needsDepCheck) {
        if (pvs == null) {
            pvs = mbd.getPropertyValues();
        }
        PropertyDescriptor[] filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
        if (hasInstAwareBpps) {
            for (BeanPostProcessor bp : getBeanPostProcessors()) {
                if (bp instanceof InstantiationAwareBeanPostProcessor) {
                    InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                    pvs = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
                    if (pvs == null) {
                        return;
                    }
                }
            }
        }
        if (needsDepCheck) {
            checkDependencies(beanName, mbd, filteredPds, pvs);
        }
    }

    if (pvs != null) {
        applyPropertyValues(beanName, mbd, bw, pvs);
    }
}
``` 
