[TOC]

# mybatis 执行分析  
sql 执行时通过 SqlSession 来操作的，而 SqlSession 是通过 SqlSessionFactory 获取的。 

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

## openSession
我们已经知道，内部使用的是默认的 DefaultSqlSessionFactory 实现  
```java
// openSession 方法，主要调用的是这个（在与 spring 集成的时候，实际业务开发不直接用 SqlSession）
  @Override
  public SqlSession openSession() {
    return openSessionFromDataSource(configuration.getDefaultExecutorType(), null, false);
  }

@Override
  public SqlSession openSession(Connection connection) {
    return openSessionFromConnection(configuration.getDefaultExecutorType(), connection);
  }
``` 

有从数据源获取和直接使用连接获取。  
其中 executorType   
```java
public enum ExecutorType {
  SIMPLE, REUSE, BATCH
}
``` 
对应的是不同的 Executor 实现，默认的是 Simple， REUSE 代表服用 Statement（这里的 Statement 是 JDBC 里面的），只要连接没断开， BATCH 代表批处理  

具体分析 `openSessionFromDataSource` 方法   
```java
// 从 数据源 获取
  private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
    Transaction tx = null;
    try {
      // 环境配置(事务和数据源)
      final Environment environment = configuration.getEnvironment();
      // 获取事务工厂，如果没有配置就用默认的
      final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
      // 新建事务类（管理事务的 Transaction ）
      tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
      // 构造执行器（根据 type 分为 batch，reuse，simple）
      final Executor executor = configuration.newExecutor(tx, execType);
      // 构造默认的 SqlSession 实现
      return new DefaultSqlSession(configuration, executor, autoCommit);
    } catch (Exception e) {
      closeTransaction(tx); // may have fetched a connection so lets call close()
      throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
``` 

这里的 `TransactionIsolationLevel` 代表事务等级 
```java
public enum TransactionIsolationLevel {
  NONE(Connection.TRANSACTION_NONE),
  READ_COMMITTED(Connection.TRANSACTION_READ_COMMITTED),
  READ_UNCOMMITTED(Connection.TRANSACTION_READ_UNCOMMITTED),
  REPEATABLE_READ(Connection.TRANSACTION_REPEATABLE_READ),
  SERIALIZABLE(Connection.TRANSACTION_SERIALIZABLE);

  private final int level;

  private TransactionIsolationLevel(int level) {
    this.level = level;
  }

  public int getLevel() {
    return level;
  }
}
``` 

这里补充一下事务等级的含义  
>READ-UNCOMMITED(读取未提交内容)

看名字就知道了，所以事务都可以看到其他未提交事务的执行结果，此隔离级别很少使用。

READ-COMMITED（读取提交内容）

这个大多数数据库系统默认的隔离级别，但不是MySQL默认的隔离级别。 
定义：一个事务从开始到提交前所做的任何修改都是不可见的，事务只能看到已经提交事务所做的改变。 
会有不可重复读的问题。 
也就是一个事务同一条查询语句查询了两次，在这两次查询期间，有其他事务修改了这些数据，所以，这两次查询结果在同一个事务中是不一样的。

REPEATABLE-READ(可重复读)

这是MySQL的默认事务隔离级别。能确保同一事务的多个实例在缤纷发读取数据时，会看到同样的数据行。 
但是会有幻读的问题。 
在一次事务的两次查询中数据笔数不一致。 
第一个事务对表的全部数据进行修改，同事第二个事务插入了一行新的数据，那么就会发生第一个事务结束后发现表中还有没有修改的数据行。

Seraializable(可串行化)

这是最高隔离界别，通过强制事务排序，使之不可能互相冲突，从而解决幻读，简言之，就是在每个读取的数据行上加上共享锁实现，这个级别可能会导致大量超时现象和锁竞争。  

再来看代码
1. 先从环境 Environment 里面获取配置的事务工厂 TransactionFactory，默认是 ManagedTransactionFactory   
也就是交给外部容器管理，这种模式下，事务是不会提交的。  
```java
// 获取事物管理实现（之所以有这个，是为了方便与其他容器集成）
  private TransactionFactory getTransactionFactoryFromEnvironment(Environment environment) {
    if (environment == null || environment.getTransactionFactory() == null) {
      return new ManagedTransactionFactory();
    }
    return environment.getTransactionFactory();
  }
``` 

2. 新建事务管理类 Transaction   
这里以 ManagedTransactionFactory 为例
```java
@Override
  public Transaction newTransaction(DataSource ds, TransactionIsolationLevel level, boolean autoCommit) {
    // Silently ignores autocommit and isolation level, as managed transactions are entirely
    // controlled by an external manager.  It's silently ignored so that
    // code remains portable between managed and unmanaged configurations.
    return new ManagedTransaction(ds, level, closeConnection);
  }

public ManagedTransaction(DataSource ds, TransactionIsolationLevel level, boolean closeConnection) {
    this.dataSource = ds;
    this.level = level;
    this.closeConnection = closeConnection;
  }
``` 

3. 新建执行器 Executor, 不特殊配置默认是 SimpleExecutor，如果使用了二级缓存，会用 CachingExecutor 装饰，另外，扩展插件也会在这里添加    
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
``` 
我们后面详细分析 `executor = (Executor) interceptorChain.pluginAll(executor);` 这一行   

4. 构建 SqlSession 实例 DefaultSqlSession   
```java
private final Configuration configuration;
  private final Executor executor;

  private final boolean autoCommit;
  private boolean dirty;
  private List<Cursor<?>> cursorList;

  public DefaultSqlSession(Configuration configuration, Executor executor, boolean autoCommit) {
    this.configuration = configuration;
    this.executor = executor;
    this.dirty = false;
    this.autoCommit = autoCommit;
  }
``` 

获取到 SqlSession 实例后，就可以调用 selectList （selectOne 是 selectList 的特例），游标查询通常也用的少，增删改都是 update 操作。  
```java
@Override
  public <T> T selectOne(String statement, Object parameter) {
    // Popular vote was to return null on 0 results and throw exception on too many.
    List<T> list = this.<T>selectList(statement, parameter);
    if (list.size() == 1) {
      return list.get(0);
    } else if (list.size() > 1) {
      throw new TooManyResultsException("Expected one result (or null) to be returned by selectOne(), but found: " + list.size());
    } else {
      return null;
    }
  }

@Override
  public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
    try {
      // 获取对应的 MappedStatement ，就是根据key（namespace） 对应的 sql 已经其他相关信息
      // 这里的 MappedStatement 是有 xml 定义的
      MappedStatement ms = configuration.getMappedStatement(statement);
      // 然后执行，用 map 包装集合类（collection，list，array）
      return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }

  /**
   * insert update delete 最终执行的地方
   * @param statement Unique identifier matching the statement to execute.
   * @param parameter A parameter object to pass to the statement.
   * @return
   */
  @Override
  public int update(String statement, Object parameter) {
    try {
      dirty = true;
      MappedStatement ms = configuration.getMappedStatement(statement);
      return executor.update(ms, wrapCollection(parameter));
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error updating database.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }

  /**
   * 这里是查询语句+自定义处理返回的 查询实现
   * @param statement Unique identifier matching the statement to use.
   * @param parameter
   * @param rowBounds RowBound instance to limit the query results
   * @param handler ResultHandler that will handle each retrieved row
   */
  @Override
  public void select(String statement, Object parameter, RowBounds rowBounds, ResultHandler handler) {
    try {
      MappedStatement ms = configuration.getMappedStatement(statement);
      executor.query(ms, wrapCollection(parameter), rowBounds, handler);
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
``` 

其中重点是 selectList 和 update 参数都是 statement（namespace.id） 加上给的sql参数  
可以看到，实际的执行时 executor。这个 executor 是构造 SqlSession 的时候创建的，默认 SimpleExecutor，可能会被 CacheExecutor 装饰，作用就是查询添加缓存，
增删改删除缓存，这是默认行为，可以修改。    
我们先不看缓存  

```java
@Override
  public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
    // 获取sql
    BoundSql boundSql = ms.getBoundSql(parameter);
    // cache key
    CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);
    // 查询
    return query(ms, parameter, rowBounds, resultHandler, key, boundSql);
 }

@SuppressWarnings("unchecked")
  @Override
  public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    // 记录执行信息
    ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
    if (closed) {
      throw new ExecutorException("Executor was closed.");
    }
    if (queryStack == 0 && ms.isFlushCacheRequired()) {
      clearLocalCache();
    }
    List<E> list;
    try {
      queryStack++;
      // 如果有自定义 resultHandler ，那就走不了缓存了（一级缓存）
      list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
      if (list != null) {
        // 这里是处理 CALLABLE 存储过程的，一般用不着
        handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
      } else {
        // 缓存没取到，就从数据库里面获取
        list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
      }
    } finally {
      queryStack--;
    }
    if (queryStack == 0) {
      // 需要延时加载的，全部加载
      for (DeferredLoad deferredLoad : deferredLoads) {
        deferredLoad.load();
      }
      // issue #601
      deferredLoads.clear();
      // 如果缓存是依次 statement 的，那就清空缓存
      if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
        // issue #482
        clearLocalCache();
      }
    }
    return list;
  }

  @Override
  public int update(MappedStatement ms, Object parameter) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity("executing an update").object(ms.getId());
    if (closed) {
      throw new ExecutorException("Executor was closed.");
    }
    clearLocalCache();
    return doUpdate(ms, parameter);
  }
``` 

这里面也有缓存，这里是一级缓存，重点方法是 `queryFromDatabase`  
```java
private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    List<E> list;
    // 先在缓存里面丢一个占位符，防止缓存穿透
    localCache.putObject(key, EXECUTION_PLACEHOLDER);
    try {
      // 执行查询,子类实现
      list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
    } finally {
      // 删除占位符
      localCache.removeObject(key);
    }
    // 添加实际缓存(为啥不直接put吧之前的顶掉。。)
    localCache.putObject(key, list);
    // 如果是存储过程，缓存参数
    if (ms.getStatementType() == StatementType.CALLABLE) {
      localOutputParameterCache.putObject(key, parameter);
    }
    return list;
  }
``` 

其中 `doQuery` `doUpdate` 方法是对应子类实现的，由于默认是 SimpleExecutor，我们看这个实现  
```java
  @Override
  public int doUpdate(MappedStatement ms, Object parameter) throws SQLException {
    Statement stmt = null;
    try {
      Configuration configuration = ms.getConfiguration();
      // 大部分内部对象的构造，都是通过 Configuration，构造的时候，会把插件链添加进去。这里构造 StatementHandler
      StatementHandler handler = configuration.newStatementHandler(this, ms, parameter, RowBounds.DEFAULT, null, null);
      // 预处理
      stmt = prepareStatement(handler, ms.getStatementLog());
      // 执行，可以看到，操作封装在 StatementHandler 里面了
      return handler.update(stmt);
    } finally {
      closeStatement(stmt);
    }
  }

  @Override
  public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
    Statement stmt = null;
    try {
      Configuration configuration = ms.getConfiguration();
      StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
      stmt = prepareStatement(handler, ms.getStatementLog());
      // 跟 doUpdate 唯一不同的就是这里了
      return handler.<E>query(stmt, resultHandler);
    } finally {
      closeStatement(stmt);
    }
  }

  private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
    Statement stmt;
    // 调用父类的getConnection，如果允许debug日志打印，拿到的是代理
    Connection connection = getConnection(statementLog);
    // prepare 预处理
    stmt = handler.prepare(connection, transaction.getTimeout());
    // 参数设置
    handler.parameterize(stmt);
    // 返回
    return stmt;
  }
``` 

从这里可以看到，又调用了 StatementHandler 来处理所有sql，这个又是 configuration 创建的，我们看一下  
```java
  public StatementHandler newStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
    StatementHandler statementHandler = new RoutingStatementHandler(executor, mappedStatement, parameterObject, rowBounds, resultHandler, boundSql);
    // 这里就是扩展了，使用插件扩展，主要扩展接口是 Interceptor ，被代理的是 StatementHandler 的实现类
    statementHandler = (StatementHandler) interceptorChain.pluginAll(statementHandler);
    return statementHandler;
  }

  private final StatementHandler delegate;

  public RoutingStatementHandler(Executor executor, MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {

    switch (ms.getStatementType()) {
      case STATEMENT:
        delegate = new SimpleStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
        break;
      case PREPARED:
        delegate = new PreparedStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
        break;
      case CALLABLE:
        delegate = new CallableStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
        break;
      default:
        throw new ExecutorException("Unknown statement type: " + ms.getStatementType());
    }

  }
``` 

这里的 RoutingStatementHandler 只是简单的代理，根据类型来选择不同的实现，默认的 PREPARED，也就是我们直接使用 JDBC 的时候的 PreparedStatment 
我们之前看到的 handler 的使用流程是 
1. prepare
2. parameterize
3. query 或者 update    

我们看看默认的选择实现  
```java
  // 获取 Statement ，以及设置一些参数
  @Override
  public Statement prepare(Connection connection, Integer transactionTimeout) throws SQLException {
    ErrorContext.instance().sql(boundSql.getSql());
    Statement statement = null;
    try {
      // 生成 Statement 子类实现
      statement = instantiateStatement(connection);
      // 设置查询超时时间
      setStatementTimeout(statement, transactionTimeout);
      // 设置每次去数据库获取数据的大小（当一次获取过多的数据时候，可能会内存溢出，设置这个参数，在获取结果集的时候循环去取）
      setFetchSize(statement);
      return statement;
    } catch (SQLException e) {
      closeStatement(statement);
      throw e;
    } catch (Exception e) {
      closeStatement(statement);
      throw new ExecutorException("Error preparing statement.  Cause: " + e, e);
    }
  }
``` 

prepare 是基类的方法，留一个 instantiateStatement 子类实现，也就是创建不同的 Statement，基类统一设置参数    
```java
  @Override
  protected Statement instantiateStatement(Connection connection) throws SQLException {
    String sql = boundSql.getSql();
    // 这里生成 prepareStatement，对自动生成（自增字段等）的值进行设置
    if (mappedStatement.getKeyGenerator() instanceof Jdbc3KeyGenerator) {
      String[] keyColumnNames = mappedStatement.getKeyColumns();
      if (keyColumnNames == null) {
        // 如果没有指明生成的字段名，那么就要设置可检索
        return connection.prepareStatement(sql, PreparedStatement.RETURN_GENERATED_KEYS);
      } else {
        // 如果设置了字段名，那么直接让JDBC 带出来
        return connection.prepareStatement(sql, keyColumnNames);
      }
    } else if (mappedStatement.getResultSetType() != null) {
      // 设置结果集的滑动方向（单向，双向），设置结果集只读
      return connection.prepareStatement(sql, mappedStatement.getResultSetType().getValue(), ResultSet.CONCUR_READ_ONLY);
    } else {
      // 啥都没配置，那就是默认
      return connection.prepareStatement(sql);
    }
  }
``` 

这是默认的 PreparedStatementHandler 实现。在这里，我们终于看到了 JDBC 的东西了  
然后是 parameterize 
```java
  @Override
  public void parameterize(Statement statement) throws SQLException {
    // 设置参数类 DefaultParameterHandler
    parameterHandler.setParameters((PreparedStatement) statement);
  }
``` 

这里的 parameterHandler 是构造的时候(基类)创建的  
```java
  protected BaseStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
    this.configuration = mappedStatement.getConfiguration();
    this.executor = executor;
    this.mappedStatement = mappedStatement;
    this.rowBounds = rowBounds;

    this.typeHandlerRegistry = configuration.getTypeHandlerRegistry();
    this.objectFactory = configuration.getObjectFactory();

    if (boundSql == null) { // issue #435, get the key before calculating the statement
      // 生成字段，执行sql之前处理
      generateKeys(parameterObject);
      //  statementHandler 每次都是 new 的，这里每次重新获取 sql，里面涉及到解析过程（动态sql可能跟参数相关）
      boundSql = mappedStatement.getBoundSql(parameterObject);
    }

    // sql
    this.boundSql = boundSql;

    // 参数处理类
    this.parameterHandler = configuration.newParameterHandler(mappedStatement, parameterObject, boundSql);
    // 结果处理类
    this.resultSetHandler = configuration.newResultSetHandler(executor, mappedStatement, rowBounds, parameterHandler, resultHandler, boundSql);
  }

public ParameterHandler newParameterHandler(MappedStatement mappedStatement, Object parameterObject, BoundSql boundSql) {
    ParameterHandler parameterHandler = mappedStatement.getLang().createParameterHandler(mappedStatement, parameterObject, boundSql);
    parameterHandler = (ParameterHandler) interceptorChain.pluginAll(parameterHandler);
    return parameterHandler;
  }

  @Override
  public ParameterHandler createParameterHandler(MappedStatement mappedStatement, Object parameterObject, BoundSql boundSql) {
    return new DefaultParameterHandler(mappedStatement, parameterObject, boundSql);
  }

public ResultSetHandler newResultSetHandler(Executor executor, MappedStatement mappedStatement, RowBounds rowBounds, ParameterHandler parameterHandler,
      ResultHandler resultHandler, BoundSql boundSql) {
    ResultSetHandler resultSetHandler = new DefaultResultSetHandler(executor, mappedStatement, parameterHandler, resultHandler, boundSql, rowBounds);
    resultSetHandler = (ResultSetHandler) interceptorChain.pluginAll(resultSetHandler);
    return resultSetHandler;
  }
``` 

ParameterHandler 的作用是设置参数。使用传的参数和系统默认的参数，加上类型转换   
```java
@Override
  public void setParameters(PreparedStatement ps) {
    // 参数设置
    ErrorContext.instance().activity("setting parameters").object(mappedStatement.getParameterMap().getId());
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    if (parameterMappings != null) {
      for (int i = 0; i < parameterMappings.size(); i++) {
        ParameterMapping parameterMapping = parameterMappings.get(i);
        if (parameterMapping.getMode() != ParameterMode.OUT) {
          Object value;
          String propertyName = parameterMapping.getProperty();
          if (boundSql.hasAdditionalParameter(propertyName)) { // issue #448 ask first for additional params
            value = boundSql.getAdditionalParameter(propertyName);
          } else if (parameterObject == null) {
            value = null;
          } else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
            value = parameterObject;
          } else {
            MetaObject metaObject = configuration.newMetaObject(parameterObject);
            // 重点在这里，从参数对象里面获取值
            value = metaObject.getValue(propertyName);
          }
          TypeHandler typeHandler = parameterMapping.getTypeHandler();
          JdbcType jdbcType = parameterMapping.getJdbcType();
          if (value == null && jdbcType == null) {
            jdbcType = configuration.getJdbcTypeForNull();
          }
          try {
            // 这里类型转换
            typeHandler.setParameter(ps, i + 1, value, jdbcType);
          } catch (TypeException e) {
            throw new TypeException("Could not set parameters for mapping: " + parameterMapping + ". Cause: " + e, e);
          } catch (SQLException e) {
            throw new TypeException("Could not set parameters for mapping: " + parameterMapping + ". Cause: " + e, e);
          }
        }
      }
    }
  }
``` 

最后，就是 query 和 update 了   
```java
  @Override
  public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
    PreparedStatement ps = (PreparedStatement) statement;
    // 无论是 更新 还行查询，都执行 execute，然后处理结果集
    ps.execute();
    // 结果集处理类 DefaultResultSetHandler
    return resultSetHandler.<E> handleResultSets(ps);
  }

  @Override
  public int update(Statement statement) throws SQLException {
    PreparedStatement ps = (PreparedStatement) statement;
    ps.execute();
    int rows = ps.getUpdateCount();
    // 更新操作可能需要获取数据库自动生成的字段值（insert 自增）
    Object parameterObject = boundSql.getParameterObject();
    KeyGenerator keyGenerator = mappedStatement.getKeyGenerator();
    keyGenerator.processAfter(executor, mappedStatement, ps, parameterObject);
    return rows;
  }
``` 

查询后字段映射到对象，resultSetHandler 根据对象类型或者 ResultMap，老复杂了。    
```java
// 解析过程老复杂了
  //
  // HANDLE RESULT SETS
  //
  @Override
  public List<Object> handleResultSets(Statement stmt) throws SQLException {
    ErrorContext.instance().activity("handling results").object(mappedStatement.getId());

    final List<Object> multipleResults = new ArrayList<>();

    int resultSetCount = 0;
    // 包装一下
    ResultSetWrapper rsw = getFirstResultSet(stmt);
    // 获取 resultMap 可能有内联的
    List<ResultMap> resultMaps = mappedStatement.getResultMaps();
    int resultMapCount = resultMaps.size();
    // 验证，数量
    validateResultMapsCount(rsw, resultMapCount);
    while (rsw != null && resultMapCount > resultSetCount) {
      ResultMap resultMap = resultMaps.get(resultSetCount);
      handleResultSet(rsw, resultMap, multipleResults, null);
      rsw = getNextResultSet(stmt);
      cleanUpAfterHandlingResultSet();
      resultSetCount++;
    }

    String[] resultSets = mappedStatement.getResultSets();
    if (resultSets != null) {
      while (rsw != null && resultSetCount < resultSets.length) {
        ResultMapping parentMapping = nextResultMaps.get(resultSets[resultSetCount]);
        if (parentMapping != null) {
          String nestedResultMapId = parentMapping.getNestedResultMapId();
          ResultMap resultMap = configuration.getResultMap(nestedResultMapId);
          handleResultSet(rsw, resultMap, null, parentMapping);
        }
        rsw = getNextResultSet(stmt);
        cleanUpAfterHandlingResultSet();
        resultSetCount++;
      }
    }

    return collapseSingleResultList(multipleResults);
  }
``` 

上面就是 执行的大体流程了   
