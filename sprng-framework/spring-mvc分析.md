[TOC]   

# spring-webmvc 分析

## ApplicationContextInitializer
1. class org.springframework.boot.context.config.DelegatingApplicationContextInitializer
2. class org.springframework.boot.context.ContextIdApplicationContextInitializer
3. class org.springframework.boot.context.ConfigurationWarningsApplicationContextInitializer
4. class org.springframework.boot.web.context.ServerPortInfoApplicationContextInitializer
5. class org.springframework.boot.autoconfigure.SharedMetadataReaderFactoryContextInitializer
6. class org.springframework.boot.autoconfigure.logging.ConditionEvaluationReportLoggingListener

## ApplicationListener
1. class org.springframework.boot.context.config.ConfigFileApplicationListener
2. class org.springframework.boot.context.config.AnsiOutputApplicationListener
3. class org.springframework.boot.context.logging.LoggingApplicationListener
4. class org.springframework.boot.context.logging.ClasspathLoggingApplicationListener
5. class org.springframework.boot.autoconfigure.BackgroundPreinitializer
6. class org.springframework.boot.context.config.DelegatingApplicationListener
7. class org.springframework.boot.builder.ParentContextCloserApplicationListener
8. class org.springframework.boot.ClearCachesApplicationListener
9. class org.springframework.boot.context.FileEncodingApplicationListener
10. class org.springframework.boot.liquibase.LiquibaseServiceLocatorApplicationListener



org.springframework.boot.SpringApplicationRunListener


org.springframework.boot.context.event.EventPublishingRunListener


使用的 ApplicationContext
org.springframework.boot.web.servlet.context.AnnotationConfigServletWebServerApplicationContext

继承下面的
ServletWebServerApplicationContext



## 9大组件
编号后面是接口，下面是默认实现
1. MultipartResolver
class org.springframework.web.multipart.support.StandardServletMultipartResolver
```java
public interface MultipartResolver {

	boolean isMultipart(HttpServletRequest request);

	MultipartHttpServletRequest resolveMultipart(HttpServletRequest request) throws MultipartException;

	void cleanupMultipart(MultipartHttpServletRequest request);

}
``` 

顾名思义，这里就是处理文件上传的地方，做三件事，判断请求是不是文件上传，将 request 转换成 文件上传 request，以及清理工作  
`StandardServletMultipartResolver` 标准解析，其实只是从标准 request 解析part 出来，所谓转行 request，实际上是让内部知道是否为文件上传请求，接收一个参数，用来控制是否立即解析，
默认是立即解析，非立即就是在第一次调用 getMultipart 的时候触发解析

2. LocaleResolver
class org.springframework.web.servlet.i18n.AcceptHeaderLocaleResolver

```java
public interface LocaleResolver {

	Locale resolveLocale(HttpServletRequest request);

	void setLocale(HttpServletRequest request, @Nullable HttpServletResponse response, @Nullable Locale locale);

}
``` 

语言解析，两个方法，解析使用的语言，设置语言。`AcceptHeaderLocaleResolver` 这个就是从请求头里面解析，但是没法设置   



3. ThemeResolver
class org.springframework.web.servlet.theme.FixedThemeResolver
```java
public interface ThemeResolver {

	String resolveThemeName(HttpServletRequest request);

	void setThemeName(HttpServletRequest request, @Nullable HttpServletResponse response, @Nullable String themeName);
}
``` 

主题解析和设置

4. HandlerMapping
class org.springframework.web.servlet.handler.SimpleUrlHandlerMapping
class org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping
class org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping
class org.springframework.web.servlet.handler.SimpleUrlHandlerMapping
class org.springframework.web.servlet.config.annotation.WebMvcConfigurationSupport$EmptyHandlerMapping
class org.springframework.web.servlet.config.annotation.WebMvcConfigurationSupport$EmptyHandlerMapping
class org.springframework.boot.autoconfigure.web.servlet.WelcomePageHandlerMapping

重头戏，这里很重要  
```java
public interface HandlerMapping {

	String PATH_WITHIN_HANDLER_MAPPING_ATTRIBUTE = HandlerMapping.class.getName() + ".pathWithinHandlerMapping";

	String BEST_MATCHING_PATTERN_ATTRIBUTE = HandlerMapping.class.getName() + ".bestMatchingPattern";

	String INTROSPECT_TYPE_LEVEL_MAPPING = HandlerMapping.class.getName() + ".introspectTypeLevelMapping";

	String URI_TEMPLATE_VARIABLES_ATTRIBUTE = HandlerMapping.class.getName() + ".uriTemplateVariables";

	String MATRIX_VARIABLES_ATTRIBUTE = HandlerMapping.class.getName() + ".matrixVariables";

	String PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE = HandlerMapping.class.getName() + ".producibleMediaTypes";

	@Nullable
	HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception;
}
``` 

只有一个方法，返回 `HandlerExecutionChain`,这里面将有 handler 实例（controller 里面的方法）还有拦截器，里面是具体执行的业务逻辑 
列出里面关键字段 
```java
    private final Object handler;

	@Nullable
	private HandlerInterceptor[] interceptors;

	@Nullable
	private List<HandlerInterceptor> interceptorList;

	private int interceptorIndex = -1;
``` 

重要的 handlerMapping 主要有几个，都是继承

AbstractHandlerMapping 

这个类里面处理 拦截器 跨域，url 匹配的类
最终，这里实现了 getHandler 方法，并且是 final 的，不允许付款，也就是说，这里定义了寻找 handler 的具体流程
1. getHandlerInternal 这个方法留给子类实现
2. 上一步如果没找到就 getDefaultHandler  这个默认处理器是外部设置的
3. 上面两步没找到，那就是没有了
4. 如果找到了，判断是否为 string，如果是，则认为是 beanName，直接在容器里根据 beanName 获取，这里没捉异常，没获取到就异常出去了
5. 使用这个找到的 handler 构建 HandlerExecutionChain ，添加拦截器
6. 如果是cors跨域请求，分两种，一是 option 方法 handler 换成 PreFlightHandler 内部是 DefaultCorsProcessor 处理的，二是正常请求，带 origin 了，在 chain 里面添加一个 cors 拦截器 CorsInterceptor，内部也是 DefaultCorsProcessor 处理的 


接下来，子类各自实现 getHandlerInternal 方法就可以了	
下面一层有两个子类 
1. AbstractUrlHandlerMapping 这个是基于处理 url 的
2. AbstractHandlerMethodMapping  

AbstractUrlHandlerMapping 里面又添加了一个 rootHandler ，外部设置，用来处理 / 的请求，这里的 getHandlerInternal 处理流程是
1. 根据 request 获取 url：lookupPath
2. 使用这个 lookupPath 获取 handler ，调用的是 lookupHandler 方法
3. 如果没找到，判断 URL 是否为 /,如果是，就使用 rootHandler 
4. 如果还是 null  就使用 defaultHandler
5. 如果存在了 ，判断是否为 string，如果是，则认为是 beanName，直接在容器里根据 beanName 获取，这里没捉异常，没获取到就异常出去了
6. validateHandler 验证一下，这个留给子类实现
7. 构建 HandlerExecutionChain，这里构建的 chain 里面添加了 PathExposingHandlerInterceptor 拦截器，可能添加 UriTemplateVariablesHandlerInterceptor 拦截器 这两个拦截器的作用就是在 request 里面设置一些属性，还有 第二步，如果找到 handler 后，构建步骤跟这里的一样

这个类有一个 map 存放 url -> handler  有一个方法 registerHandler 给子类往里面注册 handler  

这个抽象类下面，就有2个直接实现了	
WelcomePageHandlerMapping
这个类重写了 getHandlerInternal 方法，根据 request 的 accept ，来判断，如果包含 text/html 那就调用父类，也就是上面的查找流程，否则返回null，也就是，这里只处理 html 请求，不过看名字也就清楚了

SimpleUrlHandlerMapping
这个类就是注册处理 handler 或者 handler 的 beanName 
在本类初始化的过程中，会根据注册的信息完成初始化，简单的说，就是这个对象有一个 Map<String,Object> 左边是 url 右边是 handler 或者 beanName，在初始化过程中，如果是 beanName 那么就会从容器里面获取 handler
另外，如果注册的 url 是 / 那么本 handler 就是 rootHandler ，如果 url 是 /* 那么 handler 就是 defaultHandler 注意一点的是 


除了上面有个直接实现，还有一个抽象子类 AbstractDetectingUrlHandlerMapping

AbstractDetectingUrlHandlerMapping
这个抽象类，也是在初始化的时候寻找 handler，过程是获取容器里的所有 beanName，使用 determineUrlsForHandler 传入参数是 beanName 来获取 url 数组，如果获取到了，就注册 handler
determineUrlsForHandler 这个方法是一个抽象方法，子类实现


接下来的实现类是 
BeanNameUrlHandlerMapping

那么这个类主要功能就是判断 beanName 是不是 / 开头的，如果是，那么就代表，本类是 handler 类，处理的url 就是 beanName， 如果 beanName 没有 / 开头，继续看别名


这里是分割线，接下来换另一个继承分支,这个分支才是正常流程常走的地方
AbstractHandlerMethodMapping
这里面重要的属性是 MappingRegistry 就是 handler 注册的地方，在初始化的时候做一些事情
首先，获取所有 beanName， 然后排除一部分代理 也就是 beanName 以 scopedTarget. 开头的，排除后，就获取 beanName 对应的 class ，判断是否为 hander ，如果是，就找里面的 method 然后注册 handler
这里的 handler 类型都是 HandleMehtod ，是根据方法划分的，上面的 AbstractUrlHandlerMapping 是一个 bean 就是 handler
判断是否为 handler 类型方法 isHandler 由子类实现，
获取 handlemethod 的方法 detectHandlerMethods 逻辑为
1. 获取具体类型,如果是代理，就找到具体类
2. 过滤方法的具体逻辑是 getMappingForMethod ，这个也是子类实现，方法包含了类型的所有接口
3. 找到后就注册 handler ，三个参数 mapping, handler, method  其中 mapping 是泛型的，handler 就是处理的 beanName 或者 handler 对象，method 就是找到的方法
4. 使用 handlerMethodsInitialized 方法处理已经注册的 handler ，空方法，留给子类覆盖

上面是初始化的过程，后面要具体看看 MappingRegistry 的注册和逻辑
这个类有一个注册方法 register ，里面会 initCorsConfiguration 初始化自定义 cors 跨域配置，这是一个抽象方法

这个类的 getHandlerInternal 逻辑
1. 处理 url 获取 lookupPath
2. 根据 url 找 HandlerMethod  方法是 lookupHandlerMethod 
怎么找的，就是根据 url 找 mapping 如果没找到 mapping  如果有 mapping（列表） 那就遍历 根据 mapping 和 request 找匹配的 mapping 方法是 getMatchingMapping  子类实现
如果没有匹配的，那就使用所有的 mapping 都试着匹配一次
如果没匹配到 handleNoMatch 默认返回 null
匹配到的话，getMappingComparator 根据 request 获取排序器，将匹配到的 mapping 进行排序，然后找到里面最匹配的一个，最匹配的一个就是第一个，实际上排序就完成了选择了，如果多个匹配，则异常
执行 handleMatch 匹配成功方法，这里只是设置一个 request 属性
返回 HandlerMethod



接下来继续看下面的子类
RequestMappingInfoHandlerMapping
这里，泛型已经具体了 RequestMappingInfo  很多匹配逻辑都在里面，后面可以看看
这个抽象类实现了一些方法  
1. getMatchingMapping  找到匹配的方法 ，具体是 RequestMappingInfo.getMatchingCondition（request）
2. getMappingComparator 获取排序器，实际上是 RequestMappingInfo 有 compareTo 方法了，直接比较就行了
3. handleMatch 在匹配成功后做的一些操作，继续设置一些 request 属性，譬如 mediatype uri 参数
4. handleNoMatch 在没匹配后做的一些操作，具体就是，看到底是什么不匹配，抛出对应异常，譬如 HttpRequestMethodNotSupportedException ,HttpMediaTypeNotSupportedException 等


接下来就到最终实现类了	
RequestMappingHandlerMapping
首先，他要实现的是 isHandler ，判断 bean 是否为处理类，逻辑是类上有注解 @Controller 或者 @RequestMapping ，注解上的注解也行
接下来是 getMappingForMethod 这个方法是判断 method 是否为处理方法的。逻辑是看有没有 @RequestMapping 注解，注解上的注解也行，譬如 @GetMapping
然后 initCorsConfiguration 方法，重写了，处理类上和方法上的 @CrossOrigin 注解，自定义 cors 跨域配置
最后有一个 match 方法，是 MatchableHandlerMapping 接口定义的，使用 request 和 pattern 进行匹配

上面留了两个坑，AbstractHandlerMethodMapping 类里面的 MappingRegistry 和 RequestMappingInfoHandlerMapping 类里面的 RequestMappingInfo
这两个涉及到注册 handler 和 判断是否匹配，待填

5. HandlerAdapter
class org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter
class org.springframework.web.servlet.mvc.HttpRequestHandlerAdapter
class org.springframework.web.servlet.mvc.SimpleControllerHandlerAdapter

HandlerAdapter
这个主要是适配器，来调用 handler 的，在之前和之后做一些增强处理，每个 adapter 类型，支持对应的 handler 类型
三个方法
1. boolean supports(Object handler);
2. ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception;
3. long getLastModified(HttpServletRequest request, Object handler);

主要有 4 中 adapter
1. HttpRequestHandlerAdapter 这个处理 handler 的类型为 HttpRequestHandler 
2. SimpleServletHandlerAdapter 这个处理 handler 的类型为 Servlet 调用 service 方法
3. SimpleControllerHandlerAdapter 这个处理 handler 的类型为 Controller 
4. RequestMappingHandlerAdapter 这个是大部分逻辑走的地方，上面3个一般是 spring 自己用的，这个处理的 handler 类型是 HandlerMethod 

重点关注 RequestMappingHandlerAdapter 其他3个很简单

有一个继承 AbstractHandlerMethodAdapter
这里实现 supports 方法 支持 HandlerMethod  并且要满足 supportsInternal 返回true 这是一个抽象方法
实现 handle 方法，将 handler 转型为 HandlerMethod ，调用 handleInternal 也是一个抽象方法
实现 getLastModified 方法，将 handler 转型为 HandlerMethod ，调用 getLastModifiedInternal 也是一个抽象方法
。。。

直接看 RequestMappingHandlerAdapter 
这个里面的东西就很多了，重点要关注的是初始化方法，这个类实现了 InitializingBean 所以会触发初始化方法 afterPropertiesSet 在这个方法里面，就可以看到它有哪些扩展组件了  
初始化主要做4件事
1. initControllerAdviceCache 
看名字就知道了 找 @ControllerAdvice 注解的 bean ，
然后找里面带 @RequestMapping && @ModelAttribute 注解的方法，丢到 modelAttributeAdviceCache 里面
找带 @InitBinder 注解的方法丢到 initBinderAdviceCache 里面
然后处理 RequestBodyAdvice 和 ResponseBodyAdvice

2. argumentResolvers
这里就是请求参数处理的地方，参数处理接口是 HandlerMethodArgumentResolver
获取所有默认的参数处理，和自定义的参数处理，默认的优先
然后用 HandlerMethodArgumentResolverComposite 对象组合

3. initBinderArgumentResolvers
其实跟上面一样的，只是默认添加的东西不一样，自定义的还是在这里
一样是 HandlerMethodArgumentResolver 用 HandlerMethodArgumentResolverComposite 对象组合

4. returnValueHandlers
这个是处理返回结果的 HandlerMethodReturnValueHandler
使用 HandlerMethodReturnValueHandlerComposite 对象组合

这里面重要的是要实现3个抽象方法 
1. supportsInternal 这个固定返回 true
2. getLastModifiedInternal 返回 -1 ，到这里的请求，通常是业务处理，不用 http 缓存
3. handleInternal
就是业务处理了，其实是做业务处理前后的一些事情，具体业务处理是 HanderMethod 完成的

这里的逻辑
1. 检查 request http 方法这里是否支持，如果需要 session ，则判断 session 是否存在
2. 调用 invokeHandlerMethod 虽然有判断，本质还是调用这个方法
里面的细节不详细看了，看初始化的东西，就知道里面是要干啥了。

为什么会有 adapter 呢，是为了增强，需要在业务逻辑前后做一些处理，譬如参数注入，返回数据处理等	

`RequestMappingHandlerAdapter` 处理的 handler 为 `HandlerMethod`  
`HttpRequestHandlerAdapter` 处理的 handler 为 `HttpRequestHandler`  
`SimpleControllerHandlerAdapter` 处理的 handler 为 `Controller` 这里的 controler 是一个接口，不是常用的注解  

从 debug 路径来看，大部分的 handler 都是 `HandlerMethod`, 也就是说，使用的代理是 `RequestMappingHandlerAdapter` 
除了上面默认的几个外，还有一个实现 `SimpleServletHandlerAdapter` 这个是处理 `Servlet` 的  

接下来重点关照 `RequestMappingHandlerAdapter`	

首先，要看一下调用流程,下面是 `doDispatch` 方法里面的片段
```java
	// Determine handler adapter for the current request.
	HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

	// Process last-modified header, if supported by the handler.
	String method = request.getMethod();
	boolean isGet = "GET".equals(method);
	if (isGet || "HEAD".equals(method)) {
		long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
		if (logger.isDebugEnabled()) {
			logger.debug("Last-Modified value for [" + getRequestUri(request) + "] is: " + lastModified);
		}
		if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
			return;
		}
	}

	if (!mappedHandler.applyPreHandle(processedRequest, response)) {
		return;
	}

	// Actually invoke the handler.
	mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
```	
可以看到在获取到对应的 adapter 后，先触发前置拦截器，这里说明一下 `mappedHandler` 是 `HandlerExecutionChain` 类型，可以通过调用其 `getHandler` 方法获取处理类，`mappedHandler` 里面还有拦截器等其他信息.  
然后就是 adapter 执行 handler 了，注意传入的参数，请求，响应，以及处理 handler，这个handler 是 adapter 支持的 handler	

看看里面的处理逻辑,很长，又是代码了	

```java
protected ModelAndView handleInternal(HttpServletRequest request,
			HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

	ModelAndView mav;
	// 检查 http method 是否支持，如果需要 session 则判断是否存在 session
	checkRequest(request);

	// Execute invokeHandlerMethod in synchronized block if required.
	if (this.synchronizeOnSession) {
		HttpSession session = request.getSession(false);
		if (session != null) {
			Object mutex = WebUtils.getSessionMutex(session);
			synchronized (mutex) {
				mav = invokeHandlerMethod(request, response, handlerMethod);
			}
		}
		else {
			// No HttpSession available -> no mutex necessary
			mav = invokeHandlerMethod(request, response, handlerMethod);
		}
	}
	else {
		// No synchronization on session demanded at all...
		mav = invokeHandlerMethod(request, response, handlerMethod);
	}

	if (!response.containsHeader(HEADER_CACHE_CONTROL)) {
		if (getSessionAttributesHandler(handlerMethod).hasSessionAttributes()) {
			applyCacheSeconds(response, this.cacheSecondsForSessionAttributeHandlers);
		}
		else {
			prepareResponse(response);
		}
	}

	return mav;
}
```	

虽然多，核心方法是 `invokeHandlerMethod`,其他的是控制代码，后面有兴趣可以仔细看看	
这个方法又是一个很复杂的方法，需要细看，里面有很多请求处理相关的，譬如参数是如果映射的  
```java
protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
			HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {
	// 包装一下 request，这个就是内部常用到的 WebRequest 的实现类了
	ServletWebRequest webRequest = new ServletWebRequest(request, response);
	try {
		// 处理 @InitBinder 注解的，很少用到，跳过，全局的可以通过 @ControllerAdvice 注解的类定义，全局优先
		WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
		// @RequestMapping && @ModelAttribute 注解同时标记的，同上，全局优先,  PS: 这个注解也用的比较少，后面可以瞅瞅
		ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);
		// 这个类继承 HandlerMethod ，new 一个，就是把原来的属性全部赋值给新的对象了
		ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
		// 设置 参数解析，这里就应该熟悉了，比如常用的 @RequestParam 注解等，都是这里处理的，也可以自己定义添加 HandlerMethodArgumentResolver
		// 这里是 HandlerMethodArgumentResolverComposite 里面组合所有的实际解析类
		// 在 getDefaultArgumentResolvers 方法里面，有默认清单
		if (this.argumentResolvers != null) {
			invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
		}
		// 设置响应解析，基本同上，接口是 HandlerMethodReturnValueHandler
		if (this.returnValueHandlers != null) {
			invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
		}
		invocableMethod.setDataBinderFactory(binderFactory);
		invocableMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);

		// 这里填充 一些属性 主要是围绕 @ModelAttribute 注解
		ModelAndViewContainer mavContainer = new ModelAndViewContainer();
		mavContainer.addAllAttributes(RequestContextUtils.getInputFlashMap(request));
		modelFactory.initModel(webRequest, mavContainer, invocableMethod);
		mavContainer.setIgnoreDefaultModelOnRedirect(this.ignoreDefaultModelOnRedirect);


		// servlet3.0 的异步请求
		AsyncWebRequest asyncWebRequest = WebAsyncUtils.createAsyncWebRequest(request, response);
		asyncWebRequest.setTimeout(this.asyncRequestTimeout);

		WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
		asyncManager.setTaskExecutor(this.taskExecutor);
		asyncManager.setAsyncWebRequest(asyncWebRequest);
		asyncManager.registerCallableInterceptors(this.callableInterceptors);
		asyncManager.registerDeferredResultInterceptors(this.deferredResultInterceptors);

		if (asyncManager.hasConcurrentResult()) {
			Object result = asyncManager.getConcurrentResult();
			mavContainer = (ModelAndViewContainer) asyncManager.getConcurrentResultContext()[0];
			asyncManager.clearConcurrentResult();
			if (logger.isDebugEnabled()) {
				logger.debug("Found concurrent result value [" + result + "]");
			}
			invocableMethod = invocableMethod.wrapConcurrentResult(result);
		}

		// 上面都是异步相关，一般都是跳过，这里才是调用，之前的一些字段已经设置到 invocableMethod 里面了 mavContainer 里面包含 session 属性
		invocableMethod.invokeAndHandle(webRequest, mavContainer);
		if (asyncManager.isConcurrentHandlingStarted()) {
			return null;
		}

		// 获取响应结果
		return getModelAndView(mavContainer, modelFactory, webRequest);
	}
	finally {
		webRequest.requestCompleted();
	}
}
```	

继续重点方法分析 `invokeAndHandle`  	
```java
public void invokeAndHandle(ServletWebRequest webRequest, ModelAndViewContainer mavContainer,
			Object... providedArgs) throws Exception {
    // 调用，获取返回结果，里面涉及到 参数注入，记住一点，处理方法的参数，一定是要 HandlerMethodArgumentResolver 可以处理的
	Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);

	// 设置 http 响应状态码，大部分时候，这里是没设置的，除非使用注解 @ResponseStatus 标记
	setResponseStatus(webRequest);

	// 对于没有返回值的情况，如果是 请求资源没变（http 缓存），或者自定义响应状态码了（譬如204），或者已经被标记处理过了，那么标记处理完成
	// 如果有状态说明原因，也处理完成
	if (returnValue == null) {
		if (isRequestNotModified(webRequest) || getResponseStatus() != null || mavContainer.isRequestHandled()) {
			mavContainer.setRequestHandled(true);
			return;
		}
	}
	else if (StringUtils.hasText(getResponseStatusReason())) {
		mavContainer.setRequestHandled(true);
		return;
	}

	// 否则，接下来就处理响应结果，这是大部分时候走的地方，关于请求参数和响应结果的处理，可以详细看看初始化的时候，spring 默认设置了哪些，这里就不往下看了
	mavContainer.setRequestHandled(false);
	Assert.state(this.returnValueHandlers != null, "No return value handlers");
	try {
		this.returnValueHandlers.handleReturnValue(
				returnValue, getReturnValueType(returnValue), mavContainer, webRequest);
	}
	catch (Exception ex) {
		if (logger.isTraceEnabled()) {
			logger.trace(getReturnValueHandlingErrorMessage("Error handling return value", returnValue), ex);
		}
		throw ex;
	}
}
```	

获取最终结果
```java
private ModelAndView getModelAndView(ModelAndViewContainer mavContainer,
			ModelFactory modelFactory, NativeWebRequest webRequest) throws Exception {

	modelFactory.updateModel(webRequest, mavContainer);
	if (mavContainer.isRequestHandled()) {
		return null;
	}
	ModelMap model = mavContainer.getModel();
	ModelAndView mav = new ModelAndView(mavContainer.getViewName(), model, mavContainer.getStatus());
	if (!mavContainer.isViewReference()) {
		mav.setView((View) mavContainer.getView());
	}
	if (model instanceof RedirectAttributes) {
		Map<String, ?> flashAttributes = ((RedirectAttributes) model).getFlashAttributes();
		HttpServletRequest request = webRequest.getNativeRequest(HttpServletRequest.class);
		if (request != null) {
			RequestContextUtils.getOutputFlashMap(request).putAll(flashAttributes);
		}
	}
	return mav;
}
```	


6. HandlerExceptionResolver
class org.springframework.boot.web.servlet.error.DefaultErrorAttributes
class org.springframework.web.servlet.handler.HandlerExceptionResolverComposite

异常处理

7. RequestToViewNameTranslator
class org.springframework.web.servlet.view.DefaultRequestToViewNameTranslator

处理响应结果和异常，没有返回 mv 的时候，尝试获取对应的view

8. ViewResolver
class org.springframework.web.servlet.view.ContentNegotiatingViewResolver
class org.springframework.web.servlet.view.BeanNameViewResolver
class org.springframework.web.servlet.view.ViewResolverComposite
class org.springframework.web.servlet.view.InternalResourceViewResolver

视图渲染

9.  FlashMapManager
class org.springframework.web.servlet.support.SessionFlashMapManager

用来保存上一次请求设置的数据，这样在本次请求就可以获取了