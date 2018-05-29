[TOC]

# IOC 分析
从 AnnotationConfigApplicationContext 分析

目前由于springboot 快速流行
而springboot是基于注解的，因此以 AnnotationConfigApplicationContext 为源头开始分析源码。
分析的是 spring-framework 里面的 ioc 相关的内容，主要集中在 core bean context 包里面
我们先看官方的hello world

```java
public static void main(String[] args) {
  ApplicationContext context = new AnnotationConfigApplicationContext(Application.class);
  MessagePrinter printer = context.getBean(MessagePrinter.class);
  printer.printMessage();
}
``` 

AnnotationConfigApplicationContext 主要有两种构造方法，一种指定带注解的 class，一种指定要扫描的包名
```java
public AnnotationConfigApplicationContext() {
  this.reader = new AnnotatedBeanDefinitionReader(this);
  this.scanner = new ClassPathBeanDefinitionScanner(this);
}

public AnnotationConfigApplicationContext(String... basePackages) {
  this();
  scan(basePackages);
  refresh();
}

public AnnotationConfigApplicationContext(Class<?>... annotatedClasses) {
  this();
  register(annotatedClasses);
  refresh();
}
``` 
直接指定配置类或者包是组合用法，可以使用无参数构造方法，然后使用 scan 或者 register 方法，最后自己手动 refresh，同样可以启动 spring 容器
refresh 是spring启动的核心方法，后面重点看这个。

## refresh 之前的准备工作
构造 AnnotatedBeanDefinitionReader 和 ClassPathBeanDefinitionScanner

现在先看 AnnotatedBeanDefinitionReader 这个构造，因为在构造里面会添加几个系统的 BeanPostProcessor 类
1. ConfigurationClassPostProcessor
2. AutowiredAnnotationBeanPostProcessor
3. RequiredAnnotationBeanPostProcessor
4. CommonAnnotationBeanPostProcessor
  如果 jsr250 在classpath 下面，就添加这个
  什么是jsr250，就是注解 `javax.annotation.Resource` 在的jar包
5. PersistenceAnnotationBeanPostProcessor
  没用 JPA 就没有这个类
6. EventListenerMethodProcessor
7. DefaultEventListenerFactory  

ClassPathBeanDefinitionScanner 这个类只是正常初始化，然后就是 `register(annotatedClasses);` 注册，这个方法是将启动指定的配置类解析成 BeanDefinition 注入到容器中
这个 BeanDefinition 的解析在后面执行，这里只是简单的注入。

```java
public void register(Class<?>... annotatedClasses) {
  Assert.notEmpty(annotatedClasses, "At least one annotated class must be specified");
  this.reader.register(annotatedClasses);
}

public void register(Class<?>... annotatedClasses) {
  for (Class<?> annotatedClass : annotatedClasses) {
    registerBean(annotatedClass);
  }
}

public void registerBean(Class<?> annotatedClass) {
  doRegisterBean(annotatedClass, null, null, null);
}

<T> void doRegisterBean(Class<T> annotatedClass, @Nullable Supplier<T> instanceSupplier, @Nullable String name,
    @Nullable Class<? extends Annotation>[] qualifiers, BeanDefinitionCustomizer... definitionCustomizers) {

  AnnotatedGenericBeanDefinition abd = new AnnotatedGenericBeanDefinition(annotatedClass);
  // 一样，判断 Condition
  if (this.conditionEvaluator.shouldSkip(abd.getMetadata())) {
    return;
  }

  // null
  abd.setInstanceSupplier(instanceSupplier);
  // 默认 scope
  ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(abd);
  abd.setScope(scopeMetadata.getScopeName());
  // 生成 beanName
  String beanName = (name != null ? name : this.beanNameGenerator.generateBeanName(abd, this.registry));

  // 设置一些值，lazyinit primary role primary 等
  AnnotationConfigUtils.processCommonDefinitionAnnotations(abd);
  if (qualifiers != null) {
    for (Class<? extends Annotation> qualifier : qualifiers) {
      if (Primary.class == qualifier) {
        abd.setPrimary(true);
      }
      else if (Lazy.class == qualifier) {
        abd.setLazyInit(true);
      }
      else {
        abd.addQualifier(new AutowireCandidateQualifier(qualifier));
      }
    }
  }
  for (BeanDefinitionCustomizer customizer : definitionCustomizers) {
    customizer.customize(abd);
  }

  BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(abd, beanName);
  definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
  // 注册到容器
  BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, this.registry);
}

public static void registerBeanDefinition(
    BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
    throws BeanDefinitionStoreException {

  // Register bean definition under primary name.
  String beanName = definitionHolder.getBeanName();
  registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

  // Register aliases for bean name, if any.
  String[] aliases = definitionHolder.getAliases();
  if (aliases != null) {
    for (String alias : aliases) {
      registry.registerAlias(beanName, alias);
    }
  }
}
``` 

## refresh 方法
这个是核心方法，完成 spring 的启动流程，在 AbstractApplicationContext 类中定义的模版启动
```java
@Override
public void refresh() throws BeansException, IllegalStateException {
  synchronized (this.startupShutdownMonitor) {
    // Prepare this context for refreshing.
    prepareRefresh();

    // Tell the subclass to refresh the internal bean factory.
    ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

    // Prepare the bean factory for use in this context.
    prepareBeanFactory(beanFactory);

    try {
      // Allows post-processing of the bean factory in context subclasses.
      postProcessBeanFactory(beanFactory);

      // Invoke factory processors registered as beans in the context.
      invokeBeanFactoryPostProcessors(beanFactory);

      // Register bean processors that intercept bean creation.
      registerBeanPostProcessors(beanFactory);

      // Initialize message source for this context.
      initMessageSource();

      // Initialize event multicaster for this context.
      initApplicationEventMulticaster();

      // Initialize other special beans in specific context subclasses.
      onRefresh();

      // Check for listener beans and register them.
      registerListeners();

      // Instantiate all remaining (non-lazy-init) singletons.
      finishBeanFactoryInitialization(beanFactory);

      // Last step: publish corresponding event.
      finishRefresh();
    }

    catch (BeansException ex) {
      if (logger.isWarnEnabled()) {
        logger.warn("Exception encountered during context initialization - " +
            "cancelling refresh attempt: " + ex);
      }

      // Destroy already created singletons to avoid dangling resources.
      destroyBeans();

      // Reset 'active' flag.
      cancelRefresh(ex);

      // Propagate exception to caller.
      throw ex;
    }

    finally {
      // Reset common introspection caches in Spring's core, since we
      // might not ever need metadata for singleton beans anymore...
      resetCommonCaches();
    }
  }
}

``` 

refresh 方法固定了启动要做的事情，启动前先上锁  
接下来每个方法都看一下

### prepareRefresh
```java
/**
  * 设置启动时间，设置激活标记，初始化 properties sources 
  * Prepare this context for refreshing, setting its startup date and
  * active flag as well as performing any initialization of property sources.
  */
protected void prepareRefresh() {
  this.startupDate = System.currentTimeMillis();
  this.closed.set(false);
  this.active.set(true);

  if (logger.isInfoEnabled()) {
    logger.info("Refreshing " + this);
  }

  // 这里是一个空方法，留个子类扩展
  // Initialize any placeholder property sources in the context environment
  initPropertySources();

  // 检查必须的参数是否存在
  // Validate that all properties marked as required are resolvable
  // see ConfigurablePropertyResolver#setRequiredProperties
  getEnvironment().validateRequiredProperties();

  // 初始化
  // Allow for the collection of early ApplicationEvents,
  // to be published once the multicaster is available...
  this.earlyApplicationEvents = new LinkedHashSet<>();
}


``` 

这个方法做的事情很简单，就是初始化参数和参数检查

### obtainFreshBeanFactory();
`ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();`

获取 `BeanFactory`
```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
  // 子类实现的，刷新 BeanFactory
  refreshBeanFactory();
  // getBeanFactory 获取的是内部 BeanFactory 的实现，默认是 DefaultListableBeanFactory
  ConfigurableListableBeanFactory beanFactory = getBeanFactory();
  if (logger.isDebugEnabled()) {
    logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
  }
  return beanFactory;
}

// refreshBeanFactory 的两个实现，基于注解的 AnnotationConfigApplication 继承的是 GenericApplicationContext

// GenericApplicationContext 类中的实现
protected final void refreshBeanFactory() throws IllegalStateException {
  if (!this.refreshed.compareAndSet(false, true)) {
    throw new IllegalStateException(
        "GenericApplicationContext does not support multiple refresh attempts: just call 'refresh' once");
  }
  this.beanFactory.setSerializationId(getId());
}

// AbstractRefreshableApplicationContext 中的实现
protected final void refreshBeanFactory() throws BeansException {
  if (hasBeanFactory()) {
    destroyBeans();
    closeBeanFactory();
  }
  try {
    DefaultListableBeanFactory beanFactory = createBeanFactory();
    beanFactory.setSerializationId(getId());
    customizeBeanFactory(beanFactory);
    loadBeanDefinitions(beanFactory);
    synchronized (this.beanFactoryMonitor) {
      this.beanFactory = beanFactory;
    }
  }
  catch (IOException ex) {
    throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
  }
}
``` 

### prepareBeanFactory
```java
// Prepare the bean factory for use in this context.
prepareBeanFactory(beanFactory);

// BeanFactory 准备阶段
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
  // Tell the internal bean factory to use the context's class loader etc.
  // 设置类加载器
  beanFactory.setBeanClassLoader(getClassLoader());
  //设置beanFactory的表达式语言处理器，Spring3增加了表达式语言的支持，  
  //默认可以使用#{bean.xxx}的形式来调用相关属性值 
  beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
  //为beanFactory增加了一个默认的propertyEditor，这个主要是对bean的属性等设置管理的一个工具  
  beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

  // Configure the bean factory with context callbacks.
  // 添加BeanPostProcessor, 这个是用来调用实现了 Aware 接口的bean 的方法
  // 从这里就可以看到 Aware 接口就是通过 BeaPostProcessor 接口扩展出来的
  // BeaPostProcessor 接口，可以在 Bean 实例化前后做一些增强，譬如动态代理等
  // 这个类处理的是 EnvironmentAware EmbeddedValueResolverAware ResourceLoaderAware ApplicationEventPublisherAware MessageSourceAware ApplicationContextAware 这些 Aware接口
  beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));

  // 设置了几个忽略自动装配的接口  
  beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
  beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
  beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
  beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
  beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
  beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

  // BeanFactory interface not registered as resolvable type in a plain factory.
  // MessageSource registered (and found for autowiring) as a bean.
  // 设置了几个自动装配的特殊 规则  , 就是需要注入 一下接口的时候，注入的是给定的实例，不需要注册 BeanDefinition
  beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
  beanFactory.registerResolvableDependency(ResourceLoader.class, this);
  beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
  beanFactory.registerResolvableDependency(ApplicationContext.class, this);

  // Register early post-processor for detecting inner beans as ApplicationListeners.
  // 设置 Bean 处理器，用来处理内部的 ApplicationListeners
  beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

  // Detect a LoadTimeWeaver and prepare for weaving, if found.
  // //增加对AspectJ的支持 
  if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
    // 这里同样是 Aware 支持，LoadTimeWeaverAware
    beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
    // Set a temporary ClassLoader for type matching.
    beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
  }

  // Register default environment beans.
  // 注册系统 Bean，注意一下 Bean 和 BeanDefinition 不一样 BeanDefinition 会解析成 Bean，但是也可以直接添加 Bean，正常情况下 Bean 比 BeanDefinition 多
  // 因为 BeanDefinition 里面可以解析出多个 Bean
  if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
    beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
  }
  if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
    beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
  }
  if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
    beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
  }
}
``` 

通过看一个 BeanPostProcessor 的实现，看他的作用
ApplicationContextAwareProcessor
```java
@Override
@Nullable
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
看方法很简单，就是在 bean 实例化之前，如果 bean 实现了 Aware 接口，那就调用，带上需要的资源
至于 BeanPostProcessor 在哪里调用的，继续向下面看。

### postProcessBeanFactory
```java
// Allows post-processing of the bean factory in context subclasses.
postProcessBeanFactory(beanFactory);

``` 

这是一个空方法，留给子类实现，子类可以对 BeanFactory 做一些处理，添加一些Bean，BeanPostProcessor 添加忽略的接口等等
扩展这个接口一般也是spring其他模块的事情，作为spring的使用者，通常不关心这个方法

### invokeBeanFactoryPostProcessors
这个实现挺长的，不过很重要，得分析一下
```java
// Invoke factory processors registered as beans in the context.
invokeBeanFactoryPostProcessors(beanFactory);

protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
  // 功能就是调用 BeanFactoryPostProcessor 和 BeanDefinitionRegistryPostProcessor 
  // BeanFactoryPostProcessor 接口跟 BeanPostProcessor 类似，不过是 BeanFactory 初始化好了后调用实现类
  // BeanDefinitionRegistryPostProcessor 接口参数跟上面不一样，是 BeanDefinitionRegistry 上面的是 BeanFactory
  PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());

  // Detect a LoadTimeWeaver and prepare for weaving, if found in the meantime
  // (e.g. through an @Bean method registered by ConfigurationClassPostProcessor)
  if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
    beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
    beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
  }
}

public static void invokeBeanFactoryPostProcessors(
  ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {

  // Invoke BeanDefinitionRegistryPostProcessors first, if any.
  Set<String> processedBeans = new HashSet<>();

  if (beanFactory instanceof BeanDefinitionRegistry) {
    BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;
    List<BeanFactoryPostProcessor> regularPostProcessors = new LinkedList<>();
    List<BeanDefinitionRegistryPostProcessor> registryProcessors = new LinkedList<>();

    for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
      if (postProcessor instanceof BeanDefinitionRegistryPostProcessor) {
        BeanDefinitionRegistryPostProcessor registryProcessor =
            (BeanDefinitionRegistryPostProcessor) postProcessor;
        registryProcessor.postProcessBeanDefinitionRegistry(registry);
        registryProcessors.add(registryProcessor);
      }
      else {
        regularPostProcessors.add(postProcessor);
      }
    }

    // Do not initialize FactoryBeans here: We need to leave all regular beans
    // uninitialized to let the bean factory post-processors apply to them!
    // Separate between BeanDefinitionRegistryPostProcessors that implement
    // PriorityOrdered, Ordered, and the rest.
    List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<>();

    // First, invoke the BeanDefinitionRegistryPostProcessors that implement PriorityOrdered.
    String[] postProcessorNames =
        beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
    for (String ppName : postProcessorNames) {
      if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
        currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
        processedBeans.add(ppName);
      }
    }
    sortPostProcessors(currentRegistryProcessors, beanFactory);
    registryProcessors.addAll(currentRegistryProcessors);
    invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
    currentRegistryProcessors.clear();

    // Next, invoke the BeanDefinitionRegistryPostProcessors that implement Ordered.
    postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
    for (String ppName : postProcessorNames) {
      if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
        currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
        processedBeans.add(ppName);
      }
    }
    sortPostProcessors(currentRegistryProcessors, beanFactory);
    registryProcessors.addAll(currentRegistryProcessors);
    invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
    currentRegistryProcessors.clear();

    // Finally, invoke all other BeanDefinitionRegistryPostProcessors until no further ones appear.
    boolean reiterate = true;
    while (reiterate) {
      reiterate = false;
      postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
      for (String ppName : postProcessorNames) {
        if (!processedBeans.contains(ppName)) {
          currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
          processedBeans.add(ppName);
          reiterate = true;
        }
      }
      sortPostProcessors(currentRegistryProcessors, beanFactory);
      registryProcessors.addAll(currentRegistryProcessors);
      invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
      currentRegistryProcessors.clear();
    }

    // Now, invoke the postProcessBeanFactory callback of all processors handled so far.
    invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);
    invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
  }

  else {
    // Invoke factory processors registered with the context instance.
    invokeBeanFactoryPostProcessors(beanFactoryPostProcessors, beanFactory);
  }

  // Do not initialize FactoryBeans here: We need to leave all regular beans
  // uninitialized to let the bean factory post-processors apply to them!
  String[] postProcessorNames =
      beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);

  // Separate between BeanFactoryPostProcessors that implement PriorityOrdered,
  // Ordered, and the rest.
  List<BeanFactoryPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
  List<String> orderedPostProcessorNames = new ArrayList<>();
  List<String> nonOrderedPostProcessorNames = new ArrayList<>();
  for (String ppName : postProcessorNames) {
    if (processedBeans.contains(ppName)) {
      // skip - already processed in first phase above
    }
    else if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
      priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
    }
    else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
      orderedPostProcessorNames.add(ppName);
    }
    else {
      nonOrderedPostProcessorNames.add(ppName);
    }
  }

  // First, invoke the BeanFactoryPostProcessors that implement PriorityOrdered.
  sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
  invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);

  // Next, invoke the BeanFactoryPostProcessors that implement Ordered.
  List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList<>();
  for (String postProcessorName : orderedPostProcessorNames) {
    orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
  }
  sortPostProcessors(orderedPostProcessors, beanFactory);
  invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);

  // Finally, invoke all other BeanFactoryPostProcessors.
  List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList<>();
  for (String postProcessorName : nonOrderedPostProcessorNames) {
    nonOrderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
  }
  invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);

  // Clear cached merged bean definitions since the post-processors might have
  // modified the original metadata, e.g. replacing placeholders in values...
  beanFactory.clearMetadataCache();
}
``` 



### ConfigurationClassPostProcessor
看他实现的接口
1. BeanDefinitionRegistryPostProcessor
2. PriorityOrdered
3. ResourceLoaderAware
4. BeanClassLoaderAware
5. EnvironmentAware

第一个接口 BeanDefinitionRegistryPostProcessor 继承自 BeanFactoryPostProcessor
第二个接口 PriorityOrdered 继承自 Ordered，这是一个排序接口，用来标记配置类的加载顺序，值越小，优先级越高
后面3个 Aware 接口是获取 spring 内部内容的感知接口，就是一个set 方法，譬如 ResourceLoaderAware 就是一个 setResourceLoader 方法，参数 ResourceLoader，就是spring内部使用的，实现类可以获取这个实例，spring 中一些其他的 Aware 接口，功能是一样的

在 ConfigurationClassPostProcessor 实现里面，重要的是 BeanDefinitionRegistryPostProcessor 接口方法 postProcessBeanDefinitionRegistry 的实现，其他接口获取资源，都是为这个方法准备的。

```java
/**
  * Derive further bean definitions from the configuration classes in the registry.
  */
@Override
public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
  int registryId = System.identityHashCode(registry);
  if (this.registriesPostProcessed.contains(registryId)) {
    throw new IllegalStateException(
        "postProcessBeanDefinitionRegistry already called on this post-processor against " + registry);
  }
  if (this.factoriesPostProcessed.contains(registryId)) {
    throw new IllegalStateException(
        "postProcessBeanFactory already called on this post-processor against " + registry);
  }
  this.registriesPostProcessed.add(registryId);

  processConfigBeanDefinitions(registry);
}

/**
  * Build and validate a configuration model based on the registry of
  * {@link Configuration} classes.
  */
public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
  // 这个变量搜集将要处理的 BeanDefinition 
  List<BeanDefinitionHolder> configCandidates = new ArrayList<>();

  // 拿到所有 BeanDefinition 的 name
  String[] candidateNames = registry.getBeanDefinitionNames();

  for (String beanName : candidateNames) {
    BeanDefinition beanDef = registry.getBeanDefinition(beanName);
    // 如果 BeanDefinition 属性 configurationClass（前面加上 ConfigurationClassPostProcessor 的全名.）为 lite 或者 full，代表以及处理过了
    if (ConfigurationClassUtils.isFullConfigurationClass(beanDef) ||
        ConfigurationClassUtils.isLiteConfigurationClass(beanDef)) {
      if (logger.isDebugEnabled()) {
        logger.debug("Bean definition has already been processed as a configuration class: " + beanDef);
      }
    }
    // 判断是否为配置类，1. 有类注解 Configuration，标记为 full 2. 有类注解 Component，ComponentScan，Import，ImportResource 标记为 lite 3. 有 Bean 注解的方法
    else if (ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)) {
      configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));
    }
  }

  // Return immediately if no @Configuration classes were found
  if (configCandidates.isEmpty()) {
    return;
  }

  // 配置处理排序，值越小优先级越高（先处理）
  // Sort by previously determined @Order value, if applicable
  configCandidates.sort((bd1, bd2) -> {
    int i1 = ConfigurationClassUtils.getOrder(bd1.getBeanDefinition());
    int i2 = ConfigurationClassUtils.getOrder(bd2.getBeanDefinition());
    return Integer.compare(i1, i2);
  });

  // Detect any custom bean name generation strategy supplied through the enclosing application context
  // 这里可以修改 BeanNameGenerator 的实现, 系统内部用的，就是自己定义了，由于是延时注入的，首先注入的 BeanDefinition 是系统的和配置 Bean，到这里的时候自定义的 bean 还没开始处理
  SingletonBeanRegistry sbr = null;
  if (registry instanceof SingletonBeanRegistry) {
    sbr = (SingletonBeanRegistry) registry;
    if (!this.localBeanNameGeneratorSet) {
      BeanNameGenerator generator = (BeanNameGenerator) sbr.getSingleton(CONFIGURATION_BEAN_NAME_GENERATOR);
      if (generator != null) {
        this.componentScanBeanNameGenerator = generator;
        this.importBeanNameGenerator = generator;
      }
    }
  }

  // 环境
  if (this.environment == null) {
    this.environment = new StandardEnvironment();
  }

  // Parse each @Configuration class
  ConfigurationClassParser parser = new ConfigurationClassParser(
      this.metadataReaderFactory, this.problemReporter, this.environment,
      this.resourceLoader, this.componentScanBeanNameGenerator, registry);

  // 待处理的配置类
  Set<BeanDefinitionHolder> candidates = new LinkedHashSet<>(configCandidates);
  Set<ConfigurationClass> alreadyParsed = new HashSet<>(configCandidates.size());
  do {
    // 这里就开始解析了
    parser.parse(candidates);
    parser.validate();

    Set<ConfigurationClass> configClasses = new LinkedHashSet<>(parser.getConfigurationClasses());
    configClasses.removeAll(alreadyParsed);

    // Read the model and create bean definitions based on its content
    if (this.reader == null) {
      this.reader = new ConfigurationClassBeanDefinitionReader(
          registry, this.sourceExtractor, this.resourceLoader, this.environment,
          this.importBeanNameGenerator, parser.getImportRegistry());
    }
    this.reader.loadBeanDefinitions(configClasses);
    alreadyParsed.addAll(configClasses);

    candidates.clear();
    if (registry.getBeanDefinitionCount() > candidateNames.length) {
      String[] newCandidateNames = registry.getBeanDefinitionNames();
      Set<String> oldCandidateNames = new HashSet<>(Arrays.asList(candidateNames));
      Set<String> alreadyParsedClasses = new HashSet<>();
      for (ConfigurationClass configurationClass : alreadyParsed) {
        alreadyParsedClasses.add(configurationClass.getMetadata().getClassName());
      }
      for (String candidateName : newCandidateNames) {
        if (!oldCandidateNames.contains(candidateName)) {
          BeanDefinition bd = registry.getBeanDefinition(candidateName);
          if (ConfigurationClassUtils.checkConfigurationClassCandidate(bd, this.metadataReaderFactory) &&
              !alreadyParsedClasses.contains(bd.getBeanClassName())) {
            candidates.add(new BeanDefinitionHolder(bd, candidateName));
          }
        }
      }
      candidateNames = newCandidateNames;
    }
  }
  while (!candidates.isEmpty());

  // Register the ImportRegistry as a bean in order to support ImportAware @Configuration classes
  if (sbr != null && !sbr.containsSingleton(IMPORT_REGISTRY_BEAN_NAME)) {
    sbr.registerSingleton(IMPORT_REGISTRY_BEAN_NAME, parser.getImportRegistry());
  }

  if (this.metadataReaderFactory instanceof CachingMetadataReaderFactory) {
    // Clear cache in externally provided MetadataReaderFactory; this is a no-op
    // for a shared cache since it'll be cleared by the ApplicationContext.
    ((CachingMetadataReaderFactory) this.metadataReaderFactory).clearCache();
  }
}
``` 

看一下 ConfigurationClassParser 的解析过程

```java
public void parse(Set<BeanDefinitionHolder> configCandidates) {
  this.deferredImportSelectors = new LinkedList<>();

  // 遍历处理， AnnotatedBeanDefinition 代表是注解配置进来的， AbstractBeanDefinition 代表是 其他方式配置进来的，再else 就代表是扩展的了
  for (BeanDefinitionHolder holder : configCandidates) {
    BeanDefinition bd = holder.getBeanDefinition();
    try {
      if (bd instanceof AnnotatedBeanDefinition) {
        parse(((AnnotatedBeanDefinition) bd).getMetadata(), holder.getBeanName());
      }
      else if (bd instanceof AbstractBeanDefinition && ((AbstractBeanDefinition) bd).hasBeanClass()) {
        parse(((AbstractBeanDefinition) bd).getBeanClass(), holder.getBeanName());
      }
      else {
        parse(bd.getBeanClassName(), holder.getBeanName());
      }
    }
    catch (BeanDefinitionStoreException ex) {
      throw ex;
    }
    catch (Throwable ex) {
      throw new BeanDefinitionStoreException(
          "Failed to parse configuration class [" + bd.getBeanClassName() + "]", ex);
    }
  }

  processDeferredImportSelectors();
}

// 最终解析的是 ConfigurationClass
protected final void parse(@Nullable String className, String beanName) throws IOException {
  Assert.notNull(className, "No bean class name for configuration class bean definition");
  MetadataReader reader = this.metadataReaderFactory.getMetadataReader(className);
  processConfigurationClass(new ConfigurationClass(reader, beanName));
}

protected final void parse(Class<?> clazz, String beanName) throws IOException {
  processConfigurationClass(new ConfigurationClass(clazz, beanName));
}

protected final void parse(AnnotationMetadata metadata, String beanName) throws IOException {
  processConfigurationClass(new ConfigurationClass(metadata, beanName));
}

// 这里是具体的解析逻辑
protected void processConfigurationClass(ConfigurationClass configClass) throws IOException {
  // 看是不是有 Condition 条件，如果有判断条件
  if (this.conditionEvaluator.shouldSkip(configClass.getMetadata(), ConfigurationPhase.PARSE_CONFIGURATION)) {
    return;
  }

  // 如果configClass 已经处理过了
  ConfigurationClass existingClass = this.configurationClasses.get(configClass);
  if (existingClass != null) {
    if (configClass.isImported()) {
      if (existingClass.isImported()) {
        existingClass.mergeImportedBy(configClass);
      }
      // Otherwise ignore new imported config class; existing non-imported class overrides it.
      return;
    }
    else {
      // 重复定义，后一个更新前一个
      // Explicit bean definition found, probably replacing an import.
      // Let's remove the old one and go with the new one.
      this.configurationClasses.remove(configClass);
      this.knownSuperclasses.values().removeIf(configClass::equals);
    }
  }

  // Recursively process the configuration class and its superclass hierarchy.
  SourceClass sourceClass = asSourceClass(configClass);
  do {
    sourceClass = doProcessConfigurationClass(configClass, sourceClass);
  }
  while (sourceClass != null);

  this.configurationClasses.put(configClass, configClass);
}
``` 
