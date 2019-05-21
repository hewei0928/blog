---
layout: 
title: 《深入理解mybatis原理》 Mapper接口绑定及代理
date: 2018-06-01 14:27:56
tags:
    - Mybatis
---

> 当定义好一个`Mapper接口(UserDao)`里，我们并不需要去实现这个类，但`sqlSession.getMapper()`最终会返回一个实现该接口的对象。这个对象是`Mybatis`利用`jdk`的动态代理实现的。这里将介绍这个代理对象的生成过程及其方法的实现过程。

<!--more-->

## Mapper 接口获取与注册添加

&ensp;&ensp;一般`mybatis`项目实例化对应`Mapper`接口:
```java
SqlSession sqlSession =MySqlSessionFactory.openSession();
try {
    StudentMapper studentMapper =
            sqlSession.getMapper(StudentMapper.class);
    studentMapper.insertStudent(student);
    sqlSession.commit();
} finally {
    sqlSession.close();
}
```

**sqlSession.getMapper(StudentMapper.class);**底层调用链为：`DefultSqlSession.getMapper()` -> `Configuration.getMapper()` -> `MapperRegistry.getMapper()`:
```java
// DefultSqlSession.getMapper()
public <T> T getMapper(Class<T> type) {
    return this.configuration.getMapper(type, this);
}

//Configuration.getMapper()
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    return mapperRegistry.getMapper(type, sqlSession);
}
```

`Mybatis`初始化时会在读取mapper.xml文件后会调用`org.apache.ibatis.builder.xml.XMLMapperBuilder`类的`bindMapperForNamespace()`方法，绑定到命名空间：
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

//Configuration.addMapper()
public <T> void addMapper(Class<T> type) {
    this.mapperRegistry.addMapper(type);
}
```

发现添加和获取Mapper实例都使用到了同一个类`MapperRegistry`，在`Configuration`中的声明如下：
```java
protected final MapperRegistry mapperRegistry = new MapperRegistry(this);
```

## MapperRegistry
`MapperRegistry`源码：
```java
public class MapperRegistry {

  private final Configuration config;
  private final Map<Class<?>, MapperProxyFactory<?>> knownMappers = new HashMap<Class<?>, MapperProxyFactory<?>>();

  public MapperRegistry(Configuration config) {
    this.config = config;
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
  
  public <T> boolean hasMapper(Class<T> type) {
    return knownMappers.containsKey(type);
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
        ////解析接口上的注解或者加载mapper配置文件生成mappedStatement
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

  /**
   * @since 3.2.2
   */
  public Collection<Class<?>> getMappers() {
    return Collections.unmodifiableCollection(knownMappers.keySet());
  }

  /**
   * @since 3.2.2
   */
  public void addMappers(String packageName, Class<?> superType) {
    ResolverUtil<Class<?>> resolverUtil = new ResolverUtil<Class<?>>();
    resolverUtil.find(new ResolverUtil.IsA(superType), packageName);
    Set<Class<? extends Class<?>>> mapperSet = resolverUtil.getClasses();
    for (Class<?> mapperClass : mapperSet) {
      addMapper(mapperClass);
    }
  }

  /**
   * @since 3.2.2
   */
  public void addMappers(String packageName) {
    addMappers(packageName, Object.class);
  }
  
}
```

`MapperRegistry`有多个`addMapper`方法，这里主要解析`public <T> void addMapper(Class<T> type)`和`public <T> T getMapper(Class<T> type, SqlSession sqlSession)`

### addMapper
```java
public <T> void addMapper(Class<T> type) {
    // 必须是接口类型
    if (type.isInterface()) {
      // 不能重复注册接口
      if (hasMapper(type)) {
        throw new BindingException("Type " + type + " is already known to the MapperRegistry.");
      }
      boolean loadCompleted = false;
      try {
        //在Map中添加接口及 接口代理类工厂对象
        knownMappers.put(type, new MapperProxyFactory<T>(type));
        //解析接口上的注解或者加载mapper配置文件生成mappedStatement (一般不使用)
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
主要分为以下几步：
1. 验证要添加的映射器的类型是否是接口，如果不是接口则结束添加，如果是接口则执行下一步
2. 验证注册器集合中是否已存在该注册器（即重复注册验证），如果已存在则抛出绑定异常，否则执行下一步
3. 定义一个`boolean`值，默认为`false`
4. 执行`HashMap`集合的`put`方法，将该映射器注册到注册器中：以该接口类型为键，以接口类型为参数调用`MapperProxyFactory`的构造器创建的映射器代理工厂为值
5. 然后对使用注解方式实现的映射器进行注册（一般不使用）
6. 设置第三步的`boolean`值为`true`，表示注册完成
7. 在`finally`语句块中对注册失败的类型进行清除

步骤十分简单，重点在`MapperProxyFactory`代理工厂类的实现。

### getMapper
`addMapper`在`MapperRegistry`注册了对应接口，`getMapper`就可以去获取对应接口实例：

```java
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    //获取对应代理工厂类
    final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
    if (mapperProxyFactory == null) {
      throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
    }
    try {
      //动态代理实现接口对应实现类
      return mapperProxyFactory.newInstance(sqlSession);
    } catch (Exception e) {
      throw new BindingException("Error getting mapper instance. Cause: " + e, e);
    }
}
```

## MapperProxyFactory 和 MapperProxy

    `MapperRegistry`中在`map`中存储了接口及对应代理工厂类：`knownMappers.put(type, new MapperProxyFactory<T>(type));`；之后又利用该代理工厂去动态代理对应接口实例。


```java
public class MapperProxyFactory<T> {

  private final Class<T> mapperInterface;
  //MapperProxy中的方法缓存
  private final Map<Method, MapperMethod> methodCache = new ConcurrentHashMap<Method, MapperMethod>();

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
    //利用jdk的动态代理生成对应接口类
    return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
  }

  public T newInstance(SqlSession sqlSession) {
    // 实现了InvocationHandler接口的对象
    final MapperProxy<T> mapperProxy = new MapperProxy<T>(sqlSession, mapperInterface, methodCache);
    return newInstance(mapperProxy);
  }
}
```

`MapperProxyFactory`利用动态代理及工厂模式生成`Mapper`接口的实例，具体代理类则是`MapperProxy`:

```java
public class MapperProxy<T> implements InvocationHandler, Serializable {

  private static final long serialVersionUID = -6424540398559729838L;
  private final SqlSession sqlSession;
  private final Class<T> mapperInterface;
  //接口中方法的缓存
  private final Map<Method, MapperMethod> methodCache;

  public MapperProxy(SqlSession sqlSession, Class<T> mapperInterface, Map<Method, MapperMethod> methodCache) {
    this.sqlSession = sqlSession;
    this.mapperInterface = mapperInterface;
    this.methodCache = methodCache;
  }

  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    
    //判断是不是基础方法 比如toString() hashCode()等，这些方法直接调用不需要处理
    try {
      if (Object.class.equals(method.getDeclaringClass())) {
        return method.invoke(this, args);
      } else if (isDefaultMethod(method)) {
        return invokeDefaultMethod(proxy, method, args);
      }
    } catch (Throwable t) {
      throw ExceptionUtil.unwrapThrowable(t);
    }
    
    //对方法进行缓存，再次调用时不再实例化该对象
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    return mapperMethod.execute(sqlSession, args);
  }

  private MapperMethod cachedMapperMethod(Method method) {
    MapperMethod mapperMethod = methodCache.get(method);
    if (mapperMethod == null) {
      mapperMethod = new MapperMethod(mapperInterface, method, sqlSession.getConfiguration());
      methodCache.put(method, mapperMethod);
    }
    return mapperMethod;
  }

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
可以看到，所有`mapper`接口的方法经过代理后，最后都是执行了`MapperMethod`中的`execute(sqlSession, args)`方法。而`MapperMethod`也是整个代理机制中最重要的部分，它对对`Sqlsession`当中的操作进行了封装。

## MapperMethod与MappedStatement实现方法与标签的绑定

### 初始化

`mybatis`初始化时会对`mybatis-config.xml`文件进行解析，在这一步中，会解析`<mappers>`标签中指向的`mapper.xml`文件。之后会根据`xml`的`namespace`和各个标签属性(select、delete、update、insert)，及标签id创建`MapperStatement`对象列表，并存储至`Configuration`对象中。

**方法调用链表如下：**

`SqlSessionFactoryBuilder.builder()` -> `XMLConfigBuilder.parse()` -> `XMLConfigBuilder.parseConfiguration()` -> `XMLConfigBuilder.mapperElement()` -> `XMLMapperBuilder.parse()` -> `XMLMapperBuilder.configurationElement()` -> `XMLMapperBuilder.buildStatementFromContext()` -> `XMLMapperBuilder.buildStatementFromContext()` -> `XMLStatementBuilder.parseStatementNode()` -> `MapperBuilderAssistant.addMappedStatement()` -> `Configuration.addMappedStatement()`

在调用`MapperBuilderAssistant.addMappedStatement`时，会将`mapper.xml`文件的`namespace`及其中标签的`id`创建存入`MappedStatement`对象的id中。然后`configuration.addMappedStatement`会以`MappedStatement`的id为key，`MappedStatement`对象为value存入map中。


### 调用

之前已经讲到，所有`Mapper`接口方法都由`MapperProxy`动态代理。在invoke方法中会创建`MapperMethod`对象：
```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      if (Object.class.equals(method.getDeclaringClass())) {
        return method.invoke(this, args);
      } else if (isDefaultMethod(method)) {
        return invokeDefaultMethod(proxy, method, args);
      }
    } catch (Throwable t) {
      throw ExceptionUtil.unwrapThrowable(t);
    }
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    return mapperMethod.execute(sqlSession, args);
  }

  private MapperMethod cachedMapperMethod(Method method) {
    MapperMethod mapperMethod = methodCache.get(method);
    if (mapperMethod == null) {
      mapperMethod = new MapperMethod(mapperInterface, method, sqlSession.getConfiguration());
      methodCache.put(method, mapperMethod);
    }
    return mapperMethod;
}
```

初始化`MapperMethod`对象时，会初始化其内部类`SqlCommand`:

```java
public MapperMethod(Class<?> mapperInterface, Method method, Configuration config) {
    this.command = new SqlCommand(config, mapperInterface, method);
    this.method = new MethodSignature(config, mapperInterface, method);
}
```

初始化`SqlCommand`对象时，会根据`mapper`接口类名及方法名去获取对应`MappedStatement`对象，实现一一对应。
```java
public SqlCommand(Configuration configuration, Class<?> mapperInterface, Method method) {
      final String methodName = method.getName();
      final Class<?> declaringClass = method.getDeclaringClass();
      MappedStatement ms = resolveMappedStatement(mapperInterface, methodName, declaringClass,
          configuration);
      if (ms == null) {
        if (method.getAnnotation(Flush.class) != null) {
          name = null;
          type = SqlCommandType.FLUSH;
        } else {
          throw new BindingException("Invalid bound statement (not found): "
              + mapperInterface.getName() + "." + methodName);
        }
      } else {
        //com.sunyard.hw.mysql.testMybatis.asd.StudentMapper.insertStudent
        name = ms.getId();
        // INSERT
        type = ms.getSqlCommandType();
        if (type == SqlCommandType.UNKNOWN) {
          throw new BindingException("Unknown execution method for: " + name);
        }
      }
}
```







