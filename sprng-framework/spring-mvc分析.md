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

只有一个方法，返回 `HandlerExecutionChain`,这里面将有 handler 实例（controller 里面的方法）还有拦截器  
列出里面关键字段 
```java
    private final Object handler;

	@Nullable
	private HandlerInterceptor[] interceptors;

	@Nullable
	private List<HandlerInterceptor> interceptorList;

	private int interceptorIndex = -1;
``` 



5. HandlerAdapter
class org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter
class org.springframework.web.servlet.mvc.HttpRequestHandlerAdapter
class org.springframework.web.servlet.mvc.SimpleControllerHandlerAdapter

6. HandlerExceptionResolver
class org.springframework.boot.web.servlet.error.DefaultErrorAttributes
class org.springframework.web.servlet.handler.HandlerExceptionResolverComposite

7. RequestToViewNameTranslator
class org.springframework.web.servlet.view.DefaultRequestToViewNameTranslator

8. ViewResolver
class org.springframework.web.servlet.view.ContentNegotiatingViewResolver
class org.springframework.web.servlet.view.BeanNameViewResolver
class org.springframework.web.servlet.view.ViewResolverComposite
class org.springframework.web.servlet.view.InternalResourceViewResolver

9.  FlashMapManager
class org.springframework.web.servlet.support.SessionFlashMapManager

