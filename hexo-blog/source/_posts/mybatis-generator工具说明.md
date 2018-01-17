---
title: mybatis-generator 工具说明
date: 2018-01-17 
tags: [mybatis]
---

参考地址：[mybatis-generator中文地址](http://mbg.cndocs.ml/)
[参考博客](http://blog.csdn.net/isea533/article/details/42102297)
[mybatis-generator代码生成工具地址](https://github.com/EagleGZG/mybatis-generator)

## 使用说明
1. 维护src/main/resource下的generatorConfigWuJian.xml文件
2. 打开src/main/java下的com.wuj.generator.GenarateMybatis文件，指定要加载的文件路径及文件名，右键->run as java application 就能生成pojo文件以及对应的mapper文件     
```java
File configFile = new File(GenarateMybatis.class.getResource("/generatorConfigWuJian.xml").getPath());
```
<!-- more -->

## generatorConfigWuJian.xml文件说明

```xml
<classPathEntry location="C:/Users/T450/.m2/repository/mysql/mysql-connector-java/5.1.22/mysql-connector-java-5.1.22.jar" />
```
* location:该location指定了mysql-connector-java-5.1.22.jar 包所在的位置

```xml
<context id="context1" targetRuntime="MyBatis3" defaultModelType="flat">
```
<context> 元素用于指定生成一组对象的环境。 子元素用于指定要连接到的数据库、 要生成对象的类型和要内省的表。 多个 <context> 元素可以在 <generatorConfiguration> 元素中列出来，这样可以在同一个MyBatis Generator (MBG)从不同的数据库或者使用不同的生成生成器参数生成对象。
* targetRuntime：如果你希望不生成和Example查询有关的内容，那么可以按照如下进行配置:
```xml
<context id="Mysql" targetRuntime="MyBatis3Simple" defaultModelType="flat">
```
* defaultModelType：这个属性用来设置生成对象类型的默认值。 对象类型定义了MBG如何生成实体类。 对某些对象类型，MBG会给每一个表生成一个单独的实体类。 对另外一些对象类型，MBG会根据表结构生成不同的类。这个属性有以下可选值：   
* conditional：这是默认值这个模型和hierarchical类似，除了如果那个单独的类将只包含一个字段，将不会生成一个单独的类。因此,如果一个表的主键只有一个字段,那么不会为该字段生成单独的实体类,会将该字段合并到基本实体类中。

* flat(**推荐**):该模型为每一张表只生成一个实体类。这个实体类包含表中的所有字段。
* hierarchical：如果表有主键,那么该模型会产生一个单独的主键实体类,如果表还有BLOB字段， 则会为表生成一个包含所有BLOB字段的单独的实体类,然后为所有其他的字段生成一个单独的实体类。 MBG会在所有生成的实体类之间维护一个继承关系（注：BLOB类 继承 其他字段类 继承 主键类）。

```xml
<javaModelGenerator targetPackage="com.wuj.entity"
    targetProject="src/main/java">
	<property name="enableSubPackages" value="true" />
	<property name="trimStrings" value="true" />
</javaModelGenerator>
```
* targetPackage: 生成实体类存放的包名，一般就是放在该包下。实际还会受到其他配置的影响
* targetProject: 指定目标项目路径，可以是绝对路径或相对路径（如 targetProject="src/main/java"）

```xml
<!-- 
	ANNOTATEDMAPPER:基于注解的Mapper接口，不会有对应的XML映射文件
	MIXEDMAPPER:XML和注解的混合形式，(上面这种情况中的)
	XMLMAPPER:所有的方法都在XML中，接口调用依赖XML文件
 --> 
<javaClientGenerator targetPackage="com.wuj.dao.mapper"
	targetProject="src/main/java" type="ANNOTATEDMAPPER">
	<property name="enableSubPackages" value="true" />	
</javaClientGenerator>
```
该元素可选，最多配置一个。如果不配置该元素，就不会生成Mapper接口。该元素有3个必选属性：
* type:该属性用于选择一个预定义的客户端代码（可以理解为Mapper接口）生成器，用户可以自定义实现，需要继承org.mybatis.generator.codegen.AbstractJavaClientGenerator类，必选有一个默认的构造方法。 该属性提供了以下预定的代码生成器，首先根据<context>的targetRuntime分成三类：
**MyBatis3**
> 1. ANNOTATEDMAPPER:基于注解的Mapper接口，不会有对应的XML映射文件
> 2. MIXEDMAPPER:XML和注解的混合形式，(上面这种情况中的)SqlProvider注解方法会被XML替代。
> 3. XMLMAPPER:所有的方法都在XML中，接口调用依赖XML文件。
**MyBatis3Simple**
> 1. ANNOTATEDMAPPER:基于注解的Mapper接口，不会有对应的XML映射文件。
> 2. XMLMAPPER:所有的方法都在XML中，接口调用依赖XML文件。

关于**jdbcConnection**注意点：
> * **&** 需要转义为 **&amp**;

