---
title: web.xml详解
date: 2018-06-11 21:41:35
tags:
    - Java
    - Spring MVC
---

> 一般的Java Web项目，不论是传统Servlet项目还是Spring mvc项目，web.xml的配置都是重要的一环。

<!--more-->

## 常用标签汇总

&ensp;&ensp;`web.xml`中有许多标签，这里列举一些常用的标签
```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"   
        xmlns="http://java.sun.com/xml/ns/javaee"
        xmlns:web="http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd"   
        xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd" 
        version="3.0">
    
    <!-- display-name元素提供GUI工具可能会用来标记这个特定的Web应用的一个名称。 -->
    <display-name></display-name>
        
    <!-- context-param元素声明应用范围内的初始化参数 -->
    <context-param></context-param>
    
    <!-- filter 过滤器元素将一个名字与一个实现javax.servlet.Filter接口的类相关联。 -->
    <filter></filter>
    
    <!-- filter-mapping 一旦命名了一个过滤器，就要利用filter-mapping元素把它与一个或多个servlet或JSP页面相关联。 -->
    <filter-mapping></filter-mapping>
    
    <!-- listener 对事件监听程序的支持，事件监听程序在建立、修改和删除会话或servlet环境时得到通知。Listener元素指出事件监听程序类。 -->
    <listener></listener>
    
    <!-- servlet 在向servlet或JSP页面制定初始化参数或定制URL时，必须首先命名servlet或JSP页面。Servlet元素就是用来完成此项任务的。 -->
    <servlet></servlet>
    
    <!-- servlet-mapping 服务器一般为servlet提供一个缺省的URL：http://host/webAppPrefix/servlet/ServletName。但是，常常会更改这个URL，以便servlet可以访问初始化参数或更容易地处理相对URL。在更改缺省URL时，使用servlet-mapping元素。 -->
    <servlet-mapping></servlet-mapping>
    
    <!-- session-config 如果某个会话在一定时间内未被访问，服务器可以抛弃它以节省内存。可通过使用HttpSession的setMaxInactiveInterval方法明确设置单个会话对象的超时值，或者可利用session-config元素制定缺省超时值。 -->
    <session-config></session-config>
        
</web-app>
```

### context-param

`<context-param>`标签一般形式：
```xml
<context-param>
    <description></description>
	<param-name></param-name>
	<param-value></param-value>
</context-param>
```
&ensp;&ensp;启动一个`web`项目时，容器会去读取`web.xml`中的`<context-param>`标签，并创建一个`ServletContext`上下文对象。并且会将`<context-param>`标签以键值对的形式存入`ServletContext`对象中。
&ensp;&ensp;在一个`Servlet`中可以调用`getServletContext().getInitParam(name)`来获取值。

### listener

`<listener>`标签一般形式：
```xml
<listener>
    <description>描述</description>
	<listener-class>指定类</listener-class>
</listener>
```
&ensp;&ensp;`<listener-class>`中指定的类必须实现`ServletContextListener`接口,这个接口中定义了`contextInitialized(ServletContextEvent event)`和`contextDestroyed(ServletContextEvent event)`。类似于`<context-param>`, `web`项目启动时会创建`<listener></listener>`中的类实例,同时会调用`contextInitialized()`方法。可以在该方法中调用`ServletContextEvent.getgetServletContext().getInitParameter()`来获取`<context-param>`中设置的值。


