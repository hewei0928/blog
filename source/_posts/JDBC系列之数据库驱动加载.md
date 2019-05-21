---
title: JDBC系列之数据库驱动加载
date: 2018-01-16 22:34:51
tags:
    - JDBC
    - 数据库
---

> 最近公司的数据库模板框架代码，发现自己mybatis用多了，JDBC知识忘得快差不多了。所以把JDBC翻出来，好好重新学习一番。

## 一、 概述

&ensp;&ensp;在应用程序中进行数据库连接，调用JDBC接口，首先要将特定厂商的JDBC驱动实现加载到系统内存中，然后供系统使用。基本结构图如下：
<!--more-->
![image_1c3vmn8lc1pt1rtt1ud012p1ma9p.png-25.3kB][1]




  
## 二、 驱动加载入内存的过程
&espn;&espn; 所谓的数据库驱动，其实就是实现了`java.sql.Driver`接口的类，例如`Mysql`中就是`com.mysql.jdbc.Driver`。
```java
public class Driver extends NonRegisteringDriver implements java.sql.Driver {
```

因此将数据库驱动加载入内存即将类加载至内存：`Class.forName("driverName")`

数据库驱动类内都有一个静态代码块，在驱动类加载时，会运行其中的代码。代码段中，会创建一个驱动`Driver`的实例，放入`DriverManager`中，供`DriverManager`使用。例如`mysql`:
```java
public class Driver extends NonRegisteringDriver implements java.sql.Driver {
    public Driver() throws SQLException {
    }

    static {
        try {
            DriverManager.registerDriver(new Driver());
        } catch (SQLException var1) {
            throw new RuntimeException("Can't register driver!");
        }
    }
}

```


## 三、 Driver
`Driver`中最主要的方法有两个：acceptsURL(String url），connect（String url,Properties info）。

### acceptsURL
&espn;&espn;`acceptsURL`用于检测指定的的URL是否符合数据库的协议，只有符合自己的协议形式的url才被认为能够打开这个url，如果能够打开，返回true，反之，返回false。
例如`mysql`的url协议如下：
`jdbc:mysql://<host>:<port>/<database_name>?property1=value1&property2=value2`

`mysql`自己的`acceptsURL`在类`NonRegisteringDriver`内:
```
public boolean acceptsURL(String url) throws SQLException {
    if (url == null) {
        throw SQLError.createSQLException(Messages.getString("NonRegisteringDriver.1"), "08001", (ExceptionInterceptor)null);
    } else {
        return this.parseURL(url, (Properties)null) != null;
    }
}


public Properties parseURL(String url, Properties defaults) throws SQLException {
        Properties urlProps = defaults != null ? new Properties(defaults) : new Properties();
        if (url == null) {
            return null;
        } else if (!StringUtils.startsWithIgnoreCase(url, "jdbc:mysql://") && !StringUtils.startsWithIgnoreCase(url, "jdbc:mysql:mxj://") && !StringUtils.startsWithIgnoreCase(url, "jdbc:mysql:loadbalance://") && !StringUtils.startsWithIgnoreCase(url, "jdbc:mysql:replication://")) {
            return null;
        } else {
            int beginningOfSlashes = url.indexOf("//");
            if (StringUtils.startsWithIgnoreCase(url, "jdbc:mysql:mxj://")) {
                urlProps.setProperty("socketFactory", "com.mysql.management.driverlaunched.ServerLauncherSocketFactory");
            }

            int index = url.indexOf("?");
            String hostStuff;
            String configNames;
            //...省略部分代码
}
```

**拓展阅读:** [常用数据库URL Driver][2]

### connect()
&espn;&espn;`connect（String url,Properties info）`方法，创建`Connection`对象，用来和数据库的数据操作和交互，而Connection则是真正数据库操作的开始。

**手动加载驱动 Driver 并实例化进行数据库操作的例子**
```
//1. 加载mysql驱动类并实例化
Driver driver = (Driver) Class.forName("com.mysql.jdbc.Driver").newInstance();

String url = "jdbc:mysql://localhost:3306/hewei?useUnicode=true&characterEncoding=utf-8&useSSL=true";

//2. 测试指定的Url是否符合mysql协议
boolean accept = driver.acceptsURL(url);

if (accept) {
    //3. 创建真实数据库连接
    Properties properties = new Properties();
    properties.put("user", "root");
    properties.put("password", "hewei0928");

    Connection connection = driver.connect(url, properties);
    log.info(connection.toString());
    try{

    } finally {
        connection.close();
    }
}
```

如果现在我们加载进来了多个驱动`Driver`，那么手动创建`Driver`实例，并根据URL进行创建连接就会显得代码杂乱无章，并且还容易出错，并且不方便管理。JDBC中提供了一个`DriverManager`角色，用来管理这些驱动`Driver`。

## 四、 DriverManager


  [1]: http://static.zybuluo.com/hewei0928/zjpikt795p84gyrzl3bb6yri/image_1c3vmn8lc1pt1rtt1ud012p1ma9p.png
  [2]: http://blog.csdn.net/ring0hx/article/details/6152528