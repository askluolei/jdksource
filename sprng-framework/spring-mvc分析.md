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

上面关键的是 RequestMappingHandlerMapping 大部分业务处理在这里

5. HandlerAdapter
class org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter
class org.springframework.web.servlet.mvc.HttpRequestHandlerAdapter
class org.springframework.web.servlet.mvc.SimpleControllerHandlerAdapter

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

异常处理,处理的是 adpeter 调用 handle 方法可能出现的异常
在 processDispatchResult 方法里面处理了
有一个 HandlerExceptionResolver 列表，处理异常

7. RequestToViewNameTranslator
class org.springframework.web.servlet.view.DefaultRequestToViewNameTranslator

处理响应结果和异常，没有返回 mv 的时候，尝试获取对应的view
这里只在 getDefaultViewName 方法里面使用，从 request 的url 找到对应的模版

8. ViewResolver
class org.springframework.web.servlet.view.ContentNegotiatingViewResolver
class org.springframework.web.servlet.view.BeanNameViewResolver
class org.springframework.web.servlet.view.ViewResolverComposite
class org.springframework.web.servlet.view.InternalResourceViewResolver

视图渲染
根据 view name 找到对应的视图
一个 ViewResolver 列表，遍历

9.  FlashMapManager
class org.springframework.web.servlet.support.SessionFlashMapManager

用来保存上一次请求设置的数据，这样在本次请求就可以获取了