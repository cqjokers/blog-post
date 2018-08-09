---
title: Spring Boot (二)JSP的支持
tags:
  - spring boot
categories:
  - SpringBoot
abbrlink: c7818a32
date: 2018-08-09 00:00:00
---
![](https://i.loli.net/2018/08/09/5b6bf25825e3d.jpg)
#前言
由于SpringBoot推荐的视图是Thymeleaf,而在之前我们的开发中使用的是JSP，初次接触当然希望还是能用JSP来作试图，后面再去尝试Thymeleaf,本文内容说的就是如何让SpringBoot web项目支持JSP。
# 视图配置
项目的创建这里就不再展示，可以参考[工程构建这一篇](https://cqjokers.top/note/2018/08/08/170f521.html)，在这里我们参照Eclipse的目录风格来，首先是在main包中创建webapp目录，再在webapp中创建WEB-INF文件夹，最后创建一个存放jsp文件的文件夹，这里我使用views这个名字，最终结构如下图所示：![](https://i.loli.net/2018/08/09/5b6bad376f1d1.png)使用过springmvc我们应该知道，在配置中我们需要配置视图解析器的前缀目录与后缀，在这里我们仍然需要进行配置。配置如下图所示：![](https://i.loli.net/2018/08/09/5b6bdf10628d5.png)这里我使用的端口是9090配置文件用的是yml格式，配置好后现在虽然能运行起来但是访问页面只会返回一个错误页面。
# JSP创建
在view文件夹中新建一个`index.jsp`文件，如下图所示<!--more-->：![](https://i.loli.net/2018/08/09/5b6be4a98ecec.png)
#控制器创建
在controller包中创建一个`IndexController`类，并在首部加上`@Controller`注解，这里之所以没有使用`@RestController`注解是因为我们需要在这个类中的index方法跳转到一个页面，@RestController注解相当于@ResponseBody ＋ @Controller合在一起的作用。在index方法上添加`@RequestMapping(value = "/index")`，浏览器通过地址加上该路径则会访问该方法，代码如下图所示：![](https://i.loli.net/2018/08/09/5b6be6180f890.png)
#pom.xml
为了能够访问页面我们需要添加jsp、servlet的maven支持，开发中还会用到jstl所以在这里也一并添加进去
```java
<dependency>
	<groupId>org.apache.tomcat.embed</groupId>
	<artifactId>tomcat-embed-jasper</artifactId>
	<scope>provided</scope>
</dependency>
<dependency>
	<groupId>javax.servlet</groupId>
	<artifactId>javax.servlet-api</artifactId>
	<version>4.0.1</version>
	<scope>provided</scope>
</dependency>
<dependency>
	<groupId>javax.servlet</groupId>
	<artifactId>jstl</artifactId>
</dependency>
```
#启动
到此基本上算是完成了，现在启动项目看下是否能够达到预期的效果，结果如下图所示：![](https://i.loli.net/2018/08/09/5b6be796c3d36.png)如果显示下面页面则代表已经成功了。
#错误
虽然说通过IDEA启动能访问，但是这里还有一个问题就是，当你打包成JAR包后运行，则会发现返回的是一个错误页面，造成该错误的原因是，打包时并没有将web资源一起打进去，可以把JAR包解压查看里面的文件。所以打包时需要将jsp文件加入进去，在pom.xml中添加以下配置：
```JAVA
<resources>
	<resource>
		<directory>src/main/webapp</directory>
		<targetPath>META-INF/resources</targetPath>
		<includes>
			<include>**/**</include>
		</includes>
	</resource>
	<resource>
		<directory>src/main/resources</directory>
		<filtering>false</filtering>
		<includes>
			<include>**/**</include>
		</includes>
	</resource>
</resources>
```
最关键的一个步骤就是在spring-boot-maven-plugin中一定要加上 <version>1.4.2.RELEASE</version>否则打包出来一样无法访问jsp文件，可以进行尝试换成高于1.4.2版本看下会是怎样的结果，不出意外应该也是返回错误页面