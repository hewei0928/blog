---
title: 《深入理解mybatis原理》一级缓存
date: 2018-06-04 16:36:49
tags:
    - Mybatis
---

> `MyBatis`提供了一级缓存、二级缓存 这两个缓存机制，能够很好地处理和维护缓存，以提高系统的性能。这里主要介绍一级缓存，深入源码，解析其实现原理。

<!--more-->


## 一级缓存介绍及用处
    
&ensp;&ensp;每当我们使用`MyBatis`开启一次和数据库的会话，`MyBatis`会创建出一个`SqlSession`对象表示一次数据库会话。

&ensp;&ensp;在一次数据库会话中，可能会反复执行完全相同的`sql`查询，在大数据量的情况下，这会造成极大的资源浪费。为了解决这一问题，减少资源的浪费，`MyBatis`会在表示会话的`SqlSession`对象中建立一个简单的缓存，将每次查询到的结果结果缓存起来。当下次查询的时候，如果判断先前有个完全一样的查询，会直接从缓存中直接将结果取出，返回给用户，不需要再进行一次数据库查询了。

&ensp;&ensp;如下图所示，`mybatis`在一次会话（即一个`SqlSession`对象）中，会创建一个本地存储。对于每一次**查询**,都会尝试根据查询的条件去本地缓存中查找是否存在，如果存在则直接返回返回；否则则去查询数据库并将结果存入。
![微信截图_20180604170008.png-18.1kB][1]


## Myabtis中的一级缓存结构

&ensp;&ensp;之前看到`Mybatis`利用动态代理使用`MapperMethod.execute`来执行所有的数据库操作。实际上继续往下阅读可以发现最底层是调用了`SqlSession`中`Executor`对象的相关方法。
&ensp;&ensp;当创建了一个`SqlSession`对象时，会在内部创建一个`Executor`执行器对象，缓存信息`Cache`就存储在这个对象中。SqlSession、Executor、Cache之间的关系如下列类图所示：

![微信截图_20180604172201.png-84.7kB][2]

&ensp;&ensp;如上述的类图所示，`Executor`接口的实现类`BaseExecutor`中拥有一个`Cache`接口的实现类`PerpetualCache`，来实现对缓存的维护。

下面是`PeroetualCache`的源码：

```java
public class PerpetualCache implements Cache {

  private final String id;

  private Map<Object, Object> cache = new HashMap<Object, Object>();

  public PerpetualCache(String id) {
    this.id = id;
  }

  @Override
  public String getId() {
    return id;
  }

  @Override
  public int getSize() {
    return cache.size();
  }

  @Override
  public void putObject(Object key, Object value) {
    cache.put(key, value);
  }

  @Override
  public Object getObject(Object key) {
    return cache.get(key);
  }

  @Override
  public Object removeObject(Object key) {
    return cache.remove(key);
  }

  @Override
  public void clear() {
    cache.clear();
  }

  @Override
  public ReadWriteLock getReadWriteLock() {
    return null;
  }

  @Override
  public boolean equals(Object o) {
    if (getId() == null) {
      throw new CacheException("Cache instances require an ID.");
    }
    if (this == o) {
      return true;
    }
    if (!(o instanceof Cache)) {
      return false;
    }

    Cache otherCache = (Cache) o;
    return getId().equals(otherCache.getId());
  }

  @Override
  public int hashCode() {
    if (getId() == null) {
      throw new CacheException("Cache instances require an ID.");
    }
    return getId().hashCode();
  }

}
```

可以发现源码十分简单，就是用一个`Map`来实现存储，`key`值为一次查询的唯一标识，`value`则为一次查询的结果。

## 一级缓存生命周期

1. `MyBatis`在开启一个数据库会话时，会创建一个新的`SqlSession`对象，`SqlSession`对象中会有一个新的`Executor`对象，`Executor`对象中持有一个新的`PerpetualCache`对象；当会话结束时，`SqlSession`对象及其内部`的Executor`对象还有`PerpetualCache`对象也一并释放掉。
2. 如果`SqlSession`调用了`close()`方法，会释放掉一级缓存`PerpetualCache`对象，一级缓存将不可用；
3. 如果`SqlSession`调用了`clearCache()`，会清空`PerpetualCache`对象中的数据，但是该对象仍可使用；
4. **`SqlSession`中执行了任何一个`update`操作(`update()、delete()、insert()`) ，都会清空`PerpetualCache`对象的数据，但是该对象可以继续使用；**

## 一级缓存唯一标识（CacheKey）

&ensp;&ensp;`Mybatis`会对同一次会话中的相同查询进行缓存，那么`Mybatis`是根据什么条件判断两次查询相同呢。
&ensp;&ensp;以下为判断部分源码：
```java
@Override
  public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
    BoundSql boundSql = ms.getBoundSql(parameter);
    CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);
    return query(ms, parameter, rowBounds, resultHandler, key, boundSql);
}
```
可以看到`createCacheKey`方法有四个参数，也就是对应的判断条件：
1. 传入的`statementId`(即接口名+方法名)
2. 查询时要求的结果集中的结果范围 
3. 这次查询所产生的最终要传递给JDBC `java.sql.Preparedstatement`的Sql语句字符串（boundSql.getSql()）
4. 传递给`java.sql.Statement`要设置的参数值

**`MyBatis`认为的完全相同的查询，不是指使用`sqlSession`查询时传递给`SqlSession`的所有参数值完完全全相同，你只要保证`statementId`，`rowBounds`,最后生成的`SQL`语句，以及这个`SQL`语句所需要的参数完全一致就可以了。**



  [1]: http://static.zybuluo.com/hewei0928/8pew0b2piv3fx2ldpn7myqeo/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20180604170008.png
  [2]: http://static.zybuluo.com/hewei0928/1vh1ud7dpt3dou6lsq2gk7yy/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20180604172201.png