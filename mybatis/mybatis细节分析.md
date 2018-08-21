[TOC]   

# mybatis细节分析   
主要看看一些功能点的实现，譬如 扩展，一二级缓存，mapper 接口    

## 扩展（plugins）  
直接看在哪里增加扩展的  
```java
  public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
    executorType = executorType == null ? defaultExecutorType : executorType;
    executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
    Executor executor;
    if (ExecutorType.BATCH == executorType) {
      executor = new BatchExecutor(this, transaction);
    } else if (ExecutorType.REUSE == executorType) {
      executor = new ReuseExecutor(this, transaction);
    } else {
      executor = new SimpleExecutor(this, transaction);
    }
    if (cacheEnabled) {
      executor = new CachingExecutor(executor);
    }
    executor = (Executor) interceptorChain.pluginAll(executor);
    return executor;
  }

public ParameterHandler newParameterHandler(MappedStatement mappedStatement, Object parameterObject, BoundSql boundSql) {
    ParameterHandler parameterHandler = mappedStatement.getLang().createParameterHandler(mappedStatement, parameterObject, boundSql);
    parameterHandler = (ParameterHandler) interceptorChain.pluginAll(parameterHandler);
    return parameterHandler;
  }

  public ResultSetHandler newResultSetHandler(Executor executor, MappedStatement mappedStatement, RowBounds rowBounds, ParameterHandler parameterHandler,
      ResultHandler resultHandler, BoundSql boundSql) {
    ResultSetHandler resultSetHandler = new DefaultResultSetHandler(executor, mappedStatement, parameterHandler, resultHandler, boundSql, rowBounds);
    resultSetHandler = (ResultSetHandler) interceptorChain.pluginAll(resultSetHandler);
    return resultSetHandler;
  }

  public StatementHandler newStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
    StatementHandler statementHandler = new RoutingStatementHandler(executor, mappedStatement, parameterObject, rowBounds, resultHandler, boundSql);
    // 这里就是扩展了，使用插件扩展，主要扩展接口是 Interceptor ，被代理的是 StatementHandler 的实现类
    statementHandler = (StatementHandler) interceptorChain.pluginAll(statementHandler);
    return statementHandler;
  }
``` 

我们可以看到自定义插件可以扩展的接口有4个 Executor，ParameterHandler，ResultSetHandler，StatementHandler    
基本涵盖了整个执行流程，都是调用 `interceptorChain.pluginAll`,我们看看怎么实现的扩展    
```java
public class InterceptorChain {

  private final List<Interceptor> interceptors = new ArrayList<>();

  public Object pluginAll(Object target) {
    for (Interceptor interceptor : interceptors) {
      target = interceptor.plugin(target);
    }
    return target;
  }

  public void addInterceptor(Interceptor interceptor) {
    interceptors.add(interceptor);
  }
  
  public List<Interceptor> getInterceptors() {
    return Collections.unmodifiableList(interceptors);
  }

}
``` 

我们自定义的扩展需要实现 Interceptor 接口，然后注册到 InterceptorChain 中，最后，依次调用 plugin 方法   
我们看一个示例  
```java
@Intercepts({
      @Signature(type = Map.class, method = "get", args = {Object.class})})
  public static class AlwaysMapPlugin implements Interceptor {
    @Override
    public Object intercept(Invocation invocation) throws Throwable {
      return "Always";
    }

    @Override
    public Object plugin(Object target) {
      return Plugin.wrap(target, this);
    }

    @Override
    public void setProperties(Properties properties) {
    }
  }
``` 

扩展必须要实现 Interceptor 接口，里面有3个方法  
1. setProperties    
这个方法是在解析的时候，可以配置一些参数到里面，在add之前被调用     
我们看一下插件解析过程就清楚了
```java
  private void pluginElement(XNode parent) throws Exception {
    if (parent != null) {
      for (XNode child : parent.getChildren()) {
        String interceptor = child.getStringAttribute("interceptor");
        Properties properties = child.getChildrenAsProperties();
        Interceptor interceptorInstance = (Interceptor) resolveClass(interceptor).newInstance();
        interceptorInstance.setProperties(properties);
        configuration.addInterceptor(interceptorInstance);
      }
    }
  }
``` 

2. plugin   
这个方法的调用可以看到是在做扩展的时候。    
一般调用 `Plugin.wrap(target, this)` 方法， 
或者，可以做条件判断，是自己要处理就就这样调用，不是自己处理的，就直接返回 target。 

3. intercept
这个方法就是具体的拦截信息了    
可以在类上添加注解，拦截注解想要的类和方法，这样 Invocation 里面就是自己想要的东西了    

我们具体看看 `Plugin.wrap` 方法，这个方式是实现扩展的关键，原理就是动态代理 
```java
/**
 * JDK 的动态代理，实现 Plugin 扩展
 * @author Clinton Begin
 */
public class Plugin implements InvocationHandler {

  private final Object target;
  private final Interceptor interceptor;
  private final Map<Class<?>, Set<Method>> signatureMap;

  private Plugin(Object target, Interceptor interceptor, Map<Class<?>, Set<Method>> signatureMap) {
    this.target = target;
    this.interceptor = interceptor;
    this.signatureMap = signatureMap;
  }

  public static Object wrap(Object target, Interceptor interceptor) {
    // 获取注解数据
    Map<Class<?>, Set<Method>> signatureMap = getSignatureMap(interceptor);
    // 如果经过多次代理，那这里就是代理对象的 class了
    Class<?> type = target.getClass();
    // 就是过滤出上面需要的代理的接口，实现代理就行了
    Class<?>[] interfaces = getAllInterfaces(type, signatureMap);
    if (interfaces.length > 0) {
      return Proxy.newProxyInstance(
          type.getClassLoader(),
          interfaces,
          new Plugin(target, interceptor, signatureMap));
    }
    return target;
  }

  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      Set<Method> methods = signatureMap.get(method.getDeclaringClass());
      // 如果满足条件，就调用拦截器
      if (methods != null && methods.contains(method)) {
        return interceptor.intercept(new Invocation(target, method, args));
      }
      // 否则调用原本对象的方法
      return method.invoke(target, args);
    } catch (Exception e) {
      throw ExceptionUtil.unwrapThrowable(e);
    }
  }

  private static Map<Class<?>, Set<Method>> getSignatureMap(Interceptor interceptor) {
    Intercepts interceptsAnnotation = interceptor.getClass().getAnnotation(Intercepts.class);
    // issue #251
    // 看到，Interceptor 扩展 必须要有 Intercepts 注解
    if (interceptsAnnotation == null) {
      throw new PluginException("No @Intercepts annotation was found in interceptor " + interceptor.getClass().getName());      
    }
    // 拿到所有签名,就是过滤条件。。就是要代理哪个对象的哪个方法，可以指定多个
    Signature[] sigs = interceptsAnnotation.value();
    Map<Class<?>, Set<Method>> signatureMap = new HashMap<>();
    for (Signature sig : sigs) {
      Set<Method> methods = signatureMap.computeIfAbsent(sig.type(), k -> new HashSet<>());
      try {
        Method method = sig.type().getMethod(sig.method(), sig.args());
        methods.add(method);
      } catch (NoSuchMethodException e) {
        throw new PluginException("Could not find method on " + sig.type() + " named " + sig.method() + ". Cause: " + e, e);
      }
    }
    return signatureMap;
  }

  private static Class<?>[] getAllInterfaces(Class<?> type, Map<Class<?>, Set<Method>> signatureMap) {
    Set<Class<?>> interfaces = new HashSet<>();
    while (type != null) {
      for (Class<?> c : type.getInterfaces()) {
        if (signatureMap.containsKey(c)) {
          interfaces.add(c);
        }
      }
      type = type.getSuperclass();
    }
    return interfaces.toArray(new Class<?>[interfaces.size()]);
  }

}
``` 

看到，扩展必须要标记 @Intercepts 注解， @Signature 代表要拦截的类型，方法，参数，可以指定多个，最后，@Signature 标记过的类型（接口），才被代理  
使用的是 JDK 代理，调用的就是这里的 invoke 方法
原理很简单，可以扩展的接口就是上面说的4个（也就是 @Signature 注解里面 type 可以指定的，如果指定了其他类型是无效的，为啥无效，请自己分析）   

## 缓存（cache）
缓存在执行分析里面看过了，只不过没有细讲，我们先从二级缓存开始，也就是 CachingExecutor 类   
```java
  @Override
  public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
      throws SQLException {
    // 获取 cache，可能开了二级缓存，但是对应的 mapper 没有定义，二级缓存的粒度是到 mapper 的
    // 这里的 cache 是跟 ms 绑定的
    Cache cache = ms.getCache();
    if (cache != null) {
      // 看是否需要刷新缓存
      flushCacheIfRequired(ms);
      if (ms.isUseCache() && resultHandler == null) {
        // 这里是 callable 需要的
        ensureNoOutParams(ms, boundSql);
        // 先从缓存里面获取
        @SuppressWarnings("unchecked")
        List<E> list = (List<E>) tcm.getObject(cache, key);
        if (list == null) {
          // 没有就去查询，这里没有缓存穿透处理，处理再一级缓存里面
          // 一级缓存，就是跟 executor 绑定的
          list = delegate.<E> query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
          // 放入缓存
          tcm.putObject(cache, key, list); // issue #578 and #116
        }
        return list;
      }
    }
    // 如果没有定义缓存，直接查询
    return delegate.<E> query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }

  private void flushCacheIfRequired(MappedStatement ms) {
    Cache cache = ms.getCache();
    // 这里 ms 代表一条sql，或者一个方法，每个方法都可以单独设置是否刷新缓存，默认是 查询不刷新，增删改刷新
    if (cache != null && ms.isFlushCacheRequired()) {      
      tcm.clear(cache);
    }
  }
``` 

二级缓存使用 TransactionalCache 装饰 一级缓存   

## Mapper   
通常，我们不会直接使用 SqlSession.selectOne 这样去查询，因为不方便维护  
大部分时候，我们都用 getMapper 来获取对应 Dao   
```java
  /**
   * 这里调用 configuration 的 getMapper
   * @param type Mapper interface class
   * @param <T>
   * @return
   */
  @Override
  public <T> T getMapper(Class<T> type) {
    return configuration.<T>getMapper(type, this);
  }

  public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    return mapperRegistry.getMapper(type, sqlSession);
  }

    @SuppressWarnings("unchecked")
  public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
    if (mapperProxyFactory == null) {
      throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
    }
    try {
      return mapperProxyFactory.newInstance(sqlSession);
    } catch (Exception e) {
      throw new BindingException("Error getting mapper instance. Cause: " + e, e);
    }
  }
``` 

在之前的解析中，对应的 Mapper ，都put MapperProxyFactory  对象了，当 getMapper 的时候，就是调用 newInstance 方法    
```java
public class MapperProxyFactory<T> {

  private final Class<T> mapperInterface;
  private final Map<Method, MapperMethod> methodCache = new ConcurrentHashMap<>();

  public MapperProxyFactory(Class<T> mapperInterface) {
    this.mapperInterface = mapperInterface;
  }

  public Class<T> getMapperInterface() {
    return mapperInterface;
  }

  public Map<Method, MapperMethod> getMethodCache() {
    return methodCache;
  }

  @SuppressWarnings("unchecked")
  protected T newInstance(MapperProxy<T> mapperProxy) {
    return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
  }

  public T newInstance(SqlSession sqlSession) {
    final MapperProxy<T> mapperProxy = new MapperProxy<>(sqlSession, mapperInterface, methodCache);
    return newInstance(mapperProxy);
  }

}
``` 

可以看到，这里又使用了 JDK 的动态代理，实现类是 MapperProxy 
```java
/**
 * 通过 getMapper 获取的接口
 * 确切的说，直接使用 modelNameMapper 里面的方法的，都是走这个代理
 * @author Clinton Begin
 * @author Eduardo Macarron
 */
public class MapperProxy<T> implements InvocationHandler, Serializable {

  private static final long serialVersionUID = -6424540398559729838L;
  private final SqlSession sqlSession;
  private final Class<T> mapperInterface;
  private final Map<Method, MapperMethod> methodCache;

  public MapperProxy(SqlSession sqlSession, Class<T> mapperInterface, Map<Method, MapperMethod> methodCache) {
    this.sqlSession = sqlSession;
    this.mapperInterface = mapperInterface;
    this.methodCache = methodCache;
  }

  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      if (Object.class.equals(method.getDeclaringClass())) {
        // Object 类里面的方法 toString 之类的
        return method.invoke(this, args);
      } else if (isDefaultMethod(method)) {
        // 默认方法 接口的默认方法
        return invokeDefaultMethod(proxy, method, args);
      }
    } catch (Throwable t) {
      throw ExceptionUtil.unwrapThrowable(t);
    }
    // 或者 MapperMethod 对象，不需要重复创建，有缓存
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    // 调用方法
    return mapperMethod.execute(sqlSession, args);
  }

  private MapperMethod cachedMapperMethod(Method method) {
    return methodCache.computeIfAbsent(method, k -> new MapperMethod(mapperInterface, method, sqlSession.getConfiguration()));
  }

  /**
   * 调用接口的默认方法
   * 这些 API 第一次见
   * @param proxy
   * @param method
   * @param args
   * @return
   * @throws Throwable
   */
  @UsesJava7
  private Object invokeDefaultMethod(Object proxy, Method method, Object[] args)
      throws Throwable {
    final Constructor<MethodHandles.Lookup> constructor = MethodHandles.Lookup.class
        .getDeclaredConstructor(Class.class, int.class);
    if (!constructor.isAccessible()) {
      constructor.setAccessible(true);
    }
    final Class<?> declaringClass = method.getDeclaringClass();
    return constructor
        .newInstance(declaringClass,
            MethodHandles.Lookup.PRIVATE | MethodHandles.Lookup.PROTECTED
                | MethodHandles.Lookup.PACKAGE | MethodHandles.Lookup.PUBLIC)
        .unreflectSpecial(method, declaringClass).bindTo(proxy).invokeWithArguments(args);
  }

  /**
   * Backport of java.lang.reflect.Method#isDefault()
   */
  private boolean isDefaultMethod(Method method) {
    return (method.getModifiers()
        & (Modifier.ABSTRACT | Modifier.PUBLIC | Modifier.STATIC)) == Modifier.PUBLIC
        && method.getDeclaringClass().isInterface();
  }
}
``` 

里面，实际调用在 MapperMethod 里面  
```java
  private final SqlCommand command;
  private final MethodSignature method;

  public MapperMethod(Class<?> mapperInterface, Method method, Configuration config) {
    // 构造 SqlCommand
    this.command = new SqlCommand(config, mapperInterface, method);
    // 构造方法签名
    this.method = new MethodSignature(config, mapperInterface, method);
  }

  public Object execute(SqlSession sqlSession, Object[] args) {
    Object result;
    // 里面还是调用 SqlSession 的方法
    switch (command.getType()) {
      case INSERT: {
        // 增删改的返回类型可以是 int long boolean（包括包装类型）
    	Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.insert(command.getName(), param));
        break;
      }
      case UPDATE: {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.update(command.getName(), param));
        break;
      }
      case DELETE: {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.delete(command.getName(), param));
        break;
      }
      case SELECT:
        // 重点是查询方法的返回处理
        if (method.returnsVoid() && method.hasResultHandler()) {
          // 如果返回为空，或者自定义返回结果处理
          executeWithResultHandler(sqlSession, args);
          result = null;
        } else if (method.returnsMany()) {
          // 返回 list 或者 array
          result = executeForMany(sqlSession, args);
        } else if (method.returnsMap()) {
          // 返回 map
          result = executeForMap(sqlSession, args);
        } else if (method.returnsCursor()) {
          // 返回游标
          result = executeForCursor(sqlSession, args);
        } else {
          // 其他情况 java8 的 Optional
          Object param = method.convertArgsToSqlCommandParam(args);
          result = sqlSession.selectOne(command.getName(), param);
          if (method.returnsOptional() &&
              (result == null || !method.getReturnType().equals(result.getClass()))) {
            result = OptionalUtil.ofNullable(result);
          }
        }
        break;
      case FLUSH:
        result = sqlSession.flushStatements();
        break;
      default:
        throw new BindingException("Unknown execution method for: " + command.getName());
    }
    if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
      throw new BindingException("Mapper method '" + command.getName() 
          + " attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
    }
    return result;
  }

  private <E> Object executeForMany(SqlSession sqlSession, Object[] args) {
    List<E> result;
    Object param = method.convertArgsToSqlCommandParam(args);
    if (method.hasRowBounds()) {
      RowBounds rowBounds = method.extractRowBounds(args);
      result = sqlSession.<E>selectList(command.getName(), param, rowBounds);
    } else {
      result = sqlSession.<E>selectList(command.getName(), param);
    }
    // issue #510 Collections & arrays support
    if (!method.getReturnType().isAssignableFrom(result.getClass())) {
      if (method.getReturnType().isArray()) {
        return convertToArray(result);
      } else {
        return convertToDeclaredCollection(sqlSession.getConfiguration(), result);
      }
    }
    return result;
  }

  private Object rowCountResult(int rowCount) {
    final Object result;
    if (method.returnsVoid()) {
      result = null;
    } else if (Integer.class.equals(method.getReturnType()) || Integer.TYPE.equals(method.getReturnType())) {
      result = rowCount;
    } else if (Long.class.equals(method.getReturnType()) || Long.TYPE.equals(method.getReturnType())) {
      result = (long)rowCount;
    } else if (Boolean.class.equals(method.getReturnType()) || Boolean.TYPE.equals(method.getReturnType())) {
      result = rowCount > 0;
    } else {
      throw new BindingException("Mapper method '" + command.getName() + "' has an unsupported return type: " + method.getReturnType());
    }
    return result;
  }
``` 

其中 SqlCommand 和 MethodSignature 都静态内部类，获取一些辅助信息   
内部还是使用 sqlsession.select update delete insert 方法    
内部细节就不具体分析了，注意一点的就是默认方法直接调用，利用这个特性，当有一个通用的查询接口后（example），其他个性查询可以使用默认方法来实现，不需要写sql，或者在service里面实现   

mapper 接口里面的参数将会被封装 
```java
    public Object convertArgsToSqlCommandParam(Object[] args) {
      return paramNameResolver.getNamedParams(args);
    }

  public Object getNamedParams(Object[] args) {
    final int paramCount = names.size();
    if (args == null || paramCount == 0) {
      return null;
    } else if (!hasParamAnnotation && paramCount == 1) {
      return args[names.firstKey()];
    } else {
      final Map<String, Object> param = new ParamMap<>();
      int i = 0;
      for (Map.Entry<Integer, String> entry : names.entrySet()) {
        param.put(entry.getValue(), args[entry.getKey()]);
        // add generic param names (param1, param2, ...)
        final String genericParamName = GENERIC_NAME_PREFIX + String.valueOf(i + 1);
        // ensure not to overwrite parameter named with @Param
        if (!names.containsValue(genericParamName)) {
          param.put(genericParamName, args[entry.getKey()]);
        }
        i++;
      }
      return param;
    }
  }
``` 

特别注意，两个特殊类型不会在这里面 RowBounds ResultHandler  


## SqlSource  
一般初次构建的类型是 DynamicSqlSource 和 RawSqlSource 
这是不能直接用的。譬如 DynamicSqlSource 里面的标签都还没解析  
所以获取 BoundSql 的时候，开始解析，重新获得 SqlSource  
```java
  @Override
  public BoundSql getBoundSql(Object parameterObject) {
    // 构建 DynamicContext
    DynamicContext context = new DynamicContext(configuration, parameterObject);
    // 这里的 SqlNode 是 MixedSqlNode ，内部包含 SqlNode 的一个list，对list 里面按顺序依次解析，具体的解析过程，可以看对应 Node 里面
    rootSqlNode.apply(context);
    SqlSourceBuilder sqlSourceParser = new SqlSourceBuilder(configuration);
    Class<?> parameterType = parameterObject == null ? Object.class : parameterObject.getClass();
    // 这里的 getBindings 默认带有两个参数 _databaseId 和 _parameter， 动态sql 经过上面的解析后，就变成静态的了，#{} 就换成了 ? 了
    SqlSource sqlSource = sqlSourceParser.parse(context.getSql(), parameterType, context.getBindings());
    // 获取 BoundSql
    BoundSql boundSql = sqlSource.getBoundSql(parameterObject);
    // 传递内部参数
    for (Map.Entry<String, Object> entry : context.getBindings().entrySet()) {
      boundSql.setAdditionalParameter(entry.getKey(), entry.getValue());
    }
    return boundSql;
  }
``` 

可以看到，解析完成获得一个 SqlSource 后，继续调用 getBoundSql，我们看看解析过程 
```java
  public SqlSource parse(String originalSql, Class<?> parameterType, Map<String, Object> additionalParameters) {
    ParameterMappingTokenHandler handler = new ParameterMappingTokenHandler(configuration, parameterType, additionalParameters);
    GenericTokenParser parser = new GenericTokenParser("#{", "}", handler);
    String sql = parser.parse(originalSql);
    return new StaticSqlSource(configuration, sql, handler.getParameterMappings());
  }
``` 
说一句 `rootSqlNode.apply(context);` 这一行执行后，xml 里面的标签已经解析完成了。 
可以看到，解析就是替换 `#{}` 换成 `?` 
```java
    @Override
    public String handleToken(String content) {
      parameterMappings.add(buildParameterMapping(content));
      return "?";
    }

 private ParameterMapping buildParameterMapping(String content) {
      Map<String, String> propertiesMap = parseParameterMapping(content);
      // property 也就是 #{} 里面的内容，字段名
      String property = propertiesMap.get("property");
      Class<?> propertyType;
      // 找到字段类型
      if (metaParameters.hasGetter(property)) { // issue #448 get type from additional params
        propertyType = metaParameters.getGetterType(property);
      } else if (typeHandlerRegistry.hasTypeHandler(parameterType)) {
        propertyType = parameterType;
      } else if (JdbcType.CURSOR.name().equals(propertiesMap.get("jdbcType"))) {
        propertyType = java.sql.ResultSet.class;
      } else if (property == null || Map.class.isAssignableFrom(parameterType)) {
        propertyType = Object.class;
      } else {
        MetaClass metaClass = MetaClass.forClass(parameterType, configuration.getReflectorFactory());
        if (metaClass.hasGetter(property)) {
          propertyType = metaClass.getGetterType(property);
        } else {
          propertyType = Object.class;
        }
      }
      ParameterMapping.Builder builder = new ParameterMapping.Builder(configuration, property, propertyType);
      Class<?> javaType = propertyType;
      String typeHandlerAlias = null;
      // 语法支持，大部分时候啥都没做，一般，我们也只用 #{username} 这样的
      for (Map.Entry<String, String> entry : propertiesMap.entrySet()) {
        String name = entry.getKey();
        String value = entry.getValue();
        if ("javaType".equals(name)) {
          javaType = resolveClass(value);
          builder.javaType(javaType);
        } else if ("jdbcType".equals(name)) {
          builder.jdbcType(resolveJdbcType(value));
        } else if ("mode".equals(name)) {
          builder.mode(resolveParameterMode(value));
        } else if ("numericScale".equals(name)) {
          builder.numericScale(Integer.valueOf(value));
        } else if ("resultMap".equals(name)) {
          builder.resultMapId(value);
        } else if ("typeHandler".equals(name)) {
          typeHandlerAlias = value;
        } else if ("jdbcTypeName".equals(name)) {
          builder.jdbcTypeName(value);
        } else if ("property".equals(name)) {
          // Do Nothing
        } else if ("expression".equals(name)) {
          throw new BuilderException("Expression based parameters are not supported yet");
        } else {
          throw new BuilderException("An invalid property '" + name + "' was found in mapping #{" + content + "}.  Valid properties are " + parameterProperties);
        }
      }
      if (typeHandlerAlias != null) {
        builder.typeHandler(resolveTypeHandler(javaType, typeHandlerAlias));
      }
      return builder.build();
    }
``` 

这里会生成参数映射。  
最后返回的是 `StaticSqlSource`  

`parseParameterMapping` 方法解析，可以参见单元测试内容  
```java
@Test
  public void simpleProperty() {
    Map<String, String> result = new ParameterExpression("id");
    Assert.assertEquals(1, result.size());
    Assert.assertEquals("id", result.get("property"));
  }

  public void propertyWithSpacesInside() {
    Map<String, String> result = new ParameterExpression(" with spaces ");
    Assert.assertEquals(1, result.size());
    Assert.assertEquals("with spaces", result.get("property"));
  }

  @Test
  public void simplePropertyWithOldStyleJdbcType() {
    Map<String, String> result = new ParameterExpression("id:VARCHAR");
    Assert.assertEquals(2, result.size());
    Assert.assertEquals("id", result.get("property"));
    Assert.assertEquals("VARCHAR", result.get("jdbcType"));
  }

  @Test
  public void oldStyleJdbcTypeWithExtraWhitespaces() {
    Map<String, String> result = new ParameterExpression(" id :  VARCHAR ");
    Assert.assertEquals(2, result.size());
    Assert.assertEquals("id", result.get("property"));
    Assert.assertEquals("VARCHAR", result.get("jdbcType"));
  }

  @Test
  public void expressionWithOldStyleJdbcType() {
    Map<String, String> result = new ParameterExpression("(id.toString()):VARCHAR");
    Assert.assertEquals(2, result.size());
    Assert.assertEquals("id.toString()", result.get("expression"));
    Assert.assertEquals("VARCHAR", result.get("jdbcType"));
  }

  @Test
  public void simplePropertyWithOneAttribute() {
    Map<String, String> result = new ParameterExpression("id,name=value");
    Assert.assertEquals(2, result.size());
    Assert.assertEquals("id", result.get("property"));
    Assert.assertEquals("value", result.get("name"));
  }

  @Test
  public void expressionWithOneAttribute() {
    Map<String, String> result = new ParameterExpression("(id.toString()),name=value");
    Assert.assertEquals(2, result.size());
    Assert.assertEquals("id.toString()", result.get("expression"));
    Assert.assertEquals("value", result.get("name"));
  }

  @Test
  public void simplePropertyWithManyAttributes() {
    Map<String, String> result = new ParameterExpression("id, attr1=val1, attr2=val2, attr3=val3");
    Assert.assertEquals(4, result.size());
    Assert.assertEquals("id", result.get("property"));
    Assert.assertEquals("val1", result.get("attr1"));
    Assert.assertEquals("val2", result.get("attr2"));
    Assert.assertEquals("val3", result.get("attr3"));
  }

  @Test
  public void expressionWithManyAttributes() {
    Map<String, String> result = new ParameterExpression("(id.toString()), attr1=val1, attr2=val2, attr3=val3");
    Assert.assertEquals(4, result.size());
    Assert.assertEquals("id.toString()", result.get("expression"));
    Assert.assertEquals("val1", result.get("attr1"));
    Assert.assertEquals("val2", result.get("attr2"));
    Assert.assertEquals("val3", result.get("attr3"));
  }

  @Test
  public void simplePropertyWithOldStyleJdbcTypeAndAttributes() {
    Map<String, String> result = new ParameterExpression("id:VARCHAR, attr1=val1, attr2=val2");
    Assert.assertEquals(4, result.size());
    Assert.assertEquals("id", result.get("property"));
    Assert.assertEquals("VARCHAR", result.get("jdbcType"));
    Assert.assertEquals("val1", result.get("attr1"));
    Assert.assertEquals("val2", result.get("attr2"));
  }

  @Test
  public void simplePropertyWithSpaceAndManyAttributes() {
    Map<String, String> result = new ParameterExpression("user name, attr1=val1, attr2=val2, attr3=val3");
    Assert.assertEquals(4, result.size());
    Assert.assertEquals("user name", result.get("property"));
    Assert.assertEquals("val1", result.get("attr1"));
    Assert.assertEquals("val2", result.get("attr2"));
    Assert.assertEquals("val3", result.get("attr3"));
  }

  @Test
  public void shouldIgnoreLeadingAndTrailingSpaces() {
    Map<String, String> result = new ParameterExpression(" id , jdbcType =  VARCHAR,  attr1 = val1 ,  attr2 = val2 ");
    Assert.assertEquals(4, result.size());
    Assert.assertEquals("id", result.get("property"));
    Assert.assertEquals("VARCHAR", result.get("jdbcType"));
    Assert.assertEquals("val1", result.get("attr1"));
    Assert.assertEquals("val2", result.get("attr2"));
  }

  @Test
  public void invalidOldJdbcTypeFormat() {
    try {
      new ParameterExpression("id:");
      Assert.fail();
    } catch (BuilderException e) {
      Assert.assertTrue(e.getMessage().contains("Parsing error in {id:} in position 3"));
    }
  }

  @Test
  public void invalidJdbcTypeOptUsingExpression() {
    try {
      new ParameterExpression("(expression)+");
      Assert.fail();
    } catch (BuilderException e) {
      Assert.assertTrue(e.getMessage().contains("Parsing error in {(expression)+} in position 12"));
    }
  }
``` 

这上面，也就是  `#{}` 的用法了。  
