

## Mybatis 的工作流程

### 解析配置文件

> Mybatis 在启动时需要去解析配置文件，包括全局配置文件和映射器配置文件。这里面包含了我们怎么控制Mybatis的行为和我们要对数据库下达的指令（即我们的SQL信息），这些信息会被解析成一个`Configuration`对象

SqlSessionFactoryBuilder.java

```java
...
 public SqlSessionFactory build(InputStream inputStream, String environment, Properties properties) {
    try {
      // 用于解析 mybatis-config.xml，同时创建了 Configuration 对象 >>
      XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);
      // 解析XML，最终返回一个 DefaultSqlSessionFactory >>
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
...
```

XMLConfigBuilder.java

```java
...
public Configuration parse() {
    if (parsed) {
      throw new BuilderException("Each XMLConfigBuilder can only be used once.");
    }
    parsed = true;
    // XPathParser，dom 和 SAX 都有用到 >>
    parseConfiguration(parser.evalNode("/configuration"));
    return configuration;
  }

  private void parseConfiguration(XNode root) {
    try {
      //issue #117 read properties first
      // 对于全局配置文件各种标签的解析
      propertiesElement(root.evalNode("properties"));
      // 解析 settings 标签
      Properties settings = settingsAsProperties(root.evalNode("settings"));
      loadCustomVfs(settings);
      loadCustomLogImpl(settings);
      // 类型别名
      typeAliasesElement(root.evalNode("typeAliases"));
      // 插件
      pluginElement(root.evalNode("plugins"));
      // 用于创建对象
      objectFactoryElement(root.evalNode("objectFactory"));
      // 用于对对象进行加工
      objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
      // 反射工具箱
      reflectorFactoryElement(root.evalNode("reflectorFactory"));
      // settings 子标签赋值，默认值就是在这里提供的 >>
      settingsElement(settings);
      // read it after objectFactory and objectWrapperFactory issue #631
      // 创建了数据源 >>
      environmentsElement(root.evalNode("environments"));
      databaseIdProviderElement(root.evalNode("databaseIdProvider"));
      // 类型映射处理器的加载
      typeHandlerElement(root.evalNode("typeHandlers"));
      // 解析引用的Mapper映射器
      mapperElement(root.evalNode("mappers"));
    } catch (Exception e) {
      throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
    }
...
```

### 操作接口

接下来就是我们操作数据库的接口，它在应用程序和数据库中间，代表我们跟数据库之间的一次连接：这个就是SqlSession对象。

我们要获得一个会话就必须有一个会话工厂SqlSessionFactory。SqlSessionFactory里面又必须包含我们的所有的配置信息，所以我们会通过一个Builder类来创建工厂类。

```java
String resource = "mybatis-config.xml";
// 读取配置
InputStream inputStream = Resources.getResourceAsStream(resource);
// 创建SqlSessionFactory,在SqlSessionFactory中包含了配置信息
sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
// 获取SqlSession
SqlSession session = sqlSessionFactory.openSession();
```

Mybatis是怼JDBC的封装，也就意味着底层一定会出现JDBC的一些核心对象，比如执行SQL的Statement，结果集ResultSet。在Mybatis里，SqlSession只是提供给应用的一个接口，还不是SQL真正的执行对象。

### 执行SQL操作

SqlSession持有一个Executor对象，用来封装对数据库的操作。

```java
// DefaultSqlSession.java

public class DefaultSqlSession implements SqlSession {

  // 配置信息
  private final Configuration configuration;
  // 具体操作数据库的对象
  private final Executor executor;
  // 是否自动提交
  private final boolean autoCommit;
  private boolean dirty;
  private List<Cursor<?>> cursorList;

  public DefaultSqlSession(Configuration configuration, Executor executor, boolean autoCommit) {
    this.configuration = configuration;
    this.executor = executor;
    this.dirty = false;
    this.autoCommit = autoCommit;
  }
...
```

在执行器Executor执行query或者update操作的时候，我们创建一系列的对象来处理参数、执行SQL、处理结果集，这里我们把它简化成一个对象StatementHandler，可以把它理解为对Statement的封装，还有其他对象我们在阅读源码的时候再去做进一步的了解。

具体工作流程如下：

![Mybatis工作流程](..\..\images\mybatis\mybatis-flow.png)

## Mybatis的架构分层与模块划分

在Mybatis的主要工作流程中，不通的功能是由很多不同的类协作完成的，他们分布在Mybatis jar包的不通的package中。

Mybatis的jar包结构如下（21个包，基于3.5.4）

```
|___org
	|___apache
		|___ ibatis
			|— annotations
			|— binding
			|— builder
			|— cache
			|— cursor
			|— datasource
			|— exceptions
			|— executor
			|— io
			|— jdbc
			|— lang
			|— logging
			|— mapping
			|— parsing
			|— plugin
			|— reflection
			|— scripting
			|— session
			|— transaction
			|— type
```

按照功能职责的不同，所有的package可以分成不同的工作层次，详情如下图：

![mybatis-包分类图](..\..\images\mybatis\mybatis-包分类图.png)

### 接口层

首先接口层是我们操作最多的，核心对象是SqlSession，它是上层应用和Mybatis交互的桥梁，SqlSession上定义了非常多的对数据库的操作方法。接口层在接收到调用请求时会调用核心处理层的相应模块来完成具体的数据库操作。

### 核心处理层

所有与数据库操作相关的动作都在这一层完成：

1. 把接口中传入的参数解析并映射成JDBC类型；
2. 解析xml文件中的SQL语句，包括插入参数和动态SQL的生成；
3. 执行SQL语句；
4. 处理结果集，并映射成Java对象；

插件也属于核心层，这是由它的工作方式和拦截的对象决定的。

### 基础支持层

基础支持层主要是由一些抽取出来的通用的功能，用来支撑核心处理层的功能。比如数据源、缓存、日志、xml解析、反射、IO、事务等等。



