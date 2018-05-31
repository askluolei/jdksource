[TOC]

# spring 实现分析

从spring 的启动开始，启动始终绕不开 refresh 方法，这个是spring的核心方法，用来启动spring
```java
/**
* spring 启动的核心方法
* @throws BeansException
* @throws IllegalStateException
*/
@Override
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        // Prepare this context for refreshing.
        // 准备启动阶段，记录启动时间，准备 properties 和 验证
        prepareRefresh();

        // 获取 beanFactory  通常是 DefaultListableBeanFactory 实例，在 注解配置 和 xml 配置中的实现稍微有些不同
        // 不同总的来说就是获取一个 DefaultListableBeanFactory
        // Tell the subclass to refresh the internal bean factory.
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // 准备阶段，就是对 beanFactory 进行一些配置
        // Prepare the bean factory for use in this context.
        prepareBeanFactory(beanFactory);

        try {
            // 准备完毕后的一个方法，子类可以对 beanFactory 继续做一些修改，本类中是一个空方法
            // Allows post-processing of the bean factory in context subclasses.
            postProcessBeanFactory(beanFactory);

            // 这是一个重要的地方，对应接口为 BeanFactoryPostProcessor 以及子接口 BeanDefinitionRegistryPostProcessor
            // BeanFactoryPostProcessor 接口是可以针对 beanFactory 做一些修改，譬如添加一些 bean 之类的
            // BeanDefinitionRegistryPostProcessor 接口只是方法参数为 BeanDefinitionRegistry，可以再添加一些 BeanDefinition
            // 这个接口是一个重要的扩展接口，spring 内部 或者外部都可以通过这个接口的实现来扩展
            // 可以通过 ApplicationContext 的 addBeanFactoryPostProcessor 方法添加，或者注册 Bean，spring 就可以感知到，进行触发
            // 发生顺序很重要，先调用通过 beanFactory.add 添加的，然后调用 BeanDefinition 容器里面的
            // spring 内部的第一个调用的容器中的定义就是 ConfigurationClassPostProcessor，这个用了根据 配置类来找各个地方的 BeanDefinition ，注册到 容器中
            // 无论xml配置还是注解配置，通常有一个入口配置（springboot 中通常就是 Application），然后根据这个配置类，找到其他的配置类，根据所有配置类（找到一个解析一个）找到定义的 bean
            // Invoke factory processors registered as beans in the context.
            invokeBeanFactoryPostProcessors(beanFactory);

            // 经过上一步调用后，所有的 BeanDefinition 都已经找到并且注册到容器中类
            // 这一步就是找到容器中实现 BeanPostProcessor 的 BeanDefinition，然后添加到 beanFactory 中
            // BeanPostProcessor 这个也是一个非常重要的扩展接口，譬如spring内部各种 Aware 接口，就是有 这个接口的实现类扩展出来的
            // Register bean processors that intercept bean creation.
            registerBeanPostProcessors(beanFactory);


            // 初始化 MessageSource 接口，如果没有，就使用默认的，注册到容器里面
            // 这里需要说明一下，spring 内部有两个容器，一个是 bean 容器的，一个是 BeanDefinition 容器的，实例是根据 BeanDefinition 的定义生成的
            // Initialize message source for this context.
            initMessageSource();

            // 初始化 ApplicationEventMulticaster 接口，如果没有，就使用默认的，注册到容器里面
            // Initialize event multicaster for this context.
            initApplicationEventMulticaster();

            // 这是一个空方法，子类可以实现，可以在 bean 容器里面添加一些东西
            // Initialize other special beans in specific context subclasses.
            onRefresh();

            // 这里是注册 ApplicationListener ，一些是通过 beanFactory 注册的，一些是用户自定义的 Bean ，然后发布 early 事件
            // Check for listener beans and register them.
            registerListeners();

            // 这个方法里面将实例化所有非延时加载的单例 bean
            // 实例化就是调用 getBean 方法
            // 这里将触发 BeanPostProcessor 
            // Instantiate all remaining (non-lazy-init) singletons.
            finishBeanFactoryInitialization(beanFactory);

            // 这里是最后一步，清空 resource缓存，注册生命周期处理bean，然后获取所有 周期接口的实现启动，发布启动事件，注册到 JMX
            // Last step: publish corresponding event.
            finishRefresh();
        }

        catch (BeansException ex) {
            if (logger.isWarnEnabled()) {
                logger.warn("Exception encountered during context initialization - " +
                        "cancelling refresh attempt: " + ex);
            }

            // 如果启动出现异常，就销毁所有已经创建的实例
            // Destroy already created singletons to avoid dangling resources.
            destroyBeans();

            // 取消refresh
            // Reset 'active' flag.
            cancelRefresh(ex);

            // Propagate exception to caller.
            throw ex;
        }

        finally {
            // 最后一步，清空所有缓存
            // Reset common introspection caches in Spring's core, since we
            // might not ever need metadata for singleton beans anymore...
            resetCommonCaches();
        }
    }
}
``` 

这个方法固定住了 spring 的启动步骤，已经预留了扩展空方法（内部扩展用），和扩展接口（外部扩展用）
譬如不同环境下的 ApplicationContext 可以重写空方法来适应直接的环境（譬如web），spring的使用者，或者第三方框架集成到spirng，则可以通过扩展接口。
整个流程并不复杂，复杂的地方在可以通过扩展接口来修改内部的一些内容，spring 内部也是实现这些接口来进行扩展的 BeanFactoryPostProcessor BeanDefinitionRegistryPostProcessor 这两个接口在BeanFactory 准备好后进行修改，可以注册 bean 或者 BeanDefinition，譬如注解配置，主要找到主配置如果，就可以根据主配置上的信息寻找到各个地方的 BeanDefinition ，注册到 BeanFactory 中
BeanPostProcessor 接口，可以在 Bean 实例化前后进行调用，spring 内部扩展出来的各种 Aware 接口，就根据这个接口的实现扩展出来的

我们根据启动流程，来完整分析一下spirng的启动细节，先从注解配置开始看

## AnnotationConfigApplicationContext   

### refresh 之前
使用注解配置启动，最简单的启动代码如下
```java
@Configuration
@ComponentScan
public class Application {

    @Bean
    MessageService mockMessageService() {
        return () -> "Hello World!";
    }

    @Bean
    public BeanFactoryPostProcessor beanFactoryPostProcessor() {
        return new BeanFactoryPostProcessor() {
            @Override
            public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
                System.out.println(" = = = = = = = = = = == = =  = ");
            }
        };
    }

    public static void main(String[] args) {
        ApplicationContext context = new AnnotationConfigApplicationContext(Application.class);
        MessagePrinter printer = context.getBean(MessagePrinter.class);
        printer.printMessage();
        System.out.println("================= beanDefinition names ===========");
        for (String name : context.getBeanDefinitionNames()) {
            System.out.println(name);
        }
        System.out.println("============= bean names =============");
        Iterator<String> beanNamesIterator = ((AnnotationConfigApplicationContext) context).getBeanFactory().getBeanNamesIterator();
        while (beanNamesIterator.hasNext()) {
            System.out.println(beanNamesIterator.next());
        }
    }
}
``` 

通常指定一个或者多个配置类，或者指定要扫描的包,最后调用的构造方法为
```java
// 默认构造
public AnnotationConfigApplicationContext() {
    // 注意，这个里面在 beanFactory 中添加类一些 BeanFactoryPostProcessor
    this.reader = new AnnotatedBeanDefinitionReader(this);
    this.scanner = new ClassPathBeanDefinitionScanner(this);
}
// 指定配置类
public AnnotationConfigApplicationContext(Class<?>... annotatedClasses) {
    this();
    register(annotatedClasses);
    refresh();
}

// 指定包扫描
public AnnotationConfigApplicationContext(String... basePackages) {
    this();
    scan(basePackages);
    refresh();
}
``` 

在构造 reader 的时候回添加几个 BeanFactoryPostProcessor 的实现，这里就不贴调用过程，直接贴最后调用注册的地方
```java
public static Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(
        BeanDefinitionRegistry registry, @Nullable Object source) {

    // 通常来说 ApplicationContext 内部的 beanFactory 是 DefaultListableBeanFactory 实现
    DefaultListableBeanFactory beanFactory = unwrapDefaultListableBeanFactory(registry);
    if (beanFactory != null) {
        // 设置依赖排序实现
        if (!(beanFactory.getDependencyComparator() instanceof AnnotationAwareOrderComparator)) {
            beanFactory.setDependencyComparator(AnnotationAwareOrderComparator.INSTANCE);
        }
        // 设置注入候选解析
        if (!(beanFactory.getAutowireCandidateResolver() instanceof ContextAnnotationAutowireCandidateResolver)) {
            beanFactory.setAutowireCandidateResolver(new ContextAnnotationAutowireCandidateResolver());
        }
    }

    // 下面添加一些 BeanDefinition 到 registry
    // 注意，这里用的是 LinkedHashSet，是有顺序的
    Set<BeanDefinitionHolder> beanDefs = new LinkedHashSet<>(4);

    // 添加 ConfigurationClassPostProcessor，这个类第一个添加，实现 BeanDefinitionRegistryPostProcessor，功能就是根据配置类，解析 BeanDefinition
    if (!registry.containsBeanDefinition(CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME)) {
        RootBeanDefinition def = new RootBeanDefinition(ConfigurationClassPostProcessor.class);
        def.setSource(source);
        beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME));
    }

    // 添加 AutowiredAnnotationBeanPostProcessor，功能后面分析
    if (!registry.containsBeanDefinition(AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
        RootBeanDefinition def = new RootBeanDefinition(AutowiredAnnotationBeanPostProcessor.class);
        def.setSource(source);
        beanDefs.add(registerPostProcessor(registry, def, AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
    }

    // 添加 RequiredAnnotationBeanPostProcessor，功能后面分析
    if (!registry.containsBeanDefinition(REQUIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
        RootBeanDefinition def = new RootBeanDefinition(RequiredAnnotationBeanPostProcessor.class);
        def.setSource(source);
        beanDefs.add(registerPostProcessor(registry, def, REQUIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
    }

    // 只有 jsr250 jar在classpath下面的时候才添加 CommonAnnotationBeanPostProcessor，什么是 jsr250？就是 @Resource 注解标记定义 bean
    // Check for JSR-250 support, and if present add the CommonAnnotationBeanPostProcessor.
    if (jsr250Present && !registry.containsBeanDefinition(COMMON_ANNOTATION_PROCESSOR_BEAN_NAME)) {
        RootBeanDefinition def = new RootBeanDefinition(CommonAnnotationBeanPostProcessor.class);
        def.setSource(source);
        beanDefs.add(registerPostProcessor(registry, def, COMMON_ANNOTATION_PROCESSOR_BEAN_NAME));
    }

    // 如果jpa在classpath下，添加 org.springframework.orm.jpa.support.PersistenceAnnotationBeanPostProcessor
    // Check for JPA support, and if present add the PersistenceAnnotationBeanPostProcessor.
    if (jpaPresent && !registry.containsBeanDefinition(PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME)) {
        RootBeanDefinition def = new RootBeanDefinition();
        try {
            def.setBeanClass(ClassUtils.forName(PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME,
                    AnnotationConfigUtils.class.getClassLoader()));
        }
        catch (ClassNotFoundException ex) {
            throw new IllegalStateException(
                    "Cannot load optional framework class: " + PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME, ex);
        }
        def.setSource(source);
        beanDefs.add(registerPostProcessor(registry, def, PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME));
    }

    // 添加 EventListenerMethodProcessor
    if (!registry.containsBeanDefinition(EVENT_LISTENER_PROCESSOR_BEAN_NAME)) {
        RootBeanDefinition def = new RootBeanDefinition(EventListenerMethodProcessor.class);
        def.setSource(source);
        beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_PROCESSOR_BEAN_NAME));
    }
    
    //  添加 DefaultEventListenerFactory
    if (!registry.containsBeanDefinition(EVENT_LISTENER_FACTORY_BEAN_NAME)) {
        RootBeanDefinition def = new RootBeanDefinition(DefaultEventListenerFactory.class);
        def.setSource(source);
        beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_FACTORY_BEAN_NAME));
    }

    return beanDefs;
}
``` 

上面添加的 BeanFactoryPostProcessor 实现，后面再分析，这些实现很重要。

然后就是 register 和 scan 方法了    
registry 就是注册 BeanDefinition
```java
<T> void doRegisterBean(Class<T> annotatedClass, @Nullable Supplier<T> instanceSupplier, @Nullable String name,
			@Nullable Class<? extends Annotation>[] qualifiers, BeanDefinitionCustomizer... definitionCustomizers) {
    // 定义 BeanDefinition
    AnnotatedGenericBeanDefinition abd = new AnnotatedGenericBeanDefinition(annotatedClass);
    // 判断是否要跳过，就是看有没有 @Conditional 注解，如果有，满足不满足条件
    if (this.conditionEvaluator.shouldSkip(abd.getMetadata())) {
        return;
    }

    // 提示提供，生成对象的方法
    abd.setInstanceSupplier(instanceSupplier);
    // 默认是 singletons
    ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(abd);
    abd.setScope(scopeMetadata.getScopeName());

    // 生成 bean name
    String beanName = (name != null ? name : this.beanNameGenerator.generateBeanName(abd, this.registry));

    // 添加一些基本信息，lazyInit，primary，dependsOn，role，description 等
    AnnotationConfigUtils.processCommonDefinitionAnnotations(abd);

    // 标记的注解
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

    // 看有没有处理 BeanDefinition 的逻辑
    for (BeanDefinitionCustomizer customizer : definitionCustomizers) {
        customizer.customize(abd);
    }

    // 用 Holder 包裹
    BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(abd, beanName);
    // 看是否需要 Scope 代理处理，如果需要，beanClass 就换成 ScopedProxyFactoryBean，装饰之前的 BeanDefinition
    definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
    // 注册 BeanDefinition
    BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, this.registry);
}
``` 

注意到，这个只是注册参数的配置类，还没开始解析。


再来看看 scan 方法
```java
public int scan(String... basePackages) {
    int beanCountAtScanStart = this.registry.getBeanDefinitionCount();
    
    doScan(basePackages);

    // 这里注册 BeanFactoryPostProcessor 跟 new reader 里面注册调用的是同一个方法
    // Register annotation config processors, if necessary.
    if (this.includeAnnotationConfig) {
        AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
    }

    return (this.registry.getBeanDefinitionCount() - beanCountAtScanStart);
}

protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
    Assert.notEmpty(basePackages, "At least one base package must be specified");
    Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();
    for (String basePackage : basePackages) {
        // 扫描包，或者 BeanDefinition
        Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
        for (BeanDefinition candidate : candidates) {
            // BeanDefinition 添加基本信息
            ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
            candidate.setScope(scopeMetadata.getScopeName());
            String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
            if (candidate instanceof AbstractBeanDefinition) {
                postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
            }
            if (candidate instanceof AnnotatedBeanDefinition) {
                AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
            }
            // 如果有重复定义，看看可不可以覆盖
            if (checkCandidate(beanName, candidate)) {
                BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
                // 是否需要 Scope 包裹
                definitionHolder =
                        AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
                beanDefinitions.add(definitionHolder);
                // 注册
                registerBeanDefinition(definitionHolder, this.registry);
            }
        }
    }
    return beanDefinitions;
}
``` 

上面只有简单的注释，过于细节的没有贴代码，先要有一个 spring 启动的整体印象，后面再来填充细节
讲到这里，其实就是在讲 refresh 方法之前，做了什么
总结的说，就是先注册了一些 BeanDefinition，其中一些 BeanDefinition 是辅助启动的（BeanFactoryPostProcessor）
我们先把这个辅助实现单独列出来，后面单独分析
1. ConfigurationClassPostProcessor
2. AutowiredAnnotationBeanPostProcessor 这是一个 BeanPostProcessor
3. RequiredAnnotationBeanPostProcessor 这是一个 BeanPostProcessor
4. CommonAnnotationBeanPostProcessor 可选 jsr250
5. PersistenceAnnotationBeanPostProcessor 可选 jpa
6. EventListenerMethodProcessor 这是一个 SmartInitializingSingleton

### refresh 方法
```java
/**
* spring 启动的核心方法
* @throws BeansException
* @throws IllegalStateException
*/
@Override
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        // Prepare this context for refreshing.
        // 准备启动阶段，记录启动时间，准备 properties 和 验证
        prepareRefresh();

        // 获取 beanFactory  通常是 DefaultListableBeanFactory 实例，在 注解配置 和 xml 配置中的实现稍微有些不同
        // 不同总的来说就是获取一个 DefaultListableBeanFactory
        // Tell the subclass to refresh the internal bean factory.
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // 准备阶段，就是对 beanFactory 进行一些配置
        // Prepare the bean factory for use in this context.
        prepareBeanFactory(beanFactory);

        try {
            // 准备完毕后的一个方法，子类可以对 beanFactory 继续做一些修改，本类中是一个空方法
            // Allows post-processing of the bean factory in context subclasses.
            postProcessBeanFactory(beanFactory);

            // 这是一个重要的地方，对应接口为 BeanFactoryPostProcessor 以及子接口 BeanDefinitionRegistryPostProcessor
            // BeanFactoryPostProcessor 接口是可以针对 beanFactory 做一些修改，譬如添加一些 bean 之类的
            // BeanDefinitionRegistryPostProcessor 接口只是方法参数为 BeanDefinitionRegistry，可以再添加一些 BeanDefinition
            // 这个接口是一个重要的扩展接口，spring 内部 或者外部都可以通过这个接口的实现来扩展
            // 可以通过 ApplicationContext 的 addBeanFactoryPostProcessor 方法添加，或者注册 Bean，spring 就可以感知到，进行触发
            // 发生顺序很重要，先调用通过 beanFactory.add 添加的，然后调用 BeanDefinition 容器里面的
            // spring 内部的第一个调用的容器中的定义就是 ConfigurationClassPostProcessor，这个用了根据 配置类来找各个地方的 BeanDefinition ，注册到 容器中
            // 无论xml配置还是注解配置，通常有一个入口配置（springboot 中通常就是 Application），然后根据这个配置类，找到其他的配置类，根据所有配置类（找到一个解析一个）找到定义的 bean
            // Invoke factory processors registered as beans in the context.
            invokeBeanFactoryPostProcessors(beanFactory);

            // 经过上一步调用后，所有的 BeanDefinition 都已经找到并且注册到容器中类
            // 这一步就是找到容器中实现 BeanPostProcessor 的 BeanDefinition，然后添加到 beanFactory 中
            // BeanPostProcessor 这个也是一个非常重要的扩展接口，譬如spring内部各种 Aware 接口，就是有 这个接口的实现类扩展出来的
            // Register bean processors that intercept bean creation.
            registerBeanPostProcessors(beanFactory);


            // 初始化 MessageSource 接口，如果没有，就使用默认的，注册到容器里面
            // 这里需要说明一下，spring 内部有两个容器，一个是 bean 容器的，一个是 BeanDefinition 容器的，实例是根据 BeanDefinition 的定义生成的
            // Initialize message source for this context.
            initMessageSource();

            // 初始化 ApplicationEventMulticaster 接口，如果没有，就使用默认的，注册到容器里面
            // Initialize event multicaster for this context.
            initApplicationEventMulticaster();

            // 这是一个空方法，子类可以实现，可以在 bean 容器里面添加一些东西
            // Initialize other special beans in specific context subclasses.
            onRefresh();

            // 这里是注册 ApplicationListener ，一些是通过 beanFactory 注册的，一些是用户自定义的 Bean ，然后发布 early 事件
            // Check for listener beans and register them.
            registerListeners();

            // 这个方法里面将实例化所有非延时加载的单例 bean
            // 实例化就是调用 getBean 方法
            // 这里将触发 BeanPostProcessor 
            // Instantiate all remaining (non-lazy-init) singletons.
            finishBeanFactoryInitialization(beanFactory);

            // 这里是最后一步，清空 resource缓存，注册生命周期处理bean，然后获取所有 周期接口的实现启动，发布启动事件，注册到 JMX
            // Last step: publish corresponding event.
            finishRefresh();
        }

        catch (BeansException ex) {
            if (logger.isWarnEnabled()) {
                logger.warn("Exception encountered during context initialization - " +
                        "cancelling refresh attempt: " + ex);
            }

            // 如果启动出现异常，就销毁所有已经创建的实例
            // Destroy already created singletons to avoid dangling resources.
            destroyBeans();

            // 取消refresh
            // Reset 'active' flag.
            cancelRefresh(ex);

            // Propagate exception to caller.
            throw ex;
        }

        finally {
            // 最后一步，清空所有缓存
            // Reset common introspection caches in Spring's core, since we
            // might not ever need metadata for singleton beans anymore...
            resetCommonCaches();
        }
    }
}
``` 

这里吧 refresh 的代码再贴一遍，这里我们具体分析一下启动过程

#### prepareRefresh
准备阶段，主要是准备一些配置信息
```java
protected void prepareRefresh() {
    // 启动时间
    this.startupDate = System.currentTimeMillis();
    // 设置开启状态
    this.closed.set(false);
    this.active.set(true);

    if (logger.isInfoEnabled()) {
        logger.info("Refreshing " + this);
    }

    // 初始化 properties 默认只有 systemProperties 和 environment ，子类可以实现添加，这里是空方法
    // Initialize any placeholder property sources in the context environment
    initPropertySources();

    // 校验必须的属性是否存在
    // Validate that all properties marked as required are resolvable
    // see ConfigurablePropertyResolver#setRequiredProperties
    getEnvironment().validateRequiredProperties();

    // 用来收集 early 事件
    // Allow for the collection of early ApplicationEvents,
    // to be published once the multicaster is available...
    this.earlyApplicationEvents = new LinkedHashSet<>();
}
``` 

#### obtainFreshBeanFactory
获取 beanFactory，就是new出来的 DefaultListableBeanFactory，在 注解ApplicationContext 和 xmlApplicationContext 中不一样,我们先只看注解的
```java
ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
    // 刷新，注解里面还是直接返回，xml里面销毁里面的 bean，重新构建的
    refreshBeanFactory();
    // 获取刷新完的 beanFactory
    ConfigurableListableBeanFactory beanFactory = getBeanFactory();
    if (logger.isDebugEnabled()) {
        logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
    }
    return beanFactory;
}

protected final void refreshBeanFactory() throws IllegalStateException {
    if (!this.refreshed.compareAndSet(false, true)) {
        throw new IllegalStateException(
                "GenericApplicationContext does not support multiple refresh attempts: just call 'refresh' once");
    }
    this.beanFactory.setSerializationId(getId());
}

public final ConfigurableListableBeanFactory getBeanFactory() {
    return this.beanFactory;
}
``` 

这个方法里面也没什么内容，我们要知道 beanFactory 是一个 DefaultListableBeanFactory 对象就行了

#### prepareBeanFactory
准备 BeanFactory
```java
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
    // 添加 BeanPostProcessor, 这个是用来调用实现了 Aware 接口的bean 的方法
    // 从这里就可以看到 Aware 接口就是通过 BeaPostProcessor 接口扩展出来的
    // BeaPostProcessor 接口，可以在 Bean 实例化前后做一些增强，譬如动态代理等
    // 这个类处理的是 EnvironmentAware EmbeddedValueResolverAware ResourceLoaderAware ApplicationEventPublisherAware MessageSourceAware ApplicationContextAware 这些 Aware接口
    beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));

    // 这里忽略的接口，就是 ApplicationContextAwareProcessor 里面处理的接口
    beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
    beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
    beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
    beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

    // BeanFactory interface not registered as resolvable type in a plain factory.
    // MessageSource registered (and found for autowiring) as a bean.
    // 这里添加处理依赖时候的映射，也就是说，当有其他类依赖 key 类型（需要注入，跟 dependsOn 不一样）,注入的就是 value
    beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
    beanFactory.registerResolvableDependency(ResourceLoader.class, this);
    beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
    beanFactory.registerResolvableDependency(ApplicationContext.class, this);

    // 添加 BeanPostProcessor ，这个类 ApplicationListenerDetector 是用来找到 bean 里面实现了 ApplicationListener 接口的实例，
    // 添加到 beanFactory 的事件监听列表里面，用来接收事件的
    // Register early post-processor for detecting inner beans as ApplicationListeners.
    beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

    // 这里实际是是添加一个 Aware，LoadTimeWeaverAware，其中要set 的是 LOAD_TIME_WEAVER_BEAN_NAME 对应的bean
    // 如果容器中有这个bean，那么 LoadTimeWeaverAware 接口才处理
    // Detect a LoadTimeWeaver and prepare for weaving, if found.
    if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
        // 添加 BeanPostProcessor  
        beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
        // Set a temporary ClassLoader for type matching.
        beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
    }

    // 添加一些环境相关的 bean
    // environment， systemProperties，systemEnvironment
    // Register default environment beans.
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

准备节点，就是配置一下 BeanFactory

#### postProcessBeanFactory
这是一个空方法，子类可以重写，对已经准备好的 BeanFactory 再做一些修改,譬如web里面的扩展
```java
protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    if (this.servletContext != null) {
        beanFactory.addBeanPostProcessor(new ServletContextAwareProcessor(this.servletContext));
        beanFactory.ignoreDependencyInterface(ServletContextAware.class);
    }
    WebApplicationContextUtils.registerWebApplicationScopes(beanFactory, this.servletContext);
    WebApplicationContextUtils.registerEnvironmentBeans(beanFactory, this.servletContext);
}
``` 
代码贴出来只是说明这个方法使用的时机，这里的代码不做讲解

#### invokeBeanFactoryPostProcessors    
这是一个重要的方法，调用 BeanFactoryProcessor 接口实现类，包括系统自己注册的，和用户自定义的。
肯定是有顺序的，因为这个时候，用户自定义的 Bean 有点还没读取，经过这个方法，调用了系统内部的实现后，所有的 BeanDefinition 将读取完毕
BeanFactoryProcessor 的系统实现后面一一再看，这里只看调用的逻辑
```java
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
    // 具体的调用在这里，getBeanFactoryPostProcessors() 获取的是 applicationContext 上面直接添加的 BeanFactoryProcessor
    // 记住，BeanFactoryPostProcessor 是在 ApplicationContext 上面添加的，因为 ApplicationContext 持有 BeanFactory，完成 BeanFactory 的启动
    // 而 BeanPostProcessor 是添加在 BeanFactory 上的，因为 bean 有 BeanFactory 直接管理
    // 如果是分步骤启动的（直接new之后，经过处理再调用refresh），可以在启动之前添加 BeanFactoryProcessor， 这个优先级是最高的
    PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());
    // 这个已经在 prepare 里面处理过了
    // Detect a LoadTimeWeaver and prepare for weaving, if found in the meantime
    // (e.g. through an @Bean method registered by ConfigurationClassPostProcessor)
    if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
        beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
        beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
    }
}

public static void invokeBeanFactoryPostProcessors(
        ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {

    // 这里记录已经调用的 BeanDefinitionRegistryPostProcessor 的 beanName
    // Invoke BeanDefinitionRegistryPostProcessors first, if any.
    Set<String> processedBeans = new HashSet<>();

    // 首先调用的是 BeanFactoryProcessor 的子接口 BeanDefinitionRegistryPostProcessor
    // BeanDefinitionRegistryPostProcessor 这个接口，代表 BeanDefinitionRegistry 准备好了，接下来可以注册 BeanDefinition 了和做一些修改

    // beanFactory 是 DefaultListableBeanFactory ，是实现类 BeanDefinitionRegistry 接口的
    if (beanFactory instanceof BeanDefinitionRegistry) {
        BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;
        // 收集 BeanFactoryProcessor
        List<BeanFactoryPostProcessor> regularPostProcessors = new LinkedList<>();
        // 收集处理过的 BeanDefinitionRegistryPostProcessor
        List<BeanDefinitionRegistryPostProcessor> registryProcessors = new LinkedList<>();

        // 首先处理的是参数传过来的，也就是直接添加在 beanFactory 上的
        for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
            // 首先处理 BeanDefinitionRegistryPostProcessor 的实现
            if (postProcessor instanceof BeanDefinitionRegistryPostProcessor) {
                BeanDefinitionRegistryPostProcessor registryProcessor =
                        (BeanDefinitionRegistryPostProcessor) postProcessor;
                // 这里直接调用
                registryProcessor.postProcessBeanDefinitionRegistry(registry);
                // 收集 BeanDefinitionRegistryPostProcessor
                registryProcessors.add(registryProcessor);
            }
            else {
                // 收集 BeanFactoryProcessor
                regularPostProcessors.add(postProcessor);
            }
        }

        // Do not initialize FactoryBeans here: We need to leave all regular beans
        // uninitialized to let the bean factory post-processors apply to them!
        // Separate between BeanDefinitionRegistryPostProcessors that implement
        // PriorityOrdered, Ordered, and the rest.
        // 这里收集将要执行的 BeanDefinitionRegistryPostProcessor
        List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<>();

        // 这里第一个要处理的 BeanDefinitionRegistryPostProcessor 就是 ConfigurationClassPostProcessor
        // 系统内建的 实现 PriorityOrdered 接口优先级最高，然后 ConfigurationClassPostProcessor 会注册自定义的 BeanDefinitionRegistryPostProcessor BeanFactoryPostProcessor 到 BeanFactory
        // 然后，下面找 BeanDefinitionRegistryPostProcessor 实现的时候，自定义的就可以看到然后被调用了
        // First, invoke the BeanDefinitionRegistryPostProcessors that implement PriorityOrdered.
        String[] postProcessorNames =
                beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
        // 先处理 BeanDefinitionRegistryPostProcessor 和 PriorityOrdered
        for (String ppName : postProcessorNames) {
            // PriorityOrdered 先处理
            if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
                currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                processedBeans.add(ppName);
            }
        }
        // 排序
        sortPostProcessors(currentRegistryProcessors, beanFactory);
        registryProcessors.addAll(currentRegistryProcessors);
        // 调用
        invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
        // 清空
        currentRegistryProcessors.clear();

        // 再处理 BeanDefinitionRegistryPostProcessor 和 Ordered
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

        // 最后处理 BeanDefinitionRegistryPostProcessor 不排序的
        // 为啥是 while，因为 BeanDefinitionRegistryPostProcessor 的作用就是注册 BeanDefinition 的，可能注册的 BeanDefinition 有包含了 BeanDefinitionRegistryPostProcessor
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
        // 现在，调用 BeanFactoryProcessor，先调用 BeanDefinitionRegistryPostProcessor 接口的（是子接口），然后调用只是实现 BeanFactoryProcessor 的
        // Now, invoke the postProcessBeanFactory callback of all processors handled so far.
        // registryProcessors 里面包含 参数传进来的和 容器里的
        invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);
        // regularPostProcessors 只包含参数里面传进来的
        invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
    }

    else {
        // 如果 beanFactory 不是 BeanDefinitionRegistry 类型的，那就只需要调用 BeanFactoryProcessor 了
        // Invoke factory processors registered with the context instance.
        invokeBeanFactoryPostProcessors(beanFactoryPostProcessors, beanFactory);
    }

    // 下面就是调用 容器里面的 BeanFactoryProcessor，顺序一样的 PriorityOrdered -》 Ordered -》 无序
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
        // 处理过的跳过
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

    // 先处理 PriorityOrdered 接口的，排序，调用
    // First, invoke the BeanFactoryPostProcessors that implement PriorityOrdered.
    sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
    invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);

    // 在处理 Ordered 接口的
    // Next, invoke the BeanFactoryPostProcessors that implement Ordered.
    List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList<>();
    for (String postProcessorName : orderedPostProcessorNames) {
        orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
    }
    sortPostProcessors(orderedPostProcessors, beanFactory);
    invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);

    // 最后处理没排序的
    // Finally, invoke all other BeanFactoryPostProcessors.
    List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList<>();
    for (String postProcessorName : nonOrderedPostProcessorNames) {
        nonOrderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
    }
    invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);

    // 经过上面的处理后， BeanDefinition 可能修改了，清理缓存
    // Clear cached merged bean definitions since the post-processors might have
    // modified the original metadata, e.g. replacing placeholders in values...
    beanFactory.clearMetadataCache();
}
``` 

经过上面的调用 BeanDefinition ，该注册的注册，该修改的修改
记住，BeanFactoryPostProcessor 是在 ApplicationContext 上面添加的，因为 ApplicationContext 持有 BeanFactory，完成 BeanFactory 的启动
而 BeanPostProcessor 是添加在 BeanFactory 上的，因为 bean 有 BeanFactory 直接管理

#### initMessageSource
这个方法就是初始化 MessageSource，具体作用，再分析
```java
protected void initMessageSource() {
    ConfigurableListableBeanFactory beanFactory = getBeanFactory();
    // 如果已经有了，那就使用已经有的，设置一下 parent
    if (beanFactory.containsLocalBean(MESSAGE_SOURCE_BEAN_NAME)) {
        this.messageSource = beanFactory.getBean(MESSAGE_SOURCE_BEAN_NAME, MessageSource.class);
        // Make MessageSource aware of parent MessageSource.
        if (this.parent != null && this.messageSource instanceof HierarchicalMessageSource) {
            HierarchicalMessageSource hms = (HierarchicalMessageSource) this.messageSource;
            if (hms.getParentMessageSource() == null) {
                // Only set parent context as parent MessageSource if no parent MessageSource
                // registered already.
                hms.setParentMessageSource(getInternalParentMessageSource());
            }
        }
        if (logger.isDebugEnabled()) {
            logger.debug("Using MessageSource [" + this.messageSource + "]");
        }
    }
    // 如果没有，那就使用默认的 DelegatingMessageSource
    else {
        // Use empty MessageSource to be able to accept getMessage calls.
        DelegatingMessageSource dms = new DelegatingMessageSource();
        dms.setParentMessageSource(getInternalParentMessageSource());
        this.messageSource = dms;
        beanFactory.registerSingleton(MESSAGE_SOURCE_BEAN_NAME, this.messageSource);
        if (logger.isDebugEnabled()) {
            logger.debug("Unable to locate MessageSource with name '" + MESSAGE_SOURCE_BEAN_NAME +
                    "': using default [" + this.messageSource + "]");
        }
    }
}
``` 
继续看后面，再看看这个类是怎么用的

#### initApplicationEventMulticaster
这个就是初始化事件管理类，事件和事件监听对象都是在这个上面
```java
protected void initApplicationEventMulticaster() {
    ConfigurableListableBeanFactory beanFactory = getBeanFactory();
    // 一样的，如果有，就用有的，如果没有，就用默认的 SimpleApplicationEventMulticaster
    if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
        this.applicationEventMulticaster =
                beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
        if (logger.isDebugEnabled()) {
            logger.debug("Using ApplicationEventMulticaster [" + this.applicationEventMulticaster + "]");
        }
    }
    else {
        this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
        beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
        if (logger.isDebugEnabled()) {
            logger.debug("Unable to locate ApplicationEventMulticaster with name '" +
                    APPLICATION_EVENT_MULTICASTER_BEAN_NAME +
                    "': using default [" + this.applicationEventMulticaster + "]");
        }
    }
}
``` 

#### onRefresh
这是一个空方法，容器内部的 bean 还没开始实例化，这个时候，还可以注册一些 bean之类的操作
```java
protected void onRefresh() throws BeansException {
    // For subclasses: do nothing by default.
}
``` 

#### finishBeanFactoryInitialization
这个方法就是开始实例化 bean 了，所有的非延时加载的singletons的都要实例化
实例化是调用 getBean 完成的
```java
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
    // 初始化 ConversionService，这是一个类型转换接口
    // Initialize conversion service for this context.
    if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
            beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
        beanFactory.setConversionService(
                beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
    }

    // 注册默认的 value 解析器
    // Register a default embedded value resolver if no bean post-processor
    // (such as a PropertyPlaceholderConfigurer bean) registered any before:
    // at this point, primarily for resolution in annotation attribute values.
    if (!beanFactory.hasEmbeddedValueResolver()) {
        beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
    }

    // 先初始化 LoadTimeWeaverAware
    // Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
    String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
    for (String weaverAwareName : weaverAwareNames) {
        getBean(weaverAwareName);
    }

    // 停止使用临时类加载器
    // Stop using the temporary ClassLoader for type matching.
    beanFactory.setTempClassLoader(null);

    // 缓存配置，不希望再有变化类，就是这个时候，不要再注册 BeanDefinition 了
    // Allow for caching all bean definition metadata, not expecting further changes.
    beanFactory.freezeConfiguration();

    // 开始实例化 singletons 类型的 bean 了
    // Instantiate all remaining (non-lazy-init) singletons.
    beanFactory.preInstantiateSingletons();
}
``` 
preInstantiateSingletons 这个方法是 DefaultListableBeanFactory 内部实例化的过程，这里先不说明，后面再看

#### finishRefresh
结束启动
```java
protected void finishRefresh() {
    // 清理 resource 缓存
    // Clear context-level resource caches (such as ASM metadata from scanning).
    clearResourceCaches();

    // 初始化 LifecycleProcessor 这是 Lifecycle 接口的管理器
    // 有定义就用定义的，没有就用默认的，挂在 context 上
    // Initialize lifecycle processor for this context.
    initLifecycleProcessor();

    // 启动 Lifecycle
    // Propagate refresh to lifecycle processor first.
    getLifecycleProcessor().onRefresh();

    // 发布 ContextRefreshedEvent 事件
    // Publish the final event.
    publishEvent(new ContextRefreshedEvent(this));

    // 注册 JMX 
    // Participate in LiveBeansView MBean, if active.
    LiveBeansView.registerApplicationContext(this);
}
``` 

#### resetCommonCaches
```java
protected void resetCommonCaches() {
    ReflectionUtils.clearCache();
    AnnotationUtils.clearCache();
    ResolvableType.clearCache();
    CachedIntrospectionResults.clearClassLoader(getClassLoader());
}
``` 
清理使用到的缓存
到这里，正常启动流程就完了
如果启动异常的话，是调用
```java
// 如果启动出现异常，就销毁所有已经创建的实例
// Destroy already created singletons to avoid dangling resources.
destroyBeans();

// 取消refresh
// Reset 'active' flag.
cancelRefresh(ex);
``` 

到这里，spring 启动的整个启动流程就大致过了一遍了。
后面要看的，就是 BeanDefinition 是在哪里注册的，那些扩展接口在干什么
bean 的实例化流程细节，等待下回分析