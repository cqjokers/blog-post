---
title: SpringBoot学习|(三)静态资源处理
tags:
  - spring boot
categories:
  - SpringBoot
abbrlink: e10179ef
date: 2018-08-10 00:00:00
---
![](https://i.loli.net/2018/08/10/5b6d18f56c5e9.jpg)
# 前言
在平时开发中难免会对静态资源的访问，如js、css、图片等其它资源，在SpringBoot中对静态资源提供了很好的支持，基本上可以直接通过配置就能满足平时开发中的需求，当然对于特殊需求也可以自定义资源映射来满足
# 默认静态资源映射
我们通过启动的时候控制台输出可以发现Spring Boot 默认将 /**映射到所有资源目录，如下图所示：![](https://i.loli.net/2018/08/10/5b6cf32ace9a4.png)
那么默认的资源目录有哪些了，我们可以通过在`application.properties`配置文件中尝试输入static然后IDEA工具会自动显示出与之相关的配置，如下图所示：![](https://i.loli.net/2018/08/10/5b6cf44a625a9.png)这里我们选择第一个，然后按住ctrl点击它则会进入`ResourceProperties`类，在类的开始处就定义了一个字符串数组来存放默认资源路径，如下图所示：![](https://i.loli.net/2018/08/10/5b6cf55d88d61.png)SpringBoot默认的资源路径有以下几个：
> 1、classpath:/META-INF/resources/
2、classpath:/resources/
3、classpath:/static/
4、classpath:/public/

<!--more-->
在访问资源时会按照这个顺序进行搜索直到找到为止。在第一张图片中发现还有一个/webjars/**,那么这个映射的路径又是什么了，首先先找到`spring-boot-autoconfigure`包中的spring.factories如图所示：![](https://i.loli.net/2018/08/10/5b6cfa701e8a9.png)然后在里面找到`WebMvcAutoConfiguration`按住ctrl点击进入该类，搜索webjars则会找到它的映射目录，如下图所示：![](https://i.loli.net/2018/08/10/5b6cfadd49fde.png)通过上图可知/webjars/**映射的目录是`classpath:/META-INF/resources/webjars/`，访问webjars下的资源都会映射到该目录里去。
# 自定义静态资源映射
有时候开发中可能需要自定义静态资源访问路径，那么这里有两种方式来实现：
1、修改配置文件中相关配置
2、继承`WebMvcConfigurationSupport`来实现
第一种方式：修改application.properties
比如说现在想让`/assets/**`映射到默认资源路径上则只需要在配置中加上`spring.mvc.static-path-pattern=/assets/**`即可，配置了该项后默认的就不再生效，比如访问/resources/1.jpg这样的将不再会访问到。如果需要修改默认资源路径，则在配置中加上`spring.resources.static-locations`并将值设置为自己的资源路径，这里可以设置多个以逗号分隔。
第二种方式：继承`WebMvcConfigurationSupport`
可以写一个类来继承`WebMvcConfigurationSupport`并得它的`addResourceHandlers`方法来达到自定义静态资源映射，如下图所示：![](https://i.loli.net/2018/08/10/5b6d035fd9dec.png)在该方法中我添加了多个资源映射，该方式与配置方式的好处在于可以添加多个，而配置方式只能设置一个。现在暂且不看`configureViewResolvers`方法可以运行起来试试看，结果应该不是预期的那样，会得到一个错误页面，并在控制台中有错误信息。如下图所示：![](https://i.loli.net/2018/08/10/5b6d04260dd12.png)根据错误信息可知是没有找到对应的视图，但是我在application.proerities中进行了视图的相关配置。要想知道错误的原因则需要从入口类中的`EnableAutoConfiguration`注解中的`EnableAutoConfiguration`进行分析，在这里大概说下是怎么回事，在启动的时候会根`EnableAutoConfiguration`注解导入的`AutoConfigurationImportSelector`类中的`getCandidateConfigurations`方法去导入`spring-boot-autoconfigure`jar下面的配置文件META-INF/spring.factories，在这个配置文件中会找到一个WebMvcAutoConfiguration，如下图所示：![](https://i.loli.net/2018/08/10/5b6d13d1e0146.png)进入该类会发现一个`ConditionalOnMissingBean`注解，它的意思是如果不存在它修饰的类的bean,则再创建这个bean。源码如下图所示：![](https://i.loli.net/2018/08/10/5b6d14ca35e9c.png)由此可得出结论：
如果有配置文件继承了DelegatingWebMvcConfiguration，
或者WebMvcConfigurationSupport，或者配置文件有@EnableWebMvc，那么 @EnableAutoConfiguration 中的WebMvcAutoConfiguration 将不会被自动配置，而是使用WebMvcConfigurationSupport的配置。所以到这里就基本上明白了上面访问的时候报错的原因了，现在只需要在自定义类中重写`configureViewResolvers`方法，加上相关设置即可。
# 总结
在开发过程中如果使用了类似`@EnableWebMvc`这样的注解，则`@EnableAutoConfiguration `中的将不再生效，这就需要在代码中进行相关设置。