---
title: Spring之Bean装配
date: 2018-06-10 11:05:26
tags:
    - Java
    - Spring 
---

> Spring 的一大重点就是IOC与DI, 这里根据《Spring in Action》以及之前使用Spring的经验来对其进行一个总结。

<!--more-->

## XML装配

&ensp;&ensp;`XML`装配`Bean`是`Spring`刚出现时的主要方式。尽管现如今已经有了更加简单方便的自动装配，但在实际项目中还是会遇到许多基于`XML`的`Spring`配置，因此总结和理解`XML`配置还是很重要的。

### XML配置文件

&ensp;&ensp;使用`XML`装配首先要做的就是创建`XML`配置文件，并且以`<beans>`元素为根。最基础的`XML`配置文件如下：
```xml
<?xml version="1.0" encoding="UTF-8"?>  
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:p="http://www.springframework.org/schema/p"
       xmlns:c="http://www.springframework.org/schema/c"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
             http://www.springframework.org/schema/beans/spring-beans.xsd
         http://www.springframework.org/schema/context
         http://www.springframework.org/schema/context/spring-context-4.0.xsd">

</beans> 
```

### 声名Bean

下面这条语句便是最简单的声名了一个`Bean`：
```xml
<bean class="base.SgtCD"></bean>
<bean class="base.SgtCD"></bean>
```
`class`属性指定为具体类的**全限定类名**。因为没有指明`Id`,则`Spring`会将第一个`Bean`的id设为`base.SgtCD#0`, 第二个设为`base.SgtCD#1`。当时用`getBean()`方法去获取时，具体情况如下所示：

```java
public static void main(String[] args) {
    ApplicationContext ctx = new ClassPathXmlApplicationContext(
            "configuration.xml");
    SgtCD s0 = (SgtCD) ctx.getBean("base.SgtCD");
    SgtCD s = (SgtCD) ctx.getBean("base.SgtCD#0");
    SgtCD s1 = (SgtCD) ctx.getBean("base.SgtCD#1");
    // 294247762 294247762  918312414
    System.out.println(s0.hashCode() + " " + s.hashCode()+ "  "+ s1.hashCode());
}
```

因此，声名`Bean`时最好使用`id`属性对`bean`进行命名。
```xml
<bean id="sgtcd1" class="base.SgtCD"></bean>
<bean id="sgtcd2" class="base.SgtCD"></bean>
```

### 构造器注入

上述声名`Bean`的方式`Spring`会默认调用无参构造器，如果对应的类没有无参构造器，则会报`BeanCreationException`。因此需要在声明时指定构造器。具体方法有以下两种：
1. `<constructor-arg>`元素
2. 使用Spring 3.0所引入的c-命名空间

#### constructor-arg

**字面量参数：**
修改`SgtCD`类，使其构造器如下所示
```java
public SgtCD(String title, String artist) {
    this.title = title;
    this.artist = artist;
}
```
则当对进行声名时，使用`<constructor-arg>`写法如下：
```xml
<bean id="s" class="base.SgtCD">
    <constructor-arg name="artist" type="java.lang.String" value="LLL"></constructor-arg>
    <constructor-arg name="title" type="java.lang.String" value="SSS"></constructor-arg>
</bean>
```
`name`属性与构造器中的参数名一一对应，`type`指定参数的类型，`value`指定对应参数值。当然`name`属性可以用`index`进行替代，使用`0,1,2`三来对应参数所在位置的索引值：
```xml
<bean id="s" class="base.SgtCD">
    <constructor-arg index="0" type="java.lang.String" value="LLL"></constructor-arg>
    <constructor-arg index="1" type="java.lang.String" value="SSS"></constructor-arg>
</bean>
```

**自定义对象参数**
如果构造器中的参数不是`String`或其他基础类型值，则可以用`ref`属性来指定另一个已经装配的`Bean`作为参数传入：
```xml
<bean id="s" class="base.SgtCD">
    <constructor-arg index="0" ref="sss"></constructor-arg>
    <constructor-arg index="1" ref="sss"></constructor-arg>
</bean>

<bean id="sss" class="java.lang.String">
    <constructor-arg index="0" type="java.lang.String" value="LLL"></constructor-arg>
</bean>
```

**集合对象**
如果构造器中有一个参数为集合,那么就需要使用`<list>`、`<set>`、`<map>`标签对其注入：
```xml

<constructor-arg index="2">
    <list>
        <value></value>
        <value></value>
    </list>
</constructor-arg>

<!--指定bean类型结合-->
<constructor-arg index="2">
    <list>
        <ref bean=""></ref>
        <ref bean=""></ref>
    </list>
</constructor-arg>



<constructor-arg index="2">
    <set>
        <value></value>
        <value></value>
    </set>
</constructor-arg>

<constructor-arg index="2">
    <map>
        <entry key="" value=""></entry>
        <entry key="" value=""></entry>
    </map>
</constructor-arg>
```

#### c- 命名空间

**字面量参数**
```xml
//可以使用下标或参数名
<bean id="s1" class="base.SgtCD" c:_0="asd" c:artist="asdsad"/>
```

**自定义类型**
```xml
//可以使用参数下标，或参数名称
<bean id="s1" class="base.SgtCD" c:_0-ref="sss" c:artist-ref="sss"/>

<bean id="sss" class="java.lang.String">
    <constructor-arg index="0" type="java.lang.String" value="LLL"></constructor-arg>
</bean>
```

**集合类型**
c- 命名空间不支持集合命名空间


### 设置属性

除了使用构造器注入，还可以使用属性`<property>`和p-命名空间对其成员变量进行设置。
#### property设置
```xml

//可以设置字面变量
<bean id="s1" class="base.SgtCD" c:_0="asd" c:artist="sss">
    <property name="artist" value="aaaa"></property>
</bean>

//也可以指定具体bean
<bean id="s1" class="base.SgtCD" c:_0="asd" c:artist="aaa">
    <property name="artist" ref="sss"></property>
</bean>
<bean id="sss" class="java.lang.String">
    <constructor-arg index="0" type="java.lang.String" value="LLL"></constructor-arg>
</bean>
```
注意：
1. `property`标签的`name`属性与成员变量名对应
2. 对应的属性需要有对应的`setter`方法
3. `property`中的`value`属性只是将值传给`setter`方法，最终属性值与`setter`方法中的逻辑有关。
4. `property`中设置的属性会覆盖构造器中传入的属性。类似于`new`了对象之后在调用`setter`方法
5. 集合类型的成员变量设置与构造器注入类似

#### p- 命名空间设置
```xml
//字面量属性设置
<bean id="s1" class="base.SgtCD" c:_0="asd" c:artist="aaa" p:artist="asd"></bean>

//指定具体bean
<bean id="s1" class="base.SgtCD" c:_0="asd" c:artist="aaa" p:artist-ref="sss"></bean>
<bean id="sss" class="java.lang.String">
    <constructor-arg index="0" type="java.lang.String" value="LLL"></constructor-arg>
</bean>
```
**注意**：
1. p-命名空间可以对集合变量进行设置：结合`<util:list>`、`<util:set>`、`<util:map>`即可


## Java代码装配

&ensp;&ensp;现在许多`Spring`项目已经使用自动装配实现，但是有时候自动装配并不一定能用。例如使用第三方类库时，无法对其中的类添加`@Compent`注解，只能使用手动注入的方式。除了`XML`装配之外，最常用的就是`Java`代码装配。

### 创建配置类
类似于`XML`装配中的创建`XML`配置文件，使用`Java`代码装配首先需要创建配置类：
```java
@Configuration //表明这个类是一个配置类
public class CDPlayerConfig {

}
```

### 声明bean

在`config`类中声明`Bean`只需创建如下方法：

```java
@Bean
public CD cd(){
    return new MovieCD();
}
```
在方法上添加`@Bean`,即把方法的返回值声明为`Bean`添加至`Spring`的上下文中。类似于`XML`配置中的`bean>`标签，只不过更加简洁灵活。
另外，用这种方法创建的`bean`的Id与方法名相同。自定义`id`对其设置`name`属性：
```java
@Bean(name = "cds")
public CD cd(){
    return new MovieCD();
}
```
这之后就可以利用`AnnotationConfigApplicationContext`来加载配置类，获取`Bean`:
```
public static void main(String[] args) {
    ApplicationContext applicationContext = new AnnotationConfigApplicationContext(CDPlayerConfig.class);
    //有三种获取方法：
    CD cd = (CD)applicationContext.getBean("cd");
    CD cd1 = applicationContext.getBean(CD.class);
    CD cd2 = applicationContext.getBean("cd", CD.class);
    System.out.println(cd1 == cd2);// true获取的都是同一个对象
    System.out.println(cd == cd1);
}
```
**注意**：使用`CD cd1 = applicationContext.getBean(CD.class);`去获取注入的`Bean`时，`Spring`上下文中只能有一个对应类型的对象，否则会报`NoUniqueBeanDefinitionException`。

### Bean装配

对于几个`Bean`的装配，可以利用直接引用创建`bean`的方法来实现：
```java
@Bean(name = "cd")
public CD cd(){
    return new MovieCD();
}

@Bean
public CDPlayer p1(){
    return new CDPlayer(cd());
}
```

**需要注意的是**：看似`CDPlayer`的构造方法调用了`cd()`方法。但其实由于添加了`@Bean`注解，`Spring`会拦截对其的调用，并确保每次调用返回相同的对象：
```java
@Configuration //表明这个类是一个配置类
public class CDPlayerConfig {

    @Bean(name = "cd")
    public CD cd(){
        return new MovieCD();
    }

    @Bean
    public CDPlayer p1(){
        System.out.println(cd());
        return new CDPlayer(cd());
    }

    @Bean
    public CDPlayer p2(){
        return new CDPlayer(cd());
    }


    public static void main(String[] args) {
        ApplicationContext applicationContext = new AnnotationConfigApplicationContext(CDPlayerConfig.class);
        // true
        System.out.println(applicationContext.getBean("p1",CDPlayer.class).getCd() == applicationContext.getBean("p2", CDPlayer.class).getCd());
    }

}
```

除了直接调用创建`bean`的方法外，还有另外一种更为简便常用的方法：
```java
@Bean(name = "cd")
public CD cd(){
    return new MovieCD();
}

//直接将bean对象当做方法参数
@Bean
public CDPlayer p1(CD cd){
    return new CDPlayer(cd);
}
```
当`Spring`调用p1方法创建`CDPlayer`对象时，它会自动将上下文中的一个`CD`对象装配入方法内。如果上下文中没有对应对象，则编译无法通过。

## 自动装配Bean

### 创建配置类
&ensp;&ensp;与基于`Java`的配置相类似，先创建配置类。不过不同的是会为配置类添加`@CompentScan`注解：
```java
@Configuration //表明这个类是一个配置类
@ComponentScan //设置自动扫描包
public class CDPlayerConfig {

}
```
`@CompentScan`这个注解会在`Spring`中启用注解扫描。如果不设置其他属性，会默认扫描配置类所在的包及其子包中的`bean`。也可以使用`basePackages = {}`或 `basePackageClasses = {xx.class, xx.class}`设置**一个或多个**默认扫描包。
在`XML`文件中添加`<context:component-scan>`标签也可以为`XML`配置开启自动扫描。


### 创建可被发现的Bean

对于希望被自动扫描发现的类，为其添加`@Compent`注解，同时可以为其命名：
```java
@Component(value = "CD")
public class CDPlayer implements MediaPlayer {
}
```

### 自动装配

`Spring`扫描到带`@Compent`注解的类会默认调用其无参构造方法。如果我们要对某个`Bean`实现自动装配。有两种实现方式：
1. 指定`Spring`生成该类时调用的构造器，可以为指定构造器添加`@Autowried`注解,`Spring`会自动将上下文中的`bean`装配入构造其中。**但是只能为一个构造方法添加该注解，以及无参构造方法不能添加该注解。**
```java
@Component(value = "CD")
public class CDPlayer implements MediaPlayer {

    private CD cd;

//    @Autowired 添加此注解则运行时报IllegalStateException
    public CDPlayer() {
    }

    @Autowired
    public CDPlayer(CD cd){
        this.cd = cd;
    }

//    @Autowired 同时有两个无参构造器添加此注解则编译错误, 有多个Bean要装配可以使用@Qualifier注解
    public CDPlayer(CD cd, CD cd1){
        this.cd = cd;
    }
}
```

2.  为`setter`方法添加`@Autowried`注解（**可以再任意方法上添加`@Autowried`注解实现该功能**）
```java
@Component(value = "CD")
public class CDPlayer implements MediaPlayer {

    private CD cd;
    
    public CDPlayer() {
    }
    
    @Autowired
    public void setCd(CD cd) {
        this.cd = cd;
    }
}
```
会在实例化`Bean`之后调用`set`方法装配对象：

```
@Component(value = "CD")
public class CDPlayer implements MediaPlayer {

    private CD cd;


    @Autowired
    public CDPlayer(@Qualifier(value = "sgtCD") CD cd){
        this.cd = cd;
    }

    @Override
    public void play() {
        this.cd.play();
    }



    @Autowired
    @Qualifier(value = "movieCD")//指定具体的Bean
    public void setCd(CD cd) {
        this.cd = cd;
    }
}

public static void main(String[] args) {
        ApplicationContext applicationContext = new AnnotationConfigApplicationContext(CDPlayerConfig.class);
        //base.MovieCD@11fc564b
        System.out.println(applicationContext.getBean(CDPlayer.class).getCd());
}
```
可以看到，最后`CD`的具体类型是`MovieCD`。证明`setter`上的装配发生在构造器装配之后。


## 多种配置混合使用

### JavaConfig中引入其他配置

1. 使用`@Import`引入另一个`JavaConfig`配置
2. 使用`@ImportResource`引入另一个`XML`配置

### XML配置中引入其他配置
1. 使用`<import>`引入另一个`XML`配置
2. 使用`<bean>`引入另一个`JavaConfig`配置













