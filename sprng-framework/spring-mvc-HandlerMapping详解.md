HandlerMapping

这个类的主要作用就是匹配 request 有哪个 handler 处理
返回的是 HandlerExecutionChain 里面包含 具体的 handler 拦截器 和 跨域处理

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