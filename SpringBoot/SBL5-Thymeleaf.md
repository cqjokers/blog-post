---
title: Spring Boot (五)Thymeleaf模板的整合
tags:
  - spring boot
  - thymeleaf
categories:
  - SpringBoot
abbrlink: 27afc888
date: 2018-08-13 14:58:18
---
![](https://i.loli.net/2018/08/13/5b712bc101a4c.jpg)
# 前言
在之前的开发中用到最多的就是JSP，Spring MVC里面也可以很方便的将JSP与一个View关联起来，使用还是非常方便的。但是把一个传统的Spring MVC项目转为一个Spring Boot项目后，却发现JSP和view关联有些麻烦，因为官方不推荐JSP在Spring Boot中使用。
# 使用
首先在 `pom.xml` 中添加对 `thymeleaf` 模板依赖
```JAVA
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
```
这里使用上一次新建的工程([参考地址](https://cqjokers.top/note/2018/08/10/ec5b638f.html))在些基础上加入`Thymeleaf`,在templaes目录中建立一个user.html模版，如下图所示：<!--more-->![](https://i.loli.net/2018/08/13/5b711c9038720.png)为什么要在templaes目录中新建模版后面会进行说明，新建模版后在顶部加入命名空间`<html lang="en" xmlns:th="http://www.thymeleaf.org">`，将静态转化为动态的视图，需要进行动态处理的元素使用“th:”前缀。`user.html`模版内容如下：
```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
<h2>用户信息</h2>
<table>
    <tr>
        <th>ID</th>
        <th>账号</th>
        <th>昵称</th>
    </tr>
    <tr th:each="user : ${userList}">
        <td th:text="${user.id}">0</td>
        <td th:text="${user.username}">王五</td>
        <td th:text="${user.nickname}">1990-06-01</td>
    </tr>
</table>
</body>
</html>
```
在原来的基础上对`UserController`进行修改，内容如下：
```JAVA
@Controller
public class UserController {

    @Autowired
    private UserService userService;

    @GetMapping("/user/{id}")
    public ModelAndView getUser(@PathVariable("id") int id) {
        ModelAndView view = new ModelAndView();
        UserInfo userInfo = userService.getUserById(id);
        view.addObject("user",userInfo);
        return  view;
    }

    @GetMapping("users")
    public ModelAndView getUsers(){
        ModelAndView view = new ModelAndView("user");
        List<UserInfo> userInfoList = userService.getAllUser();
        view.addObject("userList",userInfoList);
        return view;
    }
}
```
在浏览器中通过`localhost:9090/users`访问，如果出现以下页面则代码成功：![](https://i.loli.net/2018/08/13/5b7120154463e.png)这里只提供了users的模版，未提供访问单个user的模版。假如现在我在没有重启服务器的情况下对模版进行了修改，刷新浏览器会发现没有任何改变，要想使模版修改后立即生效可以在配置中添加`spring.thymeleaf.cache=false`这样就能够马上生效，还有一种方式就是依赖` spring-boot-devtools`热部署模块来达到同样的效果。
# 相关配置
在SpringBoot中默认情况下为我们做了如下的默认配置工作，配置如下图所示：![](https://i.loli.net/2018/08/13/5b712213d2f4c.png)通过上面的图就可以说明为什么模版必须放在templates中了，当然也可以放到其它位置，只需要对配置进行修改即可。Spring Boot集成Thymeleaf是通过org.springframework.boot.autoconfigure.thymeleaf包对Thymeleaf进行了自动配置，如下图所示：![](https://i.loli.net/2018/08/13/5b7122dd0fb63.png)配置里的所有以spring.thymeleaf为前缀的都是来自`ThymeleafProperties`这个类。通过查看`ThymeleafProperties`主要的源代码，可以看出如何设置属性以及默认配置：
```JAVA
private static final Charset DEFAULT_ENCODING;
public static final String DEFAULT_PREFIX = "classpath:/templates/";
public static final String DEFAULT_SUFFIX = ".html";
private boolean checkTemplate = true;
private boolean checkTemplateLocation = true;
private String prefix = "classpath:/templates/";
private String suffix = ".html";
private String mode = "HTML";
private Charset encoding;
private boolean cache;
private Integer templateResolverOrder;
private String[] viewNames;
private String[] excludedViewNames;
private boolean enableSpringElCompiler;
private boolean enabled;
private final ThymeleafProperties.Servlet servlet;
private final ThymeleafProperties.Reactive reactive;

public ThymeleafProperties() {
	this.encoding = DEFAULT_ENCODING;
	this.cache = true;
	this.enabled = true;
	this.servlet = new ThymeleafProperties.Servlet();
	this.reactive = new ThymeleafProperties.Reactive();
}
```
集成时所需的Bean都是通过`ThymeleafAutoConfiguration`类自动配置的，由于源代码过长这里就不再贴出。
