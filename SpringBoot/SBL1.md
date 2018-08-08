---
title: Spring Boot (一)工程构建
date: 2018-08-08
tags:
- spring boot
categories:
- SpringBoot
---
![](https://i.loli.net/2018/08/08/5b6abd5c099db.png?300x600)
# 前言
由于之前一直都是用的SSM，未接触过Spring Boot 然而现在很多公司都在用Spring Boot，因为Spring Boot 设计目的是用来简化新Spring应用的初始搭建以及开发过程，使用了特定的方式来进行配置，从而使开发人员不再需要定义样板化的配置。所以有了它开发人员不用再去注重配置文件的繁琐而是把精力放到业务逻辑上面，由于是初次接触所以记录一下学习过程。
# 项目构建
在这里我使用的是IntelliJ IDEA来构建项目，打开工具界面如下图：![](https://i.loli.net/2018/08/08/5b6a943d736f5.png)点击 Create New Project来创建一个SpringBoot应用程序，点击后效果图如下所示：<!--more-->![](https://i.loli.net/2018/08/08/5b6a94a066098.png)在左侧我们选择Spring Initializr然后点击Next进入下一个界面，填 入相关信息，如下图所示：![](https://i.loli.net/2018/08/08/5b6a95da93a13.png)填入信息后点击Next按钮进入下一个界面，这个界面上会显示很多项目所需要的依赖，在这里我们勾选上web即可。如下图所示：![](https://i.loli.net/2018/08/08/5b6a96b2d1d9b.png)最后点击Next即可，这样项目就创建成功了
# 目录结构
我们来看一下IDEA给我们生成的目录结构，如下图所示：![](https://i.loli.net/2018/08/08/5b6a97a81dbd2.png) 
> 1、/src/main/java/  存放项目所有源代码目录
2、/src//main/resources/  存放项目所有资源文件以及配置文件目录,其中static存入静态资源如css、js、img等，templates是存放模版资源
3、/src/test/  存放测试代码目录
4、pom文件为基本的依赖管理文件

`SblApplication`为程序的入口文件，这个类中使用了一个`SpringBootApplication`注解,该注解在SpringBoot项目中有且只有这么一个注解存在，它的作用就是声明当前类为SpringBoot的入口类
# pom.xml
pom.xml是整个maven的核心配置文件，里面有对项目的描述和项目所需要的依赖。我们在创建好项目后我们最好先设置一下该项目的Maven设置(IDEA对每个项目的maven设置与Eclipse有差别，Eclipse设置一次就行，而IDEA每次新建项目后都需要设置)![](https://i.loli.net/2018/08/08/5b6aa068af11f.png)将Maven目录指向我们安装的Maven位置，选择user setting配置文件后在Eclipse中会自动将本地厂库的路径设置为setting文件中配置的路径，而在IDEA中则不行，需要自己手动设置。这里最好不要用默认配置，默认配置是在C盘中，随着项目的增多依赖的库较多的时候本地厂库占用的空间也会较多，所以最后配置在其它盘中。就算后面重装了系统这些文件依然存在。
# 初次运行
进入`SblApplication`中右键运行该项目，如何在控制台输出以下信息则代表项目已经运行成功，如下图所示：![](https://i.loli.net/2018/08/08/5b6aac7b65e25.png)这里可能会有些疑问，我们没有配置WEB容器为什么能够运行起来，观看控制台的输出可以看到Starting Servlet Engine:Apache Tomcat/8.5.32说明其实也是通过Tomcat来运行的，只是SpringBoot帮我们省去了这一步的操作，因为SpringBoot内置了Tomcat。现在我们通过浏览器访问看看会得到什么样的结果，结果如下图所示：![](https://i.loli.net/2018/08/08/5b6aae48cd6c0.png)这里得到的只是一个错误页面
# 惯例HelloWorld
在sbl包下面创建一个controller为名字的包用于存放控制器，包创建好后再创建一个`IndexController`的类，在类上我们使用`@RestController`注解来声明该类是一个访问控制器，在该类中创建一个index方法，并配置一个`@RequestMapping`注解来声明它，`@RequestMapping`中的默认参数写成"/index",方法体中直接返回"HelloWorld"字符串，这样我们就可以通过http://localhost:8080/index访问到该方法，代码如下图所示：![](https://i.loli.net/2018/08/08/5b6ab4c318c1d.png)通过浏览器访问http://localhost:8080/index会看到页面上输出Hello World就说明已经成功了。
# 总结
本文主要通过Spring Boot简单特性来完成了第一个"Hello Word"web应用程序的搭建，从中可以看出Spring Boot在搭建一个项目整合组件方面很成熟，这就有效的提高团队开发效率大减少了开发人员上手周期。由于本人未接触过Spring Boot文章中若有不对的地方还请指出，写文章的目的在于对自己学习的一个记录以及对自己的一个提升。