---
layout: w
title: 《深入理解Mybatis原理》Mybatis初始化过程详解
date: 2018-05-23 15:16:27
tags:
    - Mybatis
---

> 最近阅读了《Java Persistence with MyBatis 3》一书，对Mybatis的各类配置及使用有了更深入的的了解。 进而产生了对Mybatis源码进行阅读解析的想法，就从这篇文章开始吧。先对Mybatis框架的初始化进行解析

<!-- more -->
## Mybatis 初始化做了什么
&ensp;&ensp;框架的初始化及加载运行时所需的配置信息，而`Mybatis`的配置则在`mybatis-config.xml`文件中。查看`dtd`文件即可知其主要标签结构如下：

- configuration 配置
 - properties 属性
 - settings 设置
 - typeAliases 对象别名命名
 - typeHandlers 类型处理器
 - objectFactory 对象工厂
 - plugins 插件
 - environments 环境
      - environment 环境变量
         - transactionManager 事务管理器
         - dataSource 数据源
 - mappers

&ensp;&ensp; 众所周知，要使上述`Mybatis`配置加载入Mybatis内部首先要创建`SqlSession`对象：

```java
public class MySqlSessionFactory {

    private static SqlSessionFactory sqlSessionFactory;

    public static SqlSessionFactory getSqlSessionFactory(){
        if (sqlSessionFactory == null){
            InputStream inputStream;
            try {
                inputStream = Resources.getResourceAsStream("mybatis-config.xml");
                sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
            } catch (IOException e){
                e.printStackTrace();
                throw new RuntimeException();
            }
        }

        return sqlSessionFactory;
    }


    public static SqlSession openSession() {
        return getSqlSessionFactory().openSession();
    }

}
```

由上方代码可见此次`Mybatis`初始化源码阅读之旅的入口即为：`SqlSessionFactoryBuilder().build()`方法。进一步阅读，即可发现`Mybatis`中使用`org.apache.ibatis.session.Configuration`类作为配置信息对象的存储容器
```java
public class Configuration {
    protected Environment environment;
    protected Properties variables = new Properties();
    protected final MapperRegistry mapperRegistry = new MapperRegistry(this);
    protected final InterceptorChain interceptorChain = new InterceptorChain();
    protected final TypeHandlerRegistry typeHandlerRegistry = new TypeHandlerRegistry();
    protected final TypeAliasRegistry typeAliasRegistry = new TypeAliasRegistry();
    protected final LanguageDriverRegistry languageRegistry = new LanguageDriverRegistry();
    
    ...
}
```
由此可见，`Mybatis`初始化的过程基本可以看成是`Configuration`对象创建的过程，接下来就将深入探讨`Mybatis`是如何通过`XML`配置的方式构建`COnfiguration`对象。

## MyBatis基于XML配置文件创建Configuration对象的过程

### 时序图：
![image_1ceo09j2j2b7jso1hf5lp1tvf9.png-48.1kB][1]

由上图所示，mybatis初始化要经过简单的以下几步：

 1. 调用`SqlSessionFactoryBuilder`对象的`build(inputStream)`方法；
 2. `SqlSessionFactoryBuilder`会根据输入流`inputStream`等信息创建`XMLConfigBuilder`对象;
 3. `SqlSessionFactoryBuilder`调用`XMLConfigBuilder`对象的`parse()`方法；
 4. `XMLConfigBuilder`对象返回``Configuration``对象；
 5. `SqlSessionFactoryBuilder`根据`Configuration`对象创建一个`DefaultSessionFactory`对象；
 6. `SqlSessionFactoryBuilder`返回`DefaultSessionFactory`对象。

### 创建Configuration对象的过程
`XMLConfigBuilder`调用`parse()`方法：会从`XPathParser`中取出 <configuration>节点对应的`Node`对象，然后解析此`Node`节点的子`Node`：properties, settings, typeAliases,typeHandlers, objectFactory, objectWrapperFactory, plugins, environments,databaseIdProvider, mappers
```
public Configuration parse() {
    if (parsed) {
      throw new BuilderException("Each XMLConfigBuilder can only be used once.");
    }
    parsed = true;
    parseConfiguration(parser.evalNode("/configuration"));
    return configuration;
}

private void parseConfiguration(XNode root) {
    try {
      //issue #117 read properties first
      propertiesElement(root.evalNode("properties"));
      Properties settings = settingsAsProperties(root.evalNode("settings"));
      loadCustomVfs(settings);
      typeAliasesElement(root.evalNode("typeAliases"));
      pluginElement(root.evalNode("plugins"));
      objectFactoryElement(root.evalNode("objectFactory"));
      objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
      reflectorFactoryElement(root.evalNode("reflectorFactory"));
      settingsElement(settings);
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

### 解析environments节点
```
private void environmentsElement(XNode context) throws Exception {
    if (context != null) {
      // 创建sqlSessionFactory时未指定默认环境
      if (environment == null) {
        // 则将environments节点default属性设置为默认环境
        environment = context.getStringAttribute("default");
      }
      
      //循环所有设置的环境
      for (XNode child : context.getChildren()) {
        String id = child.getStringAttribute("id");
        //只对指定的环境进行加载
        if (isSpecifiedEnvironment(id)) {
          //1.创建事务工厂 TransactionFactory 
          TransactionFactory txFactory = transactionManagerElement(child.evalNode("transactionManager"));
          DataSourceFactory dsFactory = dataSourceElement(child.evalNode("dataSource"));
          //2.创建数据源DataSource 
          DataSource dataSource = dsFactory.getDataSource();
          //3. 构造Environment对象  
          Environment.Builder environmentBuilder = new Environment.Builder(id)
              .transactionFactory(txFactory)
              .dataSource(dataSource);
          //4. 将创建的Envronment对象设置到configuration 对象中  
          configuration.setEnvironment(environmentBuilder.build());
        }
      }
    }
}
```

  [1]: http://static.zybuluo.com/hewei0928/l6mjmr5m71jx99a3x8anf0lu/image_1ceo09j2j2b7jso1hf5lp1tvf9.png