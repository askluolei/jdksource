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
	invokeHandlerMethod 还是要说几句关键步骤
	1. 根据 handler 的参数列表，使用 HandlerMethodArgumentResolver 看看能不能处理该参数，设置对应值，如果有处理不了的就异常了
	2. 使用参数，调用 handler 方法，获取返回结果
	3. 处理响应码，响应码和原因可以使用注解自定义 @ResponseStatus(code = HttpStatus.ACCEPTED, reason = "test") 如果没自定义，那就spring 自己处理了，大部分都是 200
	4. 然后只用 HandlerMethodReturnValueHandler ，看能不能处理返回类型的东西，譬如我们常用 @RestController 或者 @ResponseBody 返回对象或者字符串的，都会在这里就处理了，写入 response，然后后面就跑流程了
3. getModelAndView 这里的方法，如果响应结果有人处理了，这里就返回 null 了，如果没处理，可能就是返回的 modelandview 或者 view ，构造返回
