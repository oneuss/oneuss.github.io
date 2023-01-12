---
layout: post
title:  "MyBatis 源码简单分析"
subtitle: "常被问到"
date:   2019-04-11
background: '/img/imac_bg.png'
---
`MyBatis`的使用可以参考：
[官方文档中文版](http://www.mybatis.org/mybatis-3/zh/index.html)
记得老早前看时候还有部分未翻译，现在进去看已经全部翻译了。
根据文档介绍写了下面这一部分代码 ↓
```java
public static void main(String[] args) throws Exception {
        //从各种类加载器的路径加载文件
        InputStream res = Resources.getResourceAsStream("resources/mybatis-config.xml");
        SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(res);
        SqlSession sqlSession = sqlSessionFactory.openSession();
        try {
            BlogMapper mapper = sqlSession.getMapper(BlogMapper.class);
            Blog blog = mapper.selectBlog(1);
            System.out.println(blog);
        } finally {
            sqlSession.close();
        }
    }
```
代码很简单，现在debug启动进去看看：
# 第一行代码   加载配置
`InputStream res = Resources.getResourceAsStream("resources/mybatis-config.xml");`  
`getResourceAsStream`方法，从各个类加载器路径上尝试加载配置文件：
```java
  /*
   * Try to get a resource from a group of classloaders
   *
   * @param resource    - the resource to get
   * @param classLoader - the classloaders to examine
   * @return the resource or null
   */
  InputStream getResourceAsStream(String resource, ClassLoader[] classLoader) {
    for (ClassLoader cl : classLoader) {
      if (null != cl) {

        // try to find the resource as passed
        InputStream returnValue = cl.getResourceAsStream(resource);

        // now, some class loaders want this leading "/", so we'll add it and try again if we didn't find the resource
        if (null == returnValue) {
          returnValue = cl.getResourceAsStream("/" + resource);
        }

        if (null != returnValue) {
          return returnValue;
        }
      }
    }
    return null;
  }
```
``ClassLoader[]`` 来源：
```java
  ClassLoader[] getClassLoaders(ClassLoader classLoader) {
    return new ClassLoader[]{
        classLoader,
        defaultClassLoader,
        Thread.currentThread().getContextClassLoader(),
        getClass().getClassLoader(),
        systemClassLoader};
  }
```
# 第二行代码  构建SqlSessionFactory工厂
`SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(res);`  
`build`方法，后两个参数为空，进入该重载方法中：
```java
  public SqlSessionFactory build(InputStream inputStream, String environment, Properties properties) {
    try {
      XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);
      return build(parser.parse());
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error building SqlSession.", e);
    } finally {
      ErrorContext.instance().reset();
      try {
        inputStream.close();
      } catch (IOException e) {
        // Intentionally ignore. Prefer previous error.
      }
    }
  }

  public SqlSessionFactory build(Configuration config) {
    return new DefaultSqlSessionFactory(config);
  }
```
`XMLConfigBuilder` 的 `parse`方法：
```java
  public Configuration parse() {
    if (parsed) {
      throw new BuilderException("Each XMLConfigBuilder can only be used once.");
    }
    parsed = true;
    parseConfiguration(parser.evalNode("/configuration"));
    return configuration;
  }
```
`parseConfiguration`方法。可以看到`Configuration`的值基本上都在该方法设置进去：
```java
  private void parseConfiguration(XNode root) {
    try {
      //issue #117 read properties first
      propertiesElement(root.evalNode("properties"));
      typeAliasesElement(root.evalNode("typeAliases"));
      pluginElement(root.evalNode("plugins"));
      objectFactoryElement(root.evalNode("objectFactory"));
      objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
      reflectionFactoryElement(root.evalNode("reflectionFactory"));
      settingsElement(root.evalNode("settings"));
      // read it after objectFactory and objectWrapperFactory issue #631
      environmentsElement(root.evalNode("environments"));
      databaseIdProviderElement(root.evalNode("databaseIdProvider"));
      typeHandlerElement(root.evalNode("typeHandlers"));
      mapperElement(root.evalNode("mappers"));
    } catch (Exception e) {
      throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
    }
  }
```
再把刚才的`Configuration`作为构造器参数传入`DefaultSqlSessionFactory`构造器中：
```java
  public SqlSessionFactory build(Configuration config) {
    return new DefaultSqlSessionFactory(config);
  }
```
至此，`SqlSessionFactory`构建完成。
# 第三行代码  获取SqlSession
`SqlSession sqlSession = sqlSessionFactory.openSession();`
`DefaultSqlSessionFactory`的`openSession`方法：
```java
  @Override
  public SqlSession openSession() {
    return openSessionFromDataSource(configuration.getDefaultExecutorType(), null, false);
  }
```
`Configuration`的`getDefaultExecutorType`方法：
```java
  //看起来，ExecutorType的三种类型代表意思是：直接执行，重用，批量执行。
  protected ExecutorType defaultExecutorType = ExecutorType.SIMPLE;
  public ExecutorType getDefaultExecutorType() {
    return defaultExecutorType;
  }

```
`DefaultSqlSessionFactory`的`openSessionFromDataSource`方法：
```java
  private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
    Transaction tx = null;
    try {
      //获取configuration中的Environment环境配置
      final Environment environment = configuration.getEnvironment();
      //获取事务工厂，代码如下面部分
      final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
      //参见 Transaction创建
      tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
      final Executor executor = configuration.newExecutor(tx, execType);
      return new DefaultSqlSession(configuration, executor, autoCommit);
    } catch (Exception e) {
      closeTransaction(tx); // may have fetched a connection so lets call close()
      throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
```
`DefaultSqlSessionFactory`的`getTransactionFactoryFromEnvironment`方法：
`TransactionFactory`接口有两个实现类：`ManagedTransactionFactory`和`JdbcTransactionFactory`。
参考：[# [MyBatis源码解析（三）——Transaction事务模块](https://www.cnblogs.com/V1haoge/p/6634151.html)]
```java
  private TransactionFactory getTransactionFactoryFromEnvironment(Environment environment) {
    if (environment == null || environment.getTransactionFactory() == null) {
      return new ManagedTransactionFactory();
    }
    return environment.getTransactionFactory();
  }
```
## TransactionFactory何时创建?
这儿的`environment.getTransactionFactory()`是如何获取到事务工厂的？
记得上面的`parseConfiguration`方法吗，里面有如下方法：
根据[`mybatis-config.xml`](https://www.jianshu.com/p/190c165d948b)配置创建Environment对象

```java
  private void environmentsElement(XNode context) throws Exception {
    if (context != null) {
      if (environment == null) {
        environment = context.getStringAttribute("default");
      }
      for (XNode child : context.getChildren()) {
        String id = child.getStringAttribute("id");
        if (isSpecifiedEnvironment(id)) {
          //这句代码创建事务工厂
          TransactionFactory txFactory = transactionManagerElement(child.evalNode("transactionManager"));
          DataSourceFactory dsFactory = dataSourceElement(child.evalNode("dataSource"));
          DataSource dataSource = dsFactory.getDataSource();
          Environment.Builder environmentBuilder = new Environment.Builder(id)
              .transactionFactory(txFactory)
              .dataSource(dataSource);
          configuration.setEnvironment(environmentBuilder.build());
        }
      }
    }
  }
```
`transactionManagerElement`方法，事务工厂创建完成：
```java
  private TransactionFactory transactionManagerElement(XNode context) throws Exception {
    if (context != null) {
      //获取到 <transactionManager type="JDBC">配置，假如是JDBC
      String type = context.getStringAttribute("type");
      Properties props = context.getChildrenAsProperties();
      //这个方法会根据类型获取到对应的Class对象，再通过反射获取实例
      TransactionFactory factory = (TransactionFactory) resolveClass(type).newInstance();
      factory.setProperties(props);
      return factory;
    }
    throw new BuilderException("Environment declaration requires a TransactionFactory.");
  }
```
## `Transaction`创建
看这行代码
`tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);`
然后
```java
  @Override
  public Transaction newTransaction(DataSource ds, TransactionIsolationLevel level, boolean autoCommit) {
    // Silently ignores autocommit and isolation level, as managed transactions are entirely
    // controlled by an external manager.  It's silently ignored so that
    // code remains portable between managed and unmanaged configurations.
    return new ManagedTransaction(ds, level, closeConnection);
  }
```
## Executor创建
`final Executor executor = configuration.newExecutor(tx, execType);`
上面说过，这里的`execType`是`SIMPLE`
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
    //这句话不理解是什么意思
    executor = (Executor) interceptorChain.pluginAll(executor);
    return executor;
  }
```
## 下一句创建SqlSession了
`return new DefaultSqlSession(configuration, executor, autoCommit);`
# 第四行 创建`Mapper`
这应该是最重要的部分：`BlogMapper mapper = sqlSession.getMapper(BlogMapper.class);`
是从`configuration`里获取的mapper。
`DefaultSqlSession`的`getMapper`方法：
```java
  @Override
  public <T> T getMapper(Class<T> type) {
    return configuration.<T>getMapper(type, this);
  }
```
`Configuration`的`getMapper`方法：
```java
  public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    return mapperRegistry.getMapper(type, sqlSession);
  }
```
`MapperRegistry`的`getMapper`方法：
```java
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
## `XMLConfigBuilder的mapperElement`方法
先回到上面创建`SqlSessionFactory`时提到的`parseConfiguration`方法，里面调用的`mapperElement`方法：
`mapperElement(root.evalNode("mappers"));`
```java
  private void mapperElement(XNode parent) throws Exception {
    if (parent != null) {
      for (XNode child : parent.getChildren()) {
        if ("package".equals(child.getName())) {
          String mapperPackage = child.getStringAttribute("name");
          configuration.addMappers(mapperPackage);
        } else {
          String resource = child.getStringAttribute("resource");//resources\test\BlogMapper.xml
          String url = child.getStringAttribute("url");//null
          String mapperClass = child.getStringAttribute("class");//null
          if (resource != null && url == null && mapperClass == null) {
            ErrorContext.instance().resource(resource);
            InputStream inputStream = Resources.getResourceAsStream(resource);
            XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource, configuration.getSqlFragments());
            mapperParser.parse();
          } else if (resource == null && url != null && mapperClass == null) {
            ErrorContext.instance().resource(url);
            InputStream inputStream = Resources.getUrlAsStream(url);
            XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, url, configuration.getSqlFragments());
            mapperParser.parse();
          } else if (resource == null && url == null && mapperClass != null) {
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
看这儿时候参考下`mybatis-config.xml`里面的配置。我的代码在这里[GitHub:wangzz](https://github.com/wangzzleo/wangzz)。配置里mappers就一句：
``<mapper resource="resources\test\BlogMapper.xml"/>``
所以到`else`分支，再到这个分支：
```java
ErrorContext.instance().resource(resource);
            InputStream inputStream = Resources.getResourceAsStream(resource);
            XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource, configuration.getSqlFragments());
            mapperParser.parse();
```
主要是`parse`方法，
`XMLMapperBuilder`的`parse`方法：
```java
  public void parse() {
    if (!configuration.isResourceLoaded(resource)) {
      configurationElement(parser.evalNode("/mapper"));
      configuration.addLoadedResource(resource);
      bindMapperForNamespace();
    }

    parsePendingResultMaps();
    parsePendingChacheRefs();
    parsePendingStatements();
  }
```
`bindMapperForNamespace`方法如下：
`configuration.addMapper(boundType);`方法将类型放到`mapperRegistry`里，再将`type`和`MapperProxyFactory`放进`map`里。
```java
  private void bindMapperForNamespace() {
    String namespace = builderAssistant.getCurrentNamespace();
    if (namespace != null) {
      Class<?> boundType = null;
      try {
        boundType = Resources.classForName(namespace);
      } catch (ClassNotFoundException e) {
        //ignore, bound type is not required
      }
      if (boundType != null) {
        if (!configuration.hasMapper(boundType)) {
          // Spring may not know the real resource name so we set a flag
          // to prevent loading again this resource from the mapper interface
          // look at MapperAnnotationBuilder#loadXmlResource
          configuration.addLoadedResource("namespace:" + namespace);
          configuration.addMapper(boundType);
        }
      }
    }
  }
```
再回到咱们的`getMapper`方法。`MapperProxyFactory`有了，再看`mapperProxyFactory.newInstance(sqlSession);`这句。
```java
  protected T newInstance(MapperProxy<T> mapperProxy) {
    return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
  }

  public T newInstance(SqlSession sqlSession) {
    final MapperProxy<T> mapperProxy = new MapperProxy<T>(sqlSession, mapperInterface, methodCache);
    return newInstance(mapperProxy);
  }
```
所以这里是通过动态代理实现的我们的`BlogMapper`接口。至此，`mapper`也有了，可以执行了就。
