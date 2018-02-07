---
title: Lombok初识
date: 2017-08-10 10:33:28
tags:
    - Java
    - Lombok
---

## 前言
>&nbsp;&nbsp;`Lombok`是最近看老大代码是发现的一个非常好用的小工具，刚看到就有一种相见恨晚的感觉，上手之后更觉不错，在此记录一下。
&nbsp;&nbsp;官网地址：[https://projectlombok.org/](https://projectlombok.org/)

<!--more-->

## Lombok简介
&ensp;&ensp;`Lombok`是一个可以通过简单的注解来简化一些必须有但显得十分臃肿的`Java`代码的工具。例如，在`java`项目编写的过程中，实体类总是有许多属性字段，而我们就必须为其添加对应的`get、set`方法，虽然有`idea`快捷键，但每次修改字段时任免不了这些操作， 还是十分不便。`Lombok`的作用就是为了省去这些手动创建代码的麻烦，它能够在编译源码的时候自动帮我们生成这些方法。

## Lombok使用

### 1.设置`Lombok`环境

 1. 下载`Lombok`插件
    &ensp;&ensp;`File -> setting -> Plugins -> Browse repositories -> 搜索Lombok -> 点击install`
    &ensp;&ensp;`Setting -> Compiler -> Annotation Processors -> Enable annotation processing勾选`
    &ensp;&ensp;然后重启idea即可。

 2. 添加`Lombok`的`maven`的`pom.xml`依赖
    ```java
    <dependency>  
      <groupId>org.projectlombok</groupId>  
      <artifactId>lombok</artifactId>  
      <version>1.16.10</version>  
    </dependency>  
    ```

### 2. `Lombok`使用示例
 1. 示例类 `Student`
    ```java
    package com.pojo;
    
    import lombok.Getter;
    import lombok.Setter;
    import lombok.ToString;
    
    /**
     * Created by Administrator
     * on 2017/8/10 11:34.
     */
    @Setter
    @Getter
    @EqualsAndHashCode
    @ToString
    public class Student {

        private String name;
        private String age;
        private String group;
    
    }

    ```
    
 2. 测试类`Start`
    ```java
    package com;

    import com.pojo.Student;
    
    /**
     * Created by Administrator
     * on 2017/8/10 11:09.
     */
    public class Start {
    
        public static void main(String[] args) {
            Student s = new Student();
            s.setAge("23");
            s.setName("hewei");
            Student s2 = new Student();
            s2.setAge("23");
            s2.setName("hewei");
            System.out.println(s.getAge());
            System.out.println(s.equals(s2));
            s2.setName("he");
            System.out.println(s.equals(s2));
            System.out.println(s);
        }
    }
    ```
    
 3. 输出结果
    ```java
    23
    true
    false
    Student(name=hewei, age=23, group=null)
    ```

&ensp;&ensp;如上结果可以看出,`Lombok`注释省去了`get`、`set`、`equals`、`hashCode`、`toString`等代码，大大简化了代码编写，省去了没必要的代码量。

## Lombok常用注解
>`Lombok`官网上的文档里面有所有的注解，这里不一一罗列，只说明其中几个比较常用的。

`@NonNull`:  控制参数不为空
`@NoArgsConstructor`:  自动生成无参构造函数
`@AllArgsConstructor`: 自动生成全参构造函数
`@EqualsAndHashCode`: 添加`equals()`，`hashcode()`方法
`@Data`: 自动为所有字段添加`@ToString`, `@EqualsAndHashCode`, `@Getter`方法，为非`final`字段添加`@Setter`,和`@RequiredArgsConstructor`