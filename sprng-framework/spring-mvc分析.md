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
1. MultipartResolver
class org.springframework.web.multipart.support.StandardServletMultipartResolver

2. LocaleResolver
class org.springframework.web.servlet.i18n.AcceptHeaderLocaleResolver

3. ThemeResolver
class org.springframework.web.servlet.theme.FixedThemeResolver

4. HandlerMapping
class org.springframework.web.servlet.handler.SimpleUrlHandlerMapping
class org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping
class org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping
class org.springframework.web.servlet.handler.SimpleUrlHandlerMapping
class org.springframework.web.servlet.config.annotation.WebMvcConfigurationSupport$EmptyHandlerMapping
class org.springframework.web.servlet.config.annotation.WebMvcConfigurationSupport$EmptyHandlerMapping
class org.springframework.boot.autoconfigure.web.servlet.WelcomePageHandlerMapping

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

