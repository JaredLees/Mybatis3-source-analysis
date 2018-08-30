# Mybatis3 源码分析
> 为什么要看源码，以下是我个人感悟：

## 源码分析的目的
- 学习代码的风格
- 学习代码的取名
- 学习框架思想
- 学习设计模式
- 学习各种基础Jar包的使用，扩展如：cglib，derby等
- 明白框架的运行流程，方便开发，防止踩坑
- 了解框架的扩展点（*）

## 入口代码如下：
```java
public class MybatisHelloWorld {
    public static void main(String[] args) {
        String resource = "config.xml";
        Reader reader;
        try {
            reader = Resources.getResourceAsReader(resource);
            SqlSessionFactory sqlMapper = new SqlSessionFactoryBuilder().build(reader); //核心代码，跟进

            SqlSession session = sqlMapper.openSession();//核心代码，跟进
            try {
                User user = (User) session.selectOne("org.test.UserMapper.getUser", 1);
                System.out.println(user.getLfPartyId() + "," + user.getPartyName());
            } finally {
                session.close();
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}

public class User {
    private long lfPartyId;
    private String partyName;

    public User() {
    }

    public User(long lfPartyId, String partyName) {
        this.lfPartyId = lfPartyId;
        this.partyName = partyName;
    }

    public long getLfPartyId() {
        return lfPartyId;
    }

    public void setLfPartyId(long lfPartyId) {
        this.lfPartyId = lfPartyId;
    }

    public String getPartyName() {
        return partyName;
    }

    public void setPartyName(String partyName) {
        this.partyName = partyName;
    }
}

public interface UserMapper {
    User getUser(int lfPartyId);
}

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="org.test.UserMapper">
    <select id="getUser" parameterType="int" resultType="org.test.User">
        select lfPartyId,partyName from user where lfPartyId = #{id}
    </select>
</mapper>

//Mybatis的所有配置文件，包括数据库连接，插件或者sql都要通过configuration标签进行统一管理
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE configuration PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC"/>
            <dataSource type="POOLED">
                <property name="driver" value="com.mysql.jdbc.Driver"/>
                <property name="url" value="jdbc:mysql://127.0.0.1:3306/lfBase?useUnicode=true"/>
                <property name="username" value="root"/>
                <property name="password" value="lijun520"/>
            </dataSource>
        </environment>
    </environments>
    <mappers>
        <mapper resource="mapper/UserMapper.xml"/>
    </mappers>
</configuration>
```
核心代码两行：
```java
SqlSessionFactory sqlMapper = new SqlSessionFactoryBuilder().build(reader);
SqlSession session = sqlMapper.openSession();
```
## 执行流程
- SqlSessionFactory的创建与初始化
```java
  //创建一个SqlSessionFactoryBuilder建造者，调用XMLConfigBuilder加载所有配置
  //可选的参数是environment和properties。Environment决定加载哪种环境(开发环境/生产环境)，包括数据源和事务管理器。
  XMLConfigBuilder parser = new XMLConfigBuilder(reader, environment, properties);
  return build(parser.parse());
  
  //调用XPath解析器，用的都是JDK的类包,封装了一下，使得使用起来更方便
  public XMLConfigBuilder(Reader reader, String environment, Properties props) {
    this(new XPathParser(reader, true, props, new XMLMapperEntityResolver()), environment, props);
  }
  
  //XPath解析器解析出所有的配置文件后，立即通过parser.parse()构建Configuration对象，这个对象会一直持有所有配置
  public Configuration parse() {
    if (parsed) {
      throw new BuilderException("Each XMLConfigBuilder can only be used once.");
    }
    parsed = true;
    //所有的配置文件的根都是configuration标签管理，加载解析
    parseConfiguration(parser.evalNode("/configuration"));
    return configuration;
  }

  //构建 Configuration 对象时，看以看到做了以下若干事情
  private void parseConfiguration(XNode root) {
    try {
      //issue #117 read properties first
      //加载config.xml中的所有properties，并将值设置到parser
      propertiesElement(root.evalNode("properties"));
     
      Properties settings = settingsAsProperties(root.evalNode("settings"));
      loadCustomVfs(settings);
      
      //加载类型别名
      typeAliasesElement(root.evalNode("typeAliases"));
      
      //加载插件
      pluginElement(root.evalNode("plugins"));
      
      //加载对象工厂
      objectFactoryElement(root.evalNode("objectFactory"));
      
      //加载包装对象工厂
      objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
      
      //
      reflectorFactoryElement(root.evalNode("reflectorFactory"));
      
      //加载所有的setting
      settingsElement(settings);
      
      // read it after objectFactory and objectWrapperFactory issue #631
      //环境
      environmentsElement(root.evalNode("environments"));
      
      //databaseIdProvider
      databaseIdProviderElement(root.evalNode("databaseIdProvider"));
      
      //类型处理器
      typeHandlerElement(root.evalNode("typeHandlers"));
      
      //映射器
      mapperElement(root.evalNode("mappers"));
    } catch (Exception e) {
      throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
    }
  }
  
  //将Configuration传递给DefaultSqlSessionFactory，构建SqlSessionFactory
  public SqlSessionFactory build(Configuration config) {
    return new DefaultSqlSessionFactory(config);
  }
```
创建SqlSessionFactory简单流程描述如下：
1. 调用SqlSessionFactoryBuilder读取配置文件
2. SqlSessionFactoryBuilder委托XMLConfigBuilder
3. XMLConfigBuilder委托XPathParser（封装的JDK自带的XPath解析XML）去解析加载配置文件
4. 所有的配置信息被加载到Configuration中（几乎后续的所有操作都会与此配置对象打交道）
5. SqlSessionFactoryBuilder调用build方法，创建并将之前创建configuration交给DefaultSqlSessionFactory实例

- 获取SqlSession
> SqlSessionFactory创建完成后，接下来是对sql执行器的构建，每次执行 openSession() 都会默认创建一个DefaultSqlSession对象(封装了执行器，configuration）
```java
  public SqlSession openSession() {
    //默认的ExecutorType = SIMPLE
    return openSessionFromDataSource(configuration.getDefaultExecutorType(), null, false);
  }

 private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
    Transaction tx = null;
    try {
      //从configuration中拉取环境（开发环境|生产环境）
      final Environment environment = configuration.getEnvironment();
      
      //环境中封装了事务工厂和datasource信息 
      final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
      
      //获取事务工厂，创建当前事务
      tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
      
      //获取事务和执行器类型，创建执行器（未指定执行器类型，默认为SIMPLE）
      final Executor executor = configuration.newExecutor(tx, execType);
      
      //将执行器，configuration封装到DefaultSqlSession
      return new DefaultSqlSession(configuration, executor, autoCommit);
    } catch (Exception e) {
      closeTransaction(tx); // may have fetched a connection so lets call close()
      throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
  
   //产生执行器
  public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
    executorType = executorType == null ? defaultExecutorType : executorType;
    //这句再做一下保护,囧,防止粗心大意的人将defaultExecutorType设成null?
    executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
    Executor executor;
    //然后就是简单的3个分支，产生3种执行器BatchExecutor/ReuseExecutor/SimpleExecutor
    if (ExecutorType.BATCH == executorType) {
      executor = new BatchExecutor(this, transaction);
    } else if (ExecutorType.REUSE == executorType) {
      executor = new ReuseExecutor(this, transaction);
    } else {
      executor = new SimpleExecutor(this, transaction);
    }
    //如果要求缓存，生成另一种CachingExecutor(默认就是有缓存),装饰者模式,所以默认都是返回CachingExecutor
    if (cacheEnabled) {
      executor = new CachingExecutor(executor);
    }
    //此处调用插件,通过插件可以改变Executor行为
    executor = (Executor) interceptorChain.pluginAll(executor);
    return executor;
  }
```
获取SqlSession的简单流程描述如下：
1. SqlSession的创建会调用DefaultSqlSessionFactory的openSessionFromDataSource方法
2. 根据configuration中指定的环境信息获取事务工厂创建事务，获取datasource
3. 根据事务和datasource创建执行器（默认SIMPLE），此处会根据是否开启缓存进行CachingExecutor包装，将所有拦截器链中的对象注册executor
4. 包装configuration和executor到DefaultSqlSession返回
