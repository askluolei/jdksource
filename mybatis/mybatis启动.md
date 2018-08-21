[TOC]

# mybatis
Mybatis 启动流程解析  

[mybatis分析](https://blog.csdn.net/luanlouis/article/details/40422941)
上面这篇文章的分析挺好的（主要是图画的好，不会画图，很尴尬，盗几张图）。  

从MyBatis代码实现的角度来看，MyBatis的主要的核心部件有以下几个：

* SqlSessionFactory    SqlSession 工厂，全局唯一（单库情况）
* Configuration        MyBatis所有的配置信息都维持在Configuration对象之中。
* SqlSession            作为MyBatis工作的主要顶层API，表示和数据库交互的会话，完成必要数据库增删改查功能
* Executor              MyBatis执行器，是MyBatis 调度的核心，负责SQL语句的生成和查询缓存的维护
* StatementHandler   封装了JDBC Statement操作，负责对JDBC statement 的操作，如设置参数、将Statement结果集转换成List集合。
* ParameterHandler   负责对用户传递的参数转换成JDBC Statement 所需要的参数，
* ResultSetHandler    负责将JDBC返回的ResultSet结果集对象转换成List类型的集合；
* TypeHandler          负责java数据类型和jdbc数据类型之间的映射和转换
* MappedStatement   MappedStatement维护了一条<select|update|delete|insert>节点的封装， 
* SqlSource            负责根据用户传递的parameterObject，动态地生成SQL语句，将信息封装到BoundSql对象中，并返回
* BoundSql             表示动态生成的SQL语句以及相应的参数信息


## xml启动
mybatis 的启动单独使用时通过一个核心配置xml + n个mapper.xml 的  
与 springboot 集成的时候，通常只剩下 n个mapper.xml 了（不用xml也行，但是通常还是会使用xml来定义复杂一点的语句）。   
我们还是从xml分析   

**基础概念**
1. SqlSessionFactory
SqlSession 工厂，全局唯一（单数据源的情况下），启动实际上就是构建 SqlSessionFactory（但是配置信息全部在 Configuration）
2. SqlSession
跟数据库交互的操作都是基于这个类（即使是 Dao 接口，也是 getMapper 获取到的动态代理）
3. Configuration
核心配置类，所有 mybatis 相关的配置和解析出来的信息都在这里

**入门**
使用xml
```java
String resource = "org/mybatis/example/mybatis-config.xml";
InputStream inputStream = Resources.getResourceAsStream(resource);
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
``` 

不使用xml
```java
DataSource dataSource = BlogDataSourceFactory.getBlogDataSource();
TransactionFactory transactionFactory = new JdbcTransactionFactory();
Environment environment = new Environment("development", transactionFactory, dataSource);
Configuration configuration = new Configuration(environment);
configuration.addMapper(BlogMapper.class);
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(configuration);
```

上面就是启动，获取 SqlSessionFactory 实例后就可以继续获取 SqlSession 进行数据操作了
**数据操作**
```java
SqlSession session = sqlSessionFactory.openSession();
try {
  Blog blog = (Blog) session.selectOne("org.mybatis.example.BlogMapper.selectBlog", 101);
} finally {
  session.close();
}

SqlSession session = sqlSessionFactory.openSession();
try {
  BlogMapper mapper = session.getMapper(BlogMapper.class);
  Blog blog = mapper.selectBlog(101);
} finally {
  session.close();
}
``` 
上面两种操作方式，通常使用的是第二种，并且这部分操作都会进行封装（跟spring 集成的时候），在业务里面直接获取 `Dao` 接口进行操作（不需要实现类）. 

入口分析从 SqlSessionFactoryBuilder 开始

### SqlSessionFactoryBuilder
```java
public SqlSessionFactory build(Reader reader, String environment, Properties properties) {
    try {
      XMLConfigBuilder parser = new XMLConfigBuilder(reader, environment, properties);
      return build(parser.parse());
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error building SqlSession.", e);
    } finally {
      ErrorContext.instance().reset();
      try {
        reader.close();
      } catch (IOException e) {
        // Intentionally ignore. Prefer previous error.
      }
    }
  }
``` 

重点就是这个方法，还有一个重载方法，参数类型是 InputStream，根据 api 构建，就可以猜到 XMLConfigBuilder 经过 parse 后，就是 Configuration 实例了

获取 Configuration 实例后 

```java
public SqlSessionFactory build(Configuration config) {
    return new DefaultSqlSessionFactory(config);
}

// 构建 SqlSessionFactory ，只是将 Configuration 对象传过来的
public DefaultSqlSessionFactory(Configuration configuration) {
    this.configuration = configuration;
}
```

可以看到 SqlSessionFactory 的实现就是 DefaultSqlSessionFactory  

接下来我们看看解析细节

### XMLConfigBuilder
这里的就是就是核心配置文件  
示例
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
  PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
  "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
  <environments default="development">
    <environment id="development">
      <transactionManager type="JDBC"/>
      <dataSource type="POOLED">
        <property name="driver" value="${driver}"/>
        <property name="url" value="${url}"/>
        <property name="username" value="${username}"/>
        <property name="password" value="${password}"/>
      </dataSource>
    </environment>
  </environments>
  <mappers>
    <mapper resource="org/mybatis/example/BlogMapper.xml"/>
  </mappers>
</configuration>
``` 

```java
// 当parse 完成后，返回 configuration，这个类是核心类，有所有的信息
  public Configuration parse() {
    if (parsed) {
      throw new BuilderException("Each XMLConfigBuilder can only be used once.");
    }
    parsed = true;
    parseConfiguration(parser.evalNode("/configuration"));
    return configuration;
  }
``` 

我们不去细看xml解析语法（在构造这个实例的时候就已经解析成一个 Dom 对象了）  
这里重要的是 `parseConfiguration(parser.evalNode("/configuration"));` ,直接就能猜到是解析 configuration 节点    

```java
// 解析 xml 配置文件
  // 解析完成后，所有的信息都在 Configuration 对象上
  private void parseConfiguration(XNode root) {
    try {
      /**
       * 解析顺序,所以配置文件的顺序，通常来说也是这样的
       * 1. properties 自定义配置项，resource url 可以设置外置的 properties 文件，也可以在子元素里面设置，配置将用在接下来的解析，也会设置在 configuration 类里面（configuration是核心类，所有的东西最终都在这里）
       * 2. settings 系统配置项，一些开关，全局配置之类的
       * 3. typeAliases
       * 4. plugins
       * 5. objectFactory
       * 6. objectWrapperFactory
       * 7. reflectorFactory
       * 8. environments
       * 9. databaseIdProvider
       * 10. typeHandlers
       * 11. mappers
       */
      //issue #117 read properties first
      // 解析 properties
      /**
       * <properties resource="org/mybatis/example/config.properties">
       *   <property name="username" value="dev_user"/>
       *   <property name="password" value="F2Fa3!33TYyg"/>
       * </properties>
       */
      propertiesElement(root.evalNode("properties"));
      // 解析settings
      /**
       * <settings>
       *   <setting name="cacheEnabled" value="true"/>
       *   <setting name="lazyLoadingEnabled" value="true"/>
       *   <setting name="multipleResultSetsEnabled" value="true"/>
       *   <setting name="useColumnLabel" value="true"/>
       *   <setting name="useGeneratedKeys" value="false"/>
       *   <setting name="autoMappingBehavior" value="PARTIAL"/>
       *   <setting name="autoMappingUnknownColumnBehavior" value="WARNING"/>
       *   <setting name="defaultExecutorType" value="SIMPLE"/>
       *   <setting name="defaultStatementTimeout" value="25"/>
       *   <setting name="defaultFetchSize" value="100"/>
       *   <setting name="safeRowBoundsEnabled" value="false"/>
       *   <setting name="mapUnderscoreToCamelCase" value="false"/>
       *   <setting name="localCacheScope" value="SESSION"/>
       *   <setting name="jdbcTypeForNull" value="OTHER"/>
       *   <setting name="lazyLoadTriggerMethods" value="equals,clone,hashCode,toString"/>
       * </settings>
       */
      Properties settings = settingsAsProperties(root.evalNode("settings"));
      // 看看 settings 里面是否设置类 Vfs 实现类（大部分不需要管，只是部署在特殊的容器里面可能需要扩展，VFS 就是在不同容器里面获取资源的方式和路径不同）
      loadCustomVfs(settings);
      // 别名设置，这个经常使用，用简单类名当做全类名的别名
      // 可以直接指定 package   <package name="domain.blog"/>
      // 也可以一个一个指定 <typeAlias alias="Tag" type="domain.blog.Tag"/>
      typeAliasesElement(root.evalNode("typeAliases"));
      // 解析插件，插件是用来扩展 mybatis 的地方，比如分页插件等
      // 注册的插件添加到 configuration 上，注意，这里的 property 不能用上面的 properties 里面定义的
      /**
       * <plugins>
       *   <plugin interceptor="org.mybatis.example.ExamplePlugin">
       *     <property name="someProperty" value="100"/>
       *   </plugin>
       * </plugins>
       */
      pluginElement(root.evalNode("plugins"));
      // 解析objectFactory
      // MyBatis 每次创建结果对象的新实例时，它都会使用一个对象工厂（ObjectFactory）实例来完成。
      // 默认的对象工厂需要做的仅仅是实例化目标类，要么通过默认构造方法，要么在参数映射存在的时候通过参数构造方法来实例化。
      // 如果想覆盖对象工厂的默认行为，则可以通过创建自己的对象工厂来实现
      /**
       * <objectFactory type="org.mybatis.example.ExampleObjectFactory">
       *   <property name="someProperty" value="100"/>
       * </objectFactory>
       */
      objectFactoryElement(root.evalNode("objectFactory"));
      // 解析 objectWrapperFactory
      // 这个类就是用来包装和判断对象是否包装过，默认是没有包装的，后面看看这里有在那里
      /**
       * <objectWrapperFactory type="org..."></objectWrapperFactory>
       */
      objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
      // 解析 reflectorFactory
      // 就是用来包装 Class 对象，方便反射调用，默认即可
      /**
       * <reflectorFactory type="org..."></reflectorFactory>
       */
      reflectorFactoryElement(root.evalNode("reflectorFactory"));
      // 这里设置 settings ，没有就取默认值
      settingsElement(settings);
      // read it after objectFactory and objectWrapperFactory issue #631
      // 解析 environments
      // 这里配置事物，数据源
      // property 可以直接填直接值，可以使用上面properties 加载的
      // 注意，可以使用多个 environment，用在不同环境，但是只有 default 指定的环境生效
      /**
       * <environments default="development">
       *   <environment id="development">
       *     <transactionManager type="JDBC">
       *       <property name="..." value="..."/>
       *     </transactionManager>
       *     <dataSource type="POOLED">
       *       <property name="driver" value="${driver}"/>
       *       <property name="url" value="${url}"/>
       *       <property name="username" value="${username}"/>
       *       <property name="password" value="${password}"/>
       *     </dataSource>
       *   </environment>
       * </environments>
       */
      environmentsElement(root.evalNode("environments"));
      // 解析 databaseIdProvider
      //MyBatis 可以根据不同的数据库厂商执行不同的语句，这种多厂商的支持是基于映射语句中的 databaseId 属性。
      // MyBatis 会加载不带 databaseId 属性和带有匹配当前数据库 databaseId 属性的所有语句。
      // 如果同时找到带有 databaseId 和不带 databaseId 的相同语句，则后者会被舍弃
      /**
       * <databaseIdProvider type="DB_VENDOR">
       *   <property name="SQL Server" value="sqlserver"/>
       *   <property name="DB2" value="db2"/>
       *   <property name="Oracle" value="oracle" />
       * </databaseIdProvider>
       */
      databaseIdProviderElement(root.evalNode("databaseIdProvider"));
      // 类型处理
      // 可以一个一个指定，也可以指定一个package
      // 要有 注解标记 @MappedJdbcTypes(JdbcType.VARCHAR)
      // 这里是 java 类型和 数据库类型转换的类
      /**
       *<typeHandlers>
       *   <package name="org.mybatis.example"/>
       * </typeHandlers>
       *
       * <typeHandlers>
       *   <typeHandler handler="org.mybatis.example.ExampleTypeHandler"/>
       * </typeHandlers>
       */
      typeHandlerElement(root.evalNode("typeHandlers"));
      // 解析 mappers
      /**
       * 既然 MyBatis 的行为已经由上述元素配置完了，我们现在就要定义 SQL 映射语句了。但是首先我们需要告诉 MyBatis 到哪里去找到这些语句。
       * Java 在自动查找这方面没有提供一个很好的方法，所以最佳的方式是告诉 MyBatis 到哪里去找映射文件。
       * 你可以使用相对于类路径的资源引用， 或完全限定资源定位符
       */
      /**
       * <!-- 使用相对于类路径的资源引用 -->
       * <mappers>
       *   <mapper resource="org/mybatis/builder/AuthorMapper.xml"/>
       *   <mapper resource="org/mybatis/builder/BlogMapper.xml"/>
       *   <mapper resource="org/mybatis/builder/PostMapper.xml"/>
       * </mappers>
       *
       * <!-- 使用完全限定资源定位符（URL） -->
       * <mappers>
       *   <mapper url="file:///var/mappers/AuthorMapper.xml"/>
       *   <mapper url="file:///var/mappers/BlogMapper.xml"/>
       *   <mapper url="file:///var/mappers/PostMapper.xml"/>
       * </mappers>
       *
       * <!-- 使用映射器接口实现类的完全限定类名 -->
       * <mappers>
       *   <mapper class="org.mybatis.builder.AuthorMapper"/>
       *   <mapper class="org.mybatis.builder.BlogMapper"/>
       *   <mapper class="org.mybatis.builder.PostMapper"/>
       * </mappers>
       *
       * <!-- 将包内的映射器接口实现全部注册为映射器 -->
       * <mappers>
       *   <package name="org.mybatis.builder"/>
       * </mappers>
       */
      mapperElement(root.evalNode("mappers"));
    } catch (Exception e) {
      throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
    }
  }
``` 

这上面已经有注释了，除了解析 mappers 里面还要做特别处理外，其他的解析都不复杂，不继续向下看。   
这里面，我们用的多的就是 别名，类型处理，全局设置，自定义配置，环境（事务管理和数据源配置），如果多数据库支持那么可以使用数据库标识（databaseId），如果需要做扩展（分页）可以注册
插件（插件的实现原理我们后面具体分析）。    
所有的解析都是将xml配置信息丢到 Configuration 实例里面。    

接下来我们具体讲一下 mappers 的解析（sql 执行相关） 
```java
private void mapperElement(XNode parent) throws Exception {
    if (parent != null) {
      for (XNode child : parent.getChildren()) {
        if ("package".equals(child.getName())) {
          // package 的直接 Mapper
          String mapperPackage = child.getStringAttribute("name");
          configuration.addMappers(mapperPackage);
        } else {
          String resource = child.getStringAttribute("resource");
          String url = child.getStringAttribute("url");
          String mapperClass = child.getStringAttribute("class");
          if (resource != null && url == null && mapperClass == null) {
            // 解析 mapper 的 xml
            ErrorContext.instance().resource(resource);
            InputStream inputStream = Resources.getResourceAsStream(resource);
            XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource, configuration.getSqlFragments());
            mapperParser.parse();
          } else if (resource == null && url != null && mapperClass == null) {
            // 同上，不过是根据url获取
            ErrorContext.instance().resource(url);
            InputStream inputStream = Resources.getUrlAsStream(url);
            XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, url, configuration.getSqlFragments());
            mapperParser.parse();
          } else if (resource == null && url == null && mapperClass != null) {
            // 定义的接口，那就直接 addMapper
            Class<?> mapperInterface = Resources.classForName(mapperClass);
            configuration.addMapper(mapperInterface);
          } else {
            throw new BuilderException("A mapper element may only specify a url, resource or class, but not more than one.");
          }
        }
      }
    }
  }
``` 

mappers 里面我们可以直接指定包名（该包下的接口注册），单接口注册（class），或者 mapper.xml 文件路径（即使注册了接口，我们通常还是会写这个文件，注册接口后
mybatis也会默认找一下同名mapper.xml 文件）  
其实这里主要分为 xml 解析 和 接口注册，我们先看xml解析过程（不详细分析） 
顺带说一句 `ErrorContext` 这个类是记录解析或者执行的信息，等出席异常的时候方便定为问题。    

**xml**
```java
// 构造时候的 xml 解析，忽略，只要知道要进行xml格式检查以及最终的 dom 节点就行了。
public XMLMapperBuilder(InputStream inputStream, Configuration configuration, String resource, Map<String, XNode> sqlFragments) {
    this(new XPathParser(inputStream, true, configuration.getVariables(), new XMLMapperEntityResolver()),
        configuration, resource, sqlFragments);
  }

  private XMLMapperBuilder(XPathParser parser, Configuration configuration, String resource, Map<String, XNode> sqlFragments) {
    super(configuration);
    this.builderAssistant = new MapperBuilderAssistant(configuration, resource);
    this.parser = parser;
    this.sqlFragments = sqlFragments;
    this.resource = resource;
  }

public void parse() {
    if (!configuration.isResourceLoaded(resource)) {
      // 解析 xml
      configurationElement(parser.evalNode("/mapper"));
      // 标记为已经解析过的资源
      configuration.addLoadedResource(resource);
      // 这里尝试根据 namespace 找到对应接口，注册 Mapper，所以，namespace 建议跟对应接口名保持一致，id 跟方法名保持一致
      bindMapperForNamespace();
    }

    parsePendingResultMaps();
    parsePendingCacheRefs();
    parsePendingStatements();
  }
``` 

解析的时候先解析xml，如果已经解析过了忽略   
```java
  private void configurationElement(XNode context) {
    try {
      // namespace
      String namespace = context.getStringAttribute("namespace");
      if (namespace == null || namespace.equals("")) {
        throw new BuilderException("Mapper's namespace cannot be empty");
      }
      // 设置当前 namespace
      builderAssistant.setCurrentNamespace(namespace);
      // 解析对应标签
      // 公用一个缓存对象的标签
      cacheRefElement(context.evalNode("cache-ref"));
      // mapper 的缓存设置
      cacheElement(context.evalNode("cache"));
      // 参数map，这个可能以后会废弃，就不看了
      parameterMapElement(context.evalNodes("/mapper/parameterMap"));
      // 结果集map，这个经常用到，特别是自定义返回
      resultMapElements(context.evalNodes("/mapper/resultMap"));
      // sql 标签，公用的sql语句
      sqlElement(context.evalNodes("/mapper/sql"));
      // 增删改查语句
      buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
    } catch (Exception e) {
      throw new BuilderException("Error parsing Mapper XML. The XML location is '" + resource + "'. Cause: " + e, e);
    }
  }
``` 

缓存相关的2个解析很简单 
```java
private void cacheRefElement(XNode context) {
    if (context != null) {
      // 先配置类添加这个配置
      configuration.addCacheRef(builderAssistant.getCurrentNamespace(), context.getStringAttribute("namespace"));
      CacheRefResolver cacheRefResolver = new CacheRefResolver(builderAssistant, context.getStringAttribute("namespace"));
      try {
        // 这里的解析，只是看看引用的namespace 在前面解析没（就是有对应namespace的缓存配置不），如果没解析那就添加一些未解析，等后面解析
        cacheRefResolver.resolveCacheRef();
      } catch (IncompleteElementException e) {
        configuration.addIncompleteCacheRef(cacheRefResolver);
      }
    }
  }

  private void cacheElement(XNode context) throws Exception {
    if (context != null) {
      // type 可以是别名（内建的cache），也可以是cache类全名（自定义cache可以用）
      String type = context.getStringAttribute("type", "PERPETUAL");
      // 根据别名获取class，如果存在，就返回对应 class，如果不存在，就加载类名
      Class<? extends Cache> typeClass = typeAliasRegistry.resolveAlias(type);
      // 回收策略,这里的 Cache 是对上面的包装
      String eviction = context.getStringAttribute("eviction", "LRU");
      Class<? extends Cache> evictionClass = typeAliasRegistry.resolveAlias(eviction);
      // 刷新间隔
      Long flushInterval = context.getLongAttribute("flushInterval");
      // 对象数量
      Integer size = context.getIntAttribute("size");
      // 只读
      boolean readWrite = !context.getBooleanAttribute("readOnly", false);
      // 阻塞
      boolean blocking = context.getBooleanAttribute("blocking", false);
      // 其他自定义属性
      Properties props = context.getChildrenAsProperties();
      // 添加缓存实例到 配置中
      builderAssistant.useNewCache(typeClass, evictionClass, flushInterval, size, readWrite, blocking, props);
    }
  }
``` 

所谓 cacheRef 代表使用另一个mapper 里面的缓存配置（公用一个）
cache 表示本mapper 使用的配置（所以，当开启了二级缓存的时候，对应mapper里面也要配置才起作用）   

parameterMap 标签用的比较少，就不看了   
sql 标签是用来写公用sql 的（譬如表里面的字段名）    
resultMap 定义返回映射（常用，特别是复杂映射）
然后就是增删改查解析    

resultMap 解析出来的是 ResultMapping list，其中每一个字段的映射都对应一个 ResultMapping 实例，如果有嵌套，那么 ResultMapping 是有组合的 
具体解析过程就不看了，解析小复杂，过于细节，跳过。  

sql 标签，解析出来的是可以共用的 sql 片段，比较简单，跳过
重点在实际sql解析 增删改查  
```java
private void buildStatementFromContext(List<XNode> list) {
    if (configuration.getDatabaseId() != null) {
      buildStatementFromContext(list, configuration.getDatabaseId());
    }
    buildStatementFromContext(list, null);
  }

  private void buildStatementFromContext(List<XNode> list, String requiredDatabaseId) {
    for (XNode context : list) {
      final XMLStatementBuilder statementParser = new XMLStatementBuilder(configuration, builderAssistant, context, requiredDatabaseId);
      try {
        statementParser.parseStatementNode();
      } catch (IncompleteElementException e) {
        configuration.addIncompleteStatement(statementParser);
      }
    }
  }

  public void parseStatementNode() {
    String id = context.getStringAttribute("id");
    String databaseId = context.getStringAttribute("databaseId");

    // 数据库不匹配，不解析
    if (!databaseIdMatchesCurrent(id, databaseId, this.requiredDatabaseId)) {
      return;
    }

    // 解析属性
    Integer fetchSize = context.getIntAttribute("fetchSize");
    Integer timeout = context.getIntAttribute("timeout");
    String parameterMap = context.getStringAttribute("parameterMap");
    String parameterType = context.getStringAttribute("parameterType");
    Class<?> parameterTypeClass = resolveClass(parameterType);
    String resultMap = context.getStringAttribute("resultMap");
    String resultType = context.getStringAttribute("resultType");
    String lang = context.getStringAttribute("lang");
    LanguageDriver langDriver = getLanguageDriver(lang);

    Class<?> resultTypeClass = resolveClass(resultType);
    String resultSetType = context.getStringAttribute("resultSetType");
    StatementType statementType = StatementType.valueOf(context.getStringAttribute("statementType", StatementType.PREPARED.toString()));
    ResultSetType resultSetTypeEnum = resolveResultSetType(resultSetType);

    String nodeName = context.getNode().getNodeName();
    SqlCommandType sqlCommandType = SqlCommandType.valueOf(nodeName.toUpperCase(Locale.ENGLISH));
    boolean isSelect = sqlCommandType == SqlCommandType.SELECT;
    boolean flushCache = context.getBooleanAttribute("flushCache", !isSelect);
    boolean useCache = context.getBooleanAttribute("useCache", isSelect);
    boolean resultOrdered = context.getBooleanAttribute("resultOrdered", false);

    // Include Fragments before parsing
    XMLIncludeTransformer includeParser = new XMLIncludeTransformer(configuration, builderAssistant);
    includeParser.applyIncludes(context.getNode());

    // Parse selectKey after includes and remove them.
    processSelectKeyNodes(id, parameterTypeClass, langDriver);
    
    // 这里默认的是 Xml langDriver 获取 sqlSource
    // Parse the SQL (pre: <selectKey> and <include> were parsed and removed)
    SqlSource sqlSource = langDriver.createSqlSource(configuration, context, parameterTypeClass);
    String resultSets = context.getStringAttribute("resultSets");
    String keyProperty = context.getStringAttribute("keyProperty");
    String keyColumn = context.getStringAttribute("keyColumn");
    KeyGenerator keyGenerator;
    String keyStatementId = id + SelectKeyGenerator.SELECT_KEY_SUFFIX;
    keyStatementId = builderAssistant.applyCurrentNamespace(keyStatementId, true);
    if (configuration.hasKeyGenerator(keyStatementId)) {
      keyGenerator = configuration.getKeyGenerator(keyStatementId);
    } else {
      keyGenerator = context.getBooleanAttribute("useGeneratedKeys",
          configuration.isUseGeneratedKeys() && SqlCommandType.INSERT.equals(sqlCommandType))
          ? Jdbc3KeyGenerator.INSTANCE : NoKeyGenerator.INSTANCE;
    }

    // 解析出来的是 MappedStatement ，key 是 namespace
    builderAssistant.addMappedStatement(id, sqlSource, statementType, sqlCommandType,
        fetchSize, timeout, parameterMap, parameterTypeClass, resultMap, resultTypeClass,
        resultSetTypeEnum, flushCache, useCache, resultOrdered, 
        keyGenerator, keyProperty, keyColumn, databaseId, langDriver, resultSets);
  }
``` 

不深入分析细节，观察整个流程    

解析了看看 Mapper 的注册（接口）    

**接口**
```java
// 定义的接口，那就直接 addMapper
Class<?> mapperInterface = Resources.classForName(mapperClass);
configuration.addMapper(mapperInterface);

public <T> void addMapper(Class<T> type) {
    mapperRegistry.addMapper(type);
  }

public <T> void addMapper(Class<T> type) {
    if (type.isInterface()) {
      if (hasMapper(type)) {
        throw new BindingException("Type " + type + " is already known to the MapperRegistry.");
      }
      boolean loadCompleted = false;
      try {
        knownMappers.put(type, new MapperProxyFactory<T>(type));
        // It's important that the type is added before the parser is run
        // otherwise the binding may automatically be attempted by the
        // mapper parser. If the type is already known, it won't try.
        MapperAnnotationBuilder parser = new MapperAnnotationBuilder(config, type);
        parser.parse();
        loadCompleted = true;
      } finally {
        if (!loadCompleted) {
          knownMappers.remove(type);
        }
      }
    }
  }
``` 

这里，只能注册接口，不能重复注册，添加的是 MapperProxyFactory，到 getMapper 的时候返回的是代理，这个后面分析。  
然后解析 MapperAnnotationBuilder，解析还是解析 SqlSource 和 相关属性的解析  

```java
public void parse() {
    // 注解解析
    String resource = type.toString();
    // 如果对应接口的xml资源未解析（同名。xml，非必须），先去解析
    if (!configuration.isResourceLoaded(resource)) {
      // 加载xml,可能没这个资源，无所谓
      loadXmlResource();
      // 设置为已经加载的资源
      configuration.addLoadedResource(resource);
      // 设置当前解析的 namespace
      assistant.setCurrentNamespace(type.getName());
      // 解析缓存注解配置 @CacheNamespace
      parseCache();
      // 解析共享缓存注解配置 @CacheNamespaceRef
      parseCacheRef();
      // 每个方法单独解析
      Method[] methods = type.getMethods();
      for (Method method : methods) {
        try {
          // issue #237
          // 泛型擦除的原因，要过滤掉桥接方法
          // 什么是泛型擦除，就是 <T> 如果指定泛型 String，实际上会有两个方法 一个 Object 一个 String ，Object 那个方法就是桥接方法
          if (!method.isBridge()) {
            // 解析方法上的注解了
            parseStatement(method);
          }
        } catch (IncompleteElementException e) {
          configuration.addIncompleteMethod(new MethodResolver(this, method));
        }
      }
    }
    parsePendingMethods();
  }

void parseStatement(Method method) {
    // 获取参数类型, RowBounds ResultHandler 相当于系统参数，排除这两个类型的参数，如果是多参数，那么就是内部的 ParamMap 包装，否则就是该参数类型
    Class<?> parameterTypeClass = getParameterType(method);
    // 这个通常不会该，默认就是 xml  PS：什么是 LanguageDriver，mybatis 的本质就是将sql写在文本，然后加载成 SqlSource，默认是写在xml，也可以写 Velocity 模板，或者原生 sql
    LanguageDriver languageDriver = getLanguageDriver(method);
    // 这里就是解析 Sql 了，通过解析注解
    SqlSource sqlSource = getSqlSourceFromAnnotations(method, parameterTypeClass, languageDriver);
    if (sqlSource != null) {
      // 设置参数（系统参数）
      Options options = method.getAnnotation(Options.class);
      final String mappedStatementId = type.getName() + "." + method.getName();
      Integer fetchSize = null;
      Integer timeout = null;
      StatementType statementType = StatementType.PREPARED;
      ResultSetType resultSetType = ResultSetType.FORWARD_ONLY;
      SqlCommandType sqlCommandType = getSqlCommandType(method);
      boolean isSelect = sqlCommandType == SqlCommandType.SELECT;
      boolean flushCache = !isSelect;
      boolean useCache = isSelect;

      KeyGenerator keyGenerator;
      String keyProperty = null;
      String keyColumn = null;
      if (SqlCommandType.INSERT.equals(sqlCommandType) || SqlCommandType.UPDATE.equals(sqlCommandType)) {
        // first check for SelectKey annotation - that overrides everything else
        // 字段生成
        SelectKey selectKey = method.getAnnotation(SelectKey.class);
        if (selectKey != null) {
          keyGenerator = handleSelectKeyAnnotation(selectKey, mappedStatementId, getParameterType(method), languageDriver);
          keyProperty = selectKey.keyProperty();
        } else if (options == null) {
          keyGenerator = configuration.isUseGeneratedKeys() ? Jdbc3KeyGenerator.INSTANCE : NoKeyGenerator.INSTANCE;
        } else {
          keyGenerator = options.useGeneratedKeys() ? Jdbc3KeyGenerator.INSTANCE : NoKeyGenerator.INSTANCE;
          keyProperty = options.keyProperty();
          keyColumn = options.keyColumn();
        }
      } else {
        keyGenerator = NoKeyGenerator.INSTANCE;
      }

      // 设置参数
      if (options != null) {
        if (FlushCachePolicy.TRUE.equals(options.flushCache())) {
          flushCache = true;
        } else if (FlushCachePolicy.FALSE.equals(options.flushCache())) {
          flushCache = false;
        }
        useCache = options.useCache();
        fetchSize = options.fetchSize() > -1 || options.fetchSize() == Integer.MIN_VALUE ? options.fetchSize() : null; //issue #348
        timeout = options.timeout() > -1 ? options.timeout() : null;
        statementType = options.statementType();
        resultSetType = options.resultSetType();
      }

      String resultMapId = null;
      // resultMap
      ResultMap resultMapAnnotation = method.getAnnotation(ResultMap.class);
      if (resultMapAnnotation != null) {
        String[] resultMaps = resultMapAnnotation.value();
        StringBuilder sb = new StringBuilder();
        for (String resultMap : resultMaps) {
          if (sb.length() > 0) {
            sb.append(",");
          }
          sb.append(resultMap);
        }
        resultMapId = sb.toString();
      } else if (isSelect) {
        // 生成 resultMap
        resultMapId = parseResultMap(method);
      }

      assistant.addMappedStatement(
          mappedStatementId,
          sqlSource,
          statementType,
          sqlCommandType,
          fetchSize,
          timeout,
          // ParameterMapID
          null,
          parameterTypeClass,
          resultMapId,
          getReturnType(method),
          resultSetType,
          flushCache,
          useCache,
          // TODO gcode issue #577
          false,
          keyGenerator,
          keyProperty,
          keyColumn,
          // DatabaseID
          null,
          languageDriver,
          // ResultSets
          options != null ? nullOrEmpty(options.resultSets()) : null);
    }
  }
``` 

注意一点，就是无论 xml 还是 注解，解析完成后都会有一个 MapperedStatement 对象（里面包含 SqlSource，代表定义的sql原始信息）  

这里其实就解析完成了，理一理流程    

1. SqlSessionFactoryBuilder 使用 Congiguration 构建
2. 解析核心配置，设置到 Configuration 实例上（自定义配置，全局配置，别名，扩展配置，对象工厂，反射工厂，环境，数据库标识，类型处理，mappers）
3. 解析mappers获得 MappedStatement 对象，里面包含 SqlSource 对象和其他一些配置信息
4. 注册 Mapper 接口，注册使用的是 MapperProxyFactory，还是会解析 MappedStatement 对象（每个方法上的注解），同时解析同名 mapper.xml 文件    

其中 SqlSource 对象是根据 LanguageDriver 创建的，默认的是 XMLLanguageDriver 实现，通常这个也不会去修改  
