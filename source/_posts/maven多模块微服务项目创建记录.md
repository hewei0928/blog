---
layout: 'n'
title: maven多模块微服务项目创建记录
date: 2017-12-25 16:30:49
tags:
    - maven 
    - 微服务
---

> &ensp;&ensp;最近工作中需要将原先的项目进行一次重构，把各个模块封装成微服务。初次自己独立搭建maven多模块项目，而且是首次接触Dubbo和Zookeeper，踩了较多的坑终于完成了。这里记录一下大概的搭建过程以及搭建过程中踩到的坑，以便于日后的快速搭建。

<!--more-->

## 一、 基础项目及模块创建

### 1. 创建普通Maven项目

file -> new -> project -> maven -> next
    
![image_1c32p45h31d48ceeckt1ep91tfn9.png-105kB][1]


填写`GroupId`和`Artifactid`, 用于在`maven`仓库中标识项目。`GroupId`个人习惯一般创建为com加公司名缩写加个人名字缩写：com.sunyard.hw;`Artifactid`个人习惯创建为项目名。之后一路next。由于是多模块项目，删除`src`文件夹，基础maven项目创建完成。

![image_1c32q81bqdl91d4m1sfapl8jlu13.png-15.5kB][2]

### 2. 创建子模块
项目上右键 -> new -> Module -> next

![image_1c32qdjhd17m5133n46gg051p651g.png-40.3kB][3]

`ArtifactId`填项目名-模块名。一路next创建完成。类比此操作创建六大模块：`web`, `mybatisgen`, `dal`, `core`, `common`, `api`。

![image_1c39qksi71iqs17ti1i6r18jn14349.png-17.7kB][4]
    
 - `api`模块提供外部调用接口,用于搭建微服务。依赖`common`模块， 不继承主模块
 - `common`用于书写工具类，枚举类，全局异常处理等代码。为其他模块的基础依赖， 不继承主模块
 - `core` 用于书写业务逻辑层代码。依赖`common`、`dal`、`dal`模块
 - `dal` 用于书写数据库相关`mapper`,`xml`等代码。依赖`api`模块
 - `mybatisgen` 为公司`dal`层代码自动生成模块
 - `web` 书写controller层代码, 项目配置文件及项目的启动。依赖`core`模块

## 二、 maven项目pom设置

```
<properties>
    <dal.version>1.0-SNAPSHOT</dal.version>
    <api.version>1.0-SNAPSHOT</api.version>
    <core.version>1.0-SNAPSHOT</core.version>
    <common.version>1.0-SNAPSHOT</common.version>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <java.version>1.8</java.version>
    <spring.version>4.3.7.RELEASE</spring.version>
    <spring-boot-starter.version>1.5.8.RELEASE</spring-boot-starter.version>
    <spring-boot-starter-web.version>1.5.9.RELEASE</spring-boot-starter-web.version>
    <spring-boot-starter-test.version>1.5.9.RELEASE</spring-boot-starter-test.version>
    <springfox-swagger2.version>2.7.0</springfox-swagger2.version>
    <springfox-swagger-ui.version>2.7.0</springfox-swagger-ui.version>
    <mysql-connector-java.version>5.1.44</mysql-connector-java.version>
    <mybatis-spring-boot-starter.version>1.3.1</mybatis-spring-boot-starter.version>
    <alibaba-druid.version>1.0.18</alibaba-druid.version>
    <org-apache-zookeeper.version>3.4.11</org-apache-zookeeper.version>
    <alibaba-dubbo.version>2.5.8</alibaba-dubbo.version>
    <com-github-sgroschupf-zkclient.version>0.1</com-github-sgroschupf-zkclient.version>
    <commons-lang3.version>3.4</commons-lang3.version>
    <org-mybatis-generator.version>1.3.3</org-mybatis-generator.version>
    <common-io.version>2.5</common-io.version>
    <commons-lang.version>2.6</commons-lang.version>
</properties>


<dependencyManagement>
<dependencies>
    <!--==== spring boot JARS START ====-->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
        <version>${spring-boot-starter.version}</version>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
        <version>${spring-boot-starter-web.version}</version>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <version>${spring-boot-starter-test.version}</version>
        <exclusions>
            <exclusion>
                <groupId>commons-logging</groupId>
                <artifactId>commons-logging</artifactId>
            </exclusion>
        </exclusions>
    </dependency>
    <!--==== Spring Boot JARS END ====-->

    <!--==== Swagger2 JARS BEGIN ====-->
    <dependency>
        <groupId>io.springfox</groupId>
        <artifactId>springfox-swagger2</artifactId>
        <version>${springfox-swagger2.version}</version>
    </dependency>
    <dependency>
        <groupId>io.springfox</groupId>
        <artifactId>springfox-swagger-ui</artifactId>
        <version>${springfox-swagger-ui.version}</version>
    </dependency>
    <!--==== Swagger2 JARS END ====-->


    <!--==== MYSQL Jars START ====-->
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
        <version>${mysql-connector-java.version}</version>
    </dependency>
    <dependency>
        <groupId>org.mybatis.spring.boot</groupId>
        <artifactId>mybatis-spring-boot-starter</artifactId>
        <version>${mybatis-spring-boot-starter.version}</version>
    </dependency>

    <dependency>
        <groupId>org.mybatis.generator</groupId>
        <artifactId>mybatis-generator-core</artifactId>
        <version>${org-mybatis-generator.version}</version>
    </dependency>

    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>druid</artifactId>
        <version>${alibaba-druid.version}</version>
    </dependency>
    <!--==== MYSQL Jars END ====-->

    <!--==== 一方包 START ====-->
    <dependency>
        <groupId>com.sunyard.hw</groupId>
        <artifactId>announce-dal</artifactId>
        <version>${dal.version}</version>
    </dependency>
    <dependency>
        <groupId>com.sunyard.hw</groupId>
        <artifactId>announce-api</artifactId>
        <version>${api.version}</version>
    </dependency>
    <dependency>
        <groupId>com.sunyard.hw</groupId>
        <artifactId>announce-core</artifactId>
        <version>${core.version}</version>
    </dependency>
    <dependency>
        <groupId>com.sunyard.hw</groupId>
        <artifactId>announce-common</artifactId>
        <version>${common.version}</version>
    </dependency>
    <!--==== 一方包 END ====-->

    <!--==== APACHE JARS START ====-->
    <dependency>
        <groupId>org.apache.zookeeper</groupId>
        <artifactId>zookeeper</artifactId>
        <version>${org-apache-zookeeper.version}</version>
    </dependency>

    <dependency>
        <groupId>org.apache.commons</groupId>
        <artifactId>commons-lang3</artifactId>
        <version>${commons-lang3.version}</version>
    </dependency>

    <dependency>
        <groupId>commons-lang</groupId>
        <artifactId>commons-lang</artifactId>
        <version>${commons-lang.version}</version>
    </dependency>
    <!--==== APACHE JARS END ====-->

    <!--==== DUBBO JARS START ====-->
    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>dubbo</artifactId>
        <version>${alibaba-dubbo.version}</version>
    </dependency>
    <!--==== DUBBO JARS END ====-->


    <!--==== OTHER JARS START ====-->
    <dependency>
        <groupId>com.github.sgroschupf</groupId>
        <artifactId>zkclient</artifactId>
        <version>${com-github-sgroschupf-zkclient.version}</version>
    </dependency>

    <dependency>
        <groupId>commons-io</groupId>
        <artifactId>commons-io</artifactId>
        <version>${common-io.version}</version>
    </dependency>
    <!--==== OTHER JARS END ====-->
    
</dependencies>

</dependencyManagement>

```


## 三、 common 模块配置

>common用于书写工具类，枚举类，全局异常处理等代码。

### 1. 去除父模块继承，并添加子模块`groupId`、`version`

![image_1c3a4vs6jibkuvna6511611n149.png-15.8kB][5]

### 2. 添加相关依赖

&ensp;&ensp;由于其他模块都依赖于`common`模块，因此部分所有模块都用到的jar包以及`common`特有jar包都写入`Dependency`内：
```
<properties>
    <org-projectlombok-lombok.version>1.16.18</org-projectlombok-lombok.version>
    <com-sunyard-michiru-common.version>1.0.1-SNAPSHOT</com-sunyard-michiru-common.version>
    <org-springframework-spring-boot-web.version>1.5.9.RELEASE</org-springframework-spring-boot-web.version>
</properties>

<dependencies>
    <!--=== LOMBOK JARS START ===-->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <version>${org-projectlombok-lombok.version}</version>
    </dependency>
    <!--=== LOMBOK JARS END ===-->

    <!--=== MICHIRU JARS START ===-->
    <dependency>
        <groupId>com.sunyard.michiru</groupId>
        <artifactId>michiru-common</artifactId>
        <version>${com-sunyard-michiru-common.version}</version>
    </dependency>
    <!--=== MICHIRU JARS END ===-->

    <!--=== SPRING-WEB JARS START ===-->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
        <version>${org-springframework-spring-boot-web.version}</version>
    </dependency>
    <!--=== SPRING-WEB JARS END ===-->
</dependencies>
```

### 3. 模块发布管理

&ensp;&ensp;由于`api`模块依赖了`common`模块，因此`common`必须发布至`maven`仓库，以下便是相关环境配置：

```
<distributionManagement>
    <repository>
        <id>maven-releases</id>
        <name>maven-releases</name>
        <url>http://192.168.6.5:8081/repository/maven-releases/</url>
    </repository>
    <snapshotRepository>
        <id>maven-snapshots</id>
        <name>maven-snapshots</name>
        <url>http://192.168.6.5:8081/repository/maven-snapshots</url>
    </snapshotRepository>
</distributionManagement>
```
 - snapshot快照仓库用于保存开发过程中的不稳定版本，release正式仓库则是用来保存稳定的发行版本。

### 4. 全局异常处理设置

`TemplateCommonException`
```
@Data
public class TemplateCommonException extends RuntimeException{

    protected String code;

    public TemplateCommonException() {
        super();
    }

    public TemplateCommonException(String message) {
        super(message);
    }

    public TemplateCommonException(Throwable cause) {
        super(cause);
    }

    public TemplateCommonException(ErrorCodeEnum errorCodeEnum) {
        super(errorCodeEnum.getErrMsg());
        this.code = errorCodeEnum.getErrCode();
    }

    public TemplateCommonException(String message, Throwable cause) {
        super(message, cause);
    }

}
```

`ErroeCodeEnum`:
```
public enum ErrorCodeEnum {

    /*
     * 错误码结构: PCL+三位业务类别+三位错误码<br>
     * ===========================================<br> 100表示系统类别<br>
     * 101表示通用类别<br>
     */

    /*--------- 100开头表示系统级别错误 ---------*/
    SYS_ERROR("PCL100001", "系统内部错误"),

    /*--------- 101开头表示通用类别 ---------*/
    SUCCESS("PCL101001", "成功"),
    FAILURE("PCL101002", "失败"),
    PARAM_ERROR("PCL101003", "参数错误"),
    PARAM_EMPTY("PCL101004", "参数为空"),
    
    ...省略
}
```

`ExceptionHandler` 全局异常处理
```java

@RestControllerAdvice
@Slf4j
public class GlobalControllerExceptionHandler {

//    @Exceptio  nHandler(value = {MessageException.class})
    @ResponseStatus(HttpStatus.OK)
    public BaseResult suzukiCommonException(TemplateCommonException e) {
       log.error("获取到一个SuzukiCommonException异常:{}", e.getMessage());
        return BaseResult.create(e.getCode(), e.getMessage());
    }


    @ExceptionHandler(value = {RuntimeException.class})
    @ResponseStatus(HttpStatus.OK)
    public BaseResult runtimException(RuntimeException e) {
        log.error("获取到一个RuntimeException异常:{}", e);
        return BaseResult.create(ErrorCodeEnum.SYS_ERROR.getErrCode(), ErrorCodeEnum.SYS_ERROR.getErrMsg());
    }

    @ExceptionHandler(value = {Exception.class})
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public BaseResult noHandlerdException(Exception e) {
        log.error("获取到一个Exception异常:{}", e);
        return BaseResult.create(ErrorCodeEnum.SYS_ERROR.getErrCode(), ErrorCodeEnum.SYS_ERROR.getErrMsg());
    }

}
```

 

## 四、 api模块配置

> api模块用于提供微服务外部调用接口

### 1. 基础pom配置
基础的`api`模块配置十分简单，只需添加`common`模块依赖，并管理模块发布配置：

```
<dependencies>
    <dependency>
        <groupId>com.sunyard.hw</groupId>
        <artifactId>template-common</artifactId>
    </dependency>
</dependencies>


<distributionManagement>
    <repository>
        <id>maven-releases</id>
        <name>maven-releases</name>
        <url>http://192.168.6.5:8081/repository/maven-releases/</url>
    </repository>
    <snapshotRepository>
        <id>maven-snapshots</id>
        <name>maven-snapshots</name>
        <url>http://192.168.6.5:8081/repository/maven-snapshots</url>
    </snapshotRepository>
</distributionManagement>
```

### 2. 接口示例


`AnnounceSerivice`
```
public interface AnnounceSerivice {

    /**
     * 根据条件查询某个管理员发布的公告数
     * @param announceQueryDTO
     * @return
     */
    Integer getAnnounceCountByQueryDTO(AnnounceQueryDTO announceQueryDTO) throws TemplateCommonException;
}
```


## 五、 mybatisgen模块设置

> mybatis模块用于根据数据库表自动生成对应
dal

### 1. pom依赖添加

```
<dependencies>
    <dependency>
        <groupId>org.apache.commons</groupId>
        <artifactId>commons-lang3</artifactId>
    </dependency>
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
    </dependency>
    <dependency>
        <groupId>org.apache.velocity</groupId>
        <artifactId>velocity</artifactId>
        <version>1.7</version>
    </dependency>
</dependencies>
```

### 2. 模板代码添加， 代码自动生成：
![image_1c3af7r8a1pb1ph31uki1ua21cdp1p.png-28.9kB][6]

 -  `db.config`内设置数据库连接配置
 ![image_1c3afpboihk687o3fn14pjv1f2j.png-11.6kB][7]
 -  `generate_tables.config`内设置需要自动生成代码的数据库表，不需要的在前方添加`#`
 ![image_1c3afu7f2tj51a221fqr13uir7136.png-4.5kB][8]
 -  `MyBatisGenConst`类内设置项目名，`GroupId`以及代码生成目录
 ![image_1c3ag6j2o1501ven90l116si3l3j.png-25.9kB][9]
 
 -  运行`Main`类方法自动生成代码，运行后`dal`、`core`模块结构如下：

![image_1c3ah8co2b89mjr1ldb1s6bujg6d.png-30.7kB][10]

![image_1c3ah8thg1q5qlvjv3gtm01a5r6q.png-18.7kB][11]
 
## 六、 dal模块配置

### 1. pom配置
```
<dependencies>
    <dependency>
        <groupId>org.mybatis.spring.boot</groupId>
        <artifactId>mybatis-spring-boot-starter</artifactId>
    </dependency>
    
    <dependency>
        <groupId>com.sunyard.hw</groupId>
        <artifactId>template-api</artifactId>
    </dependency>

    <dependency>
        <groupId>com.sunyard.hw</groupId>
        <artifactId>template-common</artifactId>
    </dependency>
</dependencies>
```

### 2. 添加mybatis.config
![image_1c3aigsnlost1u4jonu11vl114177.png-6.4kB][12]
```
<?xml version="1.0" encoding="UTF-8"?>

<!DOCTYPE configuration PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <!-- 全局参数 -->
    <settings>
        <!-- 使全局的映射器启用或禁用缓存。 -->
        <setting name="cacheEnabled" value="true"/>
        <!-- 全局启用或禁用延迟加载。当禁用时，所有关联对象都会即时加载。 -->
        <setting name="lazyLoadingEnabled" value="true"/>
        <!-- 当启用时，有延迟加载属性的对象在被调用时将会完全加载任意属性。否则，每种属性将会按需要加载。 -->
        <setting name="aggressiveLazyLoading" value="true"/>
        <!-- 是否允许单条sql 返回多个数据集  (取决于驱动的兼容性) default:true -->
        <setting name="multipleResultSetsEnabled" value="true"/>
        <!-- 是否可以使用列的别名 (取决于驱动的兼容性) default:true -->
        <setting name="useColumnLabel" value="true"/>
        <!-- 允许JDBC 生成主键。需要驱动器支持。如果设为了true，这个设置将强制使用被生成的主键，有一些驱动器不兼容不过仍然可以执行。  default:false  -->
        <setting name="useGeneratedKeys" value="true"/>
        <!-- 指定 MyBatis 如何自动映射 数据基表的列 NONE：不隐射　PARTIAL:部分  FULL:全部  -->
        <setting name="autoMappingBehavior" value="PARTIAL"/>
        <!-- 这是默认的执行类型  （SIMPLE: 简单； REUSE: 执行器可能重复使用prepared statements语句；BATCH: 执行器可以重复执行语句和批量更新）  -->
        <setting name="defaultExecutorType" value="SIMPLE"/>
        <!-- 使用驼峰命名法转换字段。 -->
        <setting name="mapUnderscoreToCamelCase" value="true"/>
        <!-- 设置本地缓存范围 session:就会有数据的共享  statement:语句范围 (这样就不会有数据的共享 ) defalut:session -->
        <setting name="localCacheScope" value="SESSION"/>
        <!-- 设置但JDBC类型为空时,某些驱动程序 要指定值,default:OTHER，插入空值时不需要指定类型 -->
        <setting name="jdbcTypeForNull" value="NULL"/>
        <!--显示SQL-->
        <setting name="logPrefix" value="dao."/>
    </settings>
    <!-- mybatis-config.xml -->
    <typeHandlers>
        <package name="com.sunyard.hw.announce.dal.config"/>
    </typeHandlers>

</configuration>
```

## 七、core模块设置
> core模块编写主要service层业务代码

添加对`api`、`common`、`dal`模块的依赖
```
<dependencies>
    <dependency>
        <groupId>com.sunyard.hw</groupId>
        <artifactId>template-dal</artifactId>
    </dependency>

    <dependency>
        <groupId>com.sunyard.hw</groupId>
        <artifactId>template-common</artifactId>
    </dependency>

    <dependency>
        <groupId>com.sunyard.hw</groupId>
        <artifactId>template-api</artifactId>
    </dependency>
</dependencies>
```


## 八、 web 模块配置

### 1. pom依赖配置
```
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
    </dependency>
    <dependency>
        <groupId>io.springfox</groupId>
        <artifactId>springfox-swagger2</artifactId>
    </dependency>
    <dependency>
        <groupId>io.springfox</groupId>
        <artifactId>springfox-swagger-ui</artifactId>
    </dependency>

    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
    </dependency>
    <dependency>
        <groupId>org.mybatis.spring.boot</groupId>
        <artifactId>mybatis-spring-boot-starter</artifactId>
    </dependency>
    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>druid</artifactId>
    </dependency>

    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>dubbo</artifactId>
    </dependency>
    <dependency>
        <groupId>org.apache.zookeeper</groupId>
        <artifactId>zookeeper</artifactId>
    </dependency>
    <dependency>
        <groupId>com.github.sgroschupf</groupId>
        <artifactId>zkclient</artifactId>
    </dependency>

    <dependency>
        <groupId>com.sunyard.hw</groupId>
        <artifactId>announce-core</artifactId>
    </dependency>

    <!--springboot LocalDateTime时间格式化 Jar包-->
    <dependency>
        <groupId>com.fasterxml.jackson.datatype</groupId>
        <artifactId>jackson-datatype-jsr310</artifactId>
        <version>2.9.0</version>
    </dependency>

</dependencies>
```

### 2. 创建启动类

`TemplateWebApplication`

```
//扫描包，将各类bean装配入spring上下文中
@SpringBootApplication(scanBasePackages = "com.sunyard")
//加载dubbo配置文件
@ImportResource({"classpath*:/dubbo/dubbo-all.xml"})
public class TemplateWebApplication {
    public static void main(String[] args) {
        SpringApplication.run(TemplateWebApplication.class, args);
    }
}
```


### 3. 配置application.yml

`application.yml`
```
spring:
  application:
    name: ANNOUNCE-SERVICE
  profiles:
    active: local     # 本地开发环境
#    active: dev       # 开发环境
#    active: test      # 测试环境
#    active: pre       # 预发环境
#    active: online    # 生产环境
```

`application-local.yml`

```
server:
  port: 1024
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/hewei?autoReconnect=true
    driver-class-name: com.mysql.jdbc.Driver
    username: root
    password: hewei0928
    type: com.alibaba.druid.pool.DruidDataSource
    initialSize: 5
    minIdle: 5
    maxActive: 20
    maxWait: 60000
    timeBetweenEvictionRunsMillis: 60000
    minEvictableIdleTimeMillis: 300000
    validationQuery: SELECT 1 FROM DUAL
    testWhileIdle: true
    testOnBorrow: true
    testOnReturn: false
    poolPreparedStatements: true
    maxPoolPreparedStatementPerConnectionSize: 20
    filters: stat,wall,log4j
    connectionProperties: druid.stat.mergeSql=true;druid.stat.slowSqlMillis=5000
    cachePrepStmts: true  # 开启二级缓存
    # Druid Login Params
    loginUserName: admin
    loginPassword: Hello@1234

  jackson:
    serialization:
      write_dates_as_timestamps: false

mybatis:
  mapper-locations: classpath*:mapper/**/*.xml
  type-aliases-package: com.sunyard.hw.template.dal.model
  config-location: classpath:mybatis/mybatis-config.xml
```

### 3. 配置SwaggerUI

`Swagger2`

```
@Configuration
@EnableSwagger2
public class Swagger2 {
    @Bean
    public Docket createRestApi() {
        return new Docket(DocumentationType.SWAGGER_2)
                .apiInfo(apiInfo())
                .select()
                .apis(RequestHandlerSelectors.basePackage("com.sunyard.hw.announce.controller"))
                .paths(PathSelectors.any())
                .build();
    }

    private ApiInfo apiInfo() {
        return new ApiInfoBuilder()
                .title("消息中心RESTful APIs")
                .version("1.0")
                .build();
    }

}
```

### 4. 添加dubbo配置

`dubbo-all`
```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://code.alibabatech.com/schema/dubbo
       http://code.alibabatech.com/schema/dubbo/dubbo.xsd">

    <!-- 提供方应用信息，用于计算依赖关系 -->
    <dubbo:application name="cls-announce"  />

    <!-- 使用zookeeper注册中心暴露服务地址 -即zookeeper的所在服务器ip地址和端口号 -->
    <dubbo:registry address="zookeeper://localhost:2181" />

    <!-- 用dubbo协议在20880端口暴露服务 -->
    <dubbo:protocol name="dubbo" port="20880" />

    <import resource="dubbo-provider.xml"></import>

</beans>
```

`dubbo-provider`
```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
   xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
   xsi:schemaLocation="http://www.springframework.org/schema/beans
   http://www.springframework.org/schema/beans/spring-beans.xsd
   http://code.alibabatech.com/schema/dubbo
   http://code.alibabatech.com/schema/dubbo/dubbo.xsd">

<dubbo:service interface="com.sunyard.hw.template.api.service.AnnounceSerivice" ref="announceServiceImpl" group="com.sunyard.hw" version="1.0"/>

</beans>
```


九、 总结

&ensp;&ensp;OK， 到此为止项目的大致框架已经搭建完成，之后若遇到什么再分情况去解决吧。







 

 


    


  [1]: http://static.zybuluo.com/hewei0928/bxhjmh6477rxiyrpzoo8rjv6/image_1c32p45h31d48ceeckt1ep91tfn9.png
  [2]: http://static.zybuluo.com/hewei0928/iuyyjn3ma0h0hrq2vdd230vc/image_1c32q81bqdl91d4m1sfapl8jlu13.png
  [3]: http://static.zybuluo.com/hewei0928/fayep0f8m57re4f87uug8n3d/image_1c32qdjhd17m5133n46gg051p651g.png
  [4]: http://static.zybuluo.com/hewei0928/gmbt2ylpjbn3jb06zdbewa57/image_1c39qksi71iqs17ti1i6r18jn14349.png
  [5]: http://static.zybuluo.com/hewei0928/0fpqmwplygzqwy7i9csk2n7u/image_1c3a4vs6jibkuvna6511611n149.png
  [6]: http://static.zybuluo.com/hewei0928/5xhfxvi9fzbf4ruikry5282u/image_1c3af7r8a1pb1ph31uki1ua21cdp1p.png
  [7]: http://static.zybuluo.com/hewei0928/9dgqf4sjc9sq7iok5iofjk6x/image_1c3afpboihk687o3fn14pjv1f2j.png
  [8]: http://static.zybuluo.com/hewei0928/x1i35nrvcgwyk474nuzpml0z/image_1c3afu7f2tj51a221fqr13uir7136.png
  [9]: http://static.zybuluo.com/hewei0928/c4z9luttz5xbp0t5s16iiuvj/image_1c3ag6j2o1501ven90l116si3l3j.png
  [10]: http://static.zybuluo.com/hewei0928/u75764ch7v5usochqp4tqi7a/image_1c3ah8co2b89mjr1ldb1s6bujg6d.png
  [11]: http://static.zybuluo.com/hewei0928/v2cxzpfcm7k8wou5apqiiup9/image_1c3ah8thg1q5qlvjv3gtm01a5r6q.png
  [12]: http://static.zybuluo.com/hewei0928/0n7x5p6fks63g1kmud6h9nb7/image_1c3aigsnlost1u4jonu11vl114177.png