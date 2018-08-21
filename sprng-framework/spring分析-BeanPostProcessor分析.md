[TOC]

# spring BeanPostProcessor 分析
这里主要分析 spring 内部注册的 BeanPostProcessor，看看 spring 是怎么处理的  
先贴上实例化的代码  
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


## BeanPostProcessor
这个是 spring 框架扩展的核心接口，其他众所周知的各种 *Aware 接口，都是基于这个接口实现的    
譬如 `ApplicationContextAwareProcessor` 类，就是处理 `EnvironmentAware`, `EmbeddedValueResolverAware`, `ResourceLoaderAware`, `ApplicationEventPublisherAware`, `MessageSourceAware`, `ApplicationContextAware` 接口的
PS: 不是所有 Aware 都是通过这个接口扩展的，还有一些譬如 BeanClassloaderAware 之类的，只在初始化过程中触发的
```java
public Object postProcessBeforeInitialization(final Object bean, String beanName) throws BeansException {
    AccessControlContext acc = null;

    if (System.getSecurityManager() != null &&
            (bean instanceof EnvironmentAware || bean instanceof EmbeddedValueResolverAware ||
                    bean instanceof ResourceLoaderAware || bean instanceof ApplicationEventPublisherAware ||
                    bean instanceof MessageSourceAware || bean instanceof ApplicationContextAware)) {
        acc = this.applicationContext.getBeanFactory().getAccessControlContext();
    }

    if (acc != null) {
        AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
            invokeAwareInterfaces(bean);
            return null;
        }, acc);
    }
    else {
        invokeAwareInterfaces(bean);
    }

    return bean;
}

private void invokeAwareInterfaces(Object bean) {
    if (bean instanceof Aware) {
        if (bean instanceof EnvironmentAware) {
            ((EnvironmentAware) bean).setEnvironment(this.applicationContext.getEnvironment());
        }
        if (bean instanceof EmbeddedValueResolverAware) {
            ((EmbeddedValueResolverAware) bean).setEmbeddedValueResolver(this.embeddedValueResolver);
        }
        if (bean instanceof ResourceLoaderAware) {
            ((ResourceLoaderAware) bean).setResourceLoader(this.applicationContext);
        }
        if (bean instanceof ApplicationEventPublisherAware) {
            ((ApplicationEventPublisherAware) bean).setApplicationEventPublisher(this.applicationContext);
        }
        if (bean instanceof MessageSourceAware) {
            ((MessageSourceAware) bean).setMessageSource(this.applicationContext);
        }
        if (bean instanceof ApplicationContextAware) {
            ((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
        }
    }
}
``` 

`BeanPostProcessor` 这个接口有两个方法 
```java
// 这个方法代表初始化之前
default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
    return bean;
}

// 这个方法在初始化之后
default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
    return bean;
}
``` 

什么是初始化前，什么是初始化后，我们再瞅一下 bean 的实例化代码实现  
当一个 bean 创建完毕，依赖注入完毕后，才会开始 `BeanPostProcessor` 的处理，处理的方法是 `AbstractAutowireCapableBeanFactory.initializeBean` 方法
```java
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
// 上面至少发一下调用的地方

protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
    // 触发 Aware 接口
    // BeanNameAware  BeanClassLoaderAware  BeanFactoryAware
    if (System.getSecurityManager() != null) {
        AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
            invokeAwareMethods(beanName, bean);
            return null;
        }, getAccessControlContext());
    }
    else {
        invokeAwareMethods(beanName, bean);
    }

    Object wrappedBean = bean;
    if (mbd == null || !mbd.isSynthetic()) {
        // postProcessBeforeInitialization
        // 这里就是初始化之前 postProcessBeforeInitialization 方法调用
        wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
    }

    try {
        // 调用初始化方法 InitializingBean （afterPropertiesSet 方法） 注意，@PostConstruct 注解不在这里处理 
        invokeInitMethods(beanName, wrappedBean, mbd);
    }
    catch (Throwable ex) {
        throw new BeanCreationException(
                (mbd != null ? mbd.getResourceDescription() : null),
                beanName, "Invocation of init method failed", ex);
    }
    if (mbd == null || !mbd.isSynthetic()) {
        // postProcessAfterInitialization 处理
        // 这里就是初始化之后 postProcessAfterInitialization 方法调用
        wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    }

    return wrappedBean;
}

// 通常这两个扩展方法，如果实现类不需要处理，原样返回就可以了，默认方法（jdk8 的接口默认方法）就是这样的
@Override
public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
        throws BeansException {

    Object result = existingBean;
    for (BeanPostProcessor beanProcessor : getBeanPostProcessors()) {
        Object current = beanProcessor.postProcessBeforeInitialization(result, beanName);
        if (current == null) {
            return result;
        }
        result = current;
    }
    return result;
}

@Override
public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
        throws BeansException {

    Object result = existingBean;
    for (BeanPostProcessor beanProcessor : getBeanPostProcessors()) {
        Object current = beanProcessor.postProcessAfterInitialization(result, beanName);
        if (current == null) {
            return result;
        }
        result = current;
    }
    return result;
}
``` 

上面也可以看到 

```java
// 调用初始化方法 InitializingBean （afterPropertiesSet 方法）  自定义的初始化方法
        invokeInitMethods(beanName, wrappedBean, mbd);
```
bean 的初始化方法，是在这两个扩展调用之间的

## MergedBeanDefinitionPostProcessor
这个接口是 `BeanPostProcessor` 的子接口，多了一个方法   
```java
   /**
    * Post-process the given merged bean definition for the specified bean.
    * @param beanDefinition the merged bean definition for the bean
    * @param beanType the actual type of the managed bean instance
    * @param beanName the name of the bean
    */
void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName);
``` 

首先，什么是 `MergedBeanDefinition`? 因为 `BeanDefinition` 是可以继承的,因此，以 `BeanDefinition` 为模板构建实例的时候，首先要从父 `BeanDefinition` 拿到所有信息之后才开始构建  
然后，这个方法的触发在什么时候呢？答案是在 `BeanPostProcessor` 的两个方法之前，在spring 创建好实例之后，没依赖注入（非构造注入）之前,允许修改 `BeanDefinition`  
我们看看发生的地方  
```java
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
``` 
这里，上面已经说明了，允许修改 `BeanDefinition`     
那么，一个具体的应用实例，就是 spring 自己的 `@PostConstruct` 和 `@PreDestroy` 注解的处理    
在类 `InitDestroyAnnotationBeanPostProcessor` 中    
```java
@Override
public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {
    LifecycleMetadata metadata = findLifecycleMetadata(beanType);
    metadata.checkConfigMembers(beanDefinition);
}

public void checkConfigMembers(RootBeanDefinition beanDefinition) {
    Set<LifecycleElement> checkedInitMethods = new LinkedHashSet<>(this.initMethods.size());
    for (LifecycleElement element : this.initMethods) {
        String methodIdentifier = element.getIdentifier();
        if (!beanDefinition.isExternallyManagedInitMethod(methodIdentifier)) {
            // 这里注册 init
            beanDefinition.registerExternallyManagedInitMethod(methodIdentifier);
            checkedInitMethods.add(element);
            if (logger.isDebugEnabled()) {
                logger.debug("Registered init method on class [" + this.targetClass.getName() + "]: " + element);
            }
        }
    }
    Set<LifecycleElement> checkedDestroyMethods = new LinkedHashSet<>(this.destroyMethods.size());
    for (LifecycleElement element : this.destroyMethods) {
        String methodIdentifier = element.getIdentifier();
        if (!beanDefinition.isExternallyManagedDestroyMethod(methodIdentifier)) {
            // 这里注册 destroy
            beanDefinition.registerExternallyManagedDestroyMethod(methodIdentifier);
            checkedDestroyMethods.add(element);
            if (logger.isDebugEnabled()) {
                logger.debug("Registered destroy method on class [" + this.targetClass.getName() + "]: " + element);
            }
        }
    }
    this.checkedInitMethods = checkedInitMethods;
    this.checkedDestroyMethods = checkedDestroyMethods;
}
``` 

他在这里注册了一些 `@PostConstruct` 注解标记的初始化方法，那么是在什么时候调用的呢？答案是在 `postProcessBeforeInitialization` 这个方法里面 
而我们之前说过了，这个扩展的调用是在 `InitializingBean` 接口 和 指定初始化方法（使用 `@Bean` 注解就可以指定）之前的。   
因此，很显然，调用顺序是 1. `@PostConstruct` 2. `InitializingBean` 3. init-method   


## InstantiationAwareBeanPostProcessor
这个接口里面多了3个方法，实例化之前，实例化之后，处理 properties。  
```java
@Nullable
default Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
    return null;
}

default boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException {
    return true;
}

@Nullable
default PropertyValues postProcessPropertyValues(
        PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) throws BeansException {

    return pvs;
}
``` 

需要注意的是这里的方法名 `Instantiation` 跟 `BeanPostFactory` 的方法名 `Initialization` 是不一样的  
我们要注意到，在 `BeanPostFactory` 接口的两个方法中，参数都是一个 `Object` 对象，这说明，在调用这里的时候，对象已经是创建好的。 
但是 `InstantiationAwareBeanPostProcessor` 的接口方法 before，参数只是 class，所以，大概就猜到这个接口是在 `BeanPostFactory` 之前触发了，去源码确认一下 
```java
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
``` 

注意，正常的流程是 `Object beanInstance = doCreateBean(beanName, mbdToUse, args);`  这一行  
但是，如果 `InstantiationAwareBeanPostProcessor` 的一个实现，在方法 `postProcessBeforeInstantiation` 执行后，返回了一个非 null ，那么就不走后面的流程了 
```java
protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
    Object bean = null;
    if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
        // Make sure bean class is actually resolved at this point.
        if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
            Class<?> targetType = determineTargetType(beanName, mbd);
            if (targetType != null) {
                // 调用的 InstantiationAwareBeanPostProcessor， 依次调用，谁处理了，就返回 bean 不处理的返回null
                bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
                if (bean != null) {
                    // 创建后，如果要处理，就返回新的 bean，不处理就返回原来的bean，要是 返回了 null，那就忽略后面的 处理类了
                    bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
                }
            }
        }
        // bean 不为null，那就是处理过了
        mbd.beforeInstantiationResolved = (bean != null);
    }
    return bean;
}
``` 


这里只触发了 `BeanPostFactory` 的 `postProcessAfterInitialization` 方法 
`InstantiationAwareBeanPostProcessor` 的另外两个方法是在依赖注入的时候使用的


## DestructionAwareBeanPostProcessor
这个接口也是 `BeanPostProcessor` 的子接口，有两个方法   
```java
public interface DestructionAwareBeanPostProcessor extends BeanPostProcessor {

	void postProcessBeforeDestruction(Object bean, String beanName) throws BeansException;

	default boolean requiresDestruction(Object bean) {
		return true;
	}

}
``` 

很明显，这两个方法是 bean 销毁时候的回调。  
先说回调顺序，首先是 `@PreDestroy` 然后是 `DisposableBean` 最后是指定的 destroy-Method
为啥捏？    
直接上销毁的代码    
```java
public void destroy() {
    if (!CollectionUtils.isEmpty(this.beanPostProcessors)) {
        for (DestructionAwareBeanPostProcessor processor : this.beanPostProcessors) {
            processor.postProcessBeforeDestruction(this.bean, this.beanName);
        }
    }

    if (this.invokeDisposableBean) {
        if (logger.isDebugEnabled()) {
            logger.debug("Invoking destroy() on bean with name '" + this.beanName + "'");
        }
        try {
            if (System.getSecurityManager() != null) {
                AccessController.doPrivileged((PrivilegedExceptionAction<Object>) () -> {
                    ((DisposableBean) bean).destroy();
                    return null;
                }, acc);
            }
            else {
                ((DisposableBean) bean).destroy();
            }
        }
        catch (Throwable ex) {
            String msg = "Invocation of destroy method failed on bean with name '" + this.beanName + "'";
            if (logger.isDebugEnabled()) {
                logger.warn(msg, ex);
            }
            else {
                logger.warn(msg + ": " + ex);
            }
        }
    }

    if (this.destroyMethod != null) {
        invokeCustomDestroyMethod(this.destroyMethod);
    }
    else if (this.destroyMethodName != null) {
        Method methodToCall = determineDestroyMethod(this.destroyMethodName);
        if (methodToCall != null) {
            invokeCustomDestroyMethod(methodToCall);
        }
    }
}
``` 
很明显的一个顺序，至于销毁在哪里发生的，可以查看这个 destroy 在哪里调用的。。

## SmartInstantiationAwareBeanPostProcessor
这个接口继承 `InstantiationAwareBeanPostProcessor`， 也有3个方法

```java
public interface SmartInstantiationAwareBeanPostProcessor extends InstantiationAwareBeanPostProcessor {

    //
	@Nullable
	default Class<?> predictBeanType(Class<?> beanClass, String beanName) throws BeansException {
		return null;
	}

    // 这个方法只用来选择构造方法用的
	@Nullable
	default Constructor<?>[] determineCandidateConstructors(Class<?> beanClass, String beanName)
			throws BeansException {

		return null;
	}

    //
	default Object getEarlyBeanReference(Object bean, String beanName) throws BeansException {
		return bean;
	}

}
``` 
