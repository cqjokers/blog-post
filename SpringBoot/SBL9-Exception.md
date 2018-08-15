---
title: Spring Boot (九)全局异常处理
tags:
  - spring boot
  - exception
categories:
  - SpringBoot
abbrlink: f83b82ed
date: 2018-08-15 17:44:37
---
![](https://i.loli.net/2018/08/15/5b73f5e33c607.jpg)
# 前言
在平时开发中有时候会出现无法预料的异常，如果对每个异常都单独自己去处理就显得非常的麻烦，所以我们可以创建一个全局异常处理类来统一处理异常。
# 自定义异常
在开发过程中，除了系统自身的异常外，我们也可以根据不同业务场景来定义一些异常类，创建一个`CustomException`异常类，代码如下所示：
```JAVA
public class CustomExecption extends RuntimeException{

    private String code;
    private String message;

    public CustomExecption(String code,String message) {
        this.code = code;
        this.message = message;
    }

    public String getCode() {
        return code;
    }

    public void setCode(String code) {
        this.code = code;
    }

    @Override
    public String getMessage() {
        return message;
    }

    public void setMessage(String message) {
        this.message = message;
    }
}
```
# 异常处理
在对异常进行处理时需要用到以下注解：
@RestControllerAdvice 相当于 @ControllerAdvice + @ResponseBody
@ControllerAdvice 捕获 Controller 层抛出的异常，如果添加 @ResponseBody 返回信息则为JSON 格式。<!--more-->
@ExceptionHandler 统一处理一种类的异常，减少代码重复率，降低复杂度。
创建一个 GlobalExceptionHandler 类，并添加上 @ControllerAdvice 注解就可以定义出异常通知类了，然后在定义的方法中添加上 @ExceptionHandler 即可实现异常的捕捉，代码如下面所示：
```JAVA
@ControllerAdvice
public class GlobalExecptionHandler {

    /**
     * 返回JSON格式的错误信息
     * @param e
     * @return
     */
    @ExceptionHandler(value = Exception.class)
    @ResponseBody
    public Map runtimeExceptionHandler(Exception e) {
        Map map = new HashMap();
        map.put("code",10000);
        map.put("message",e.getMessage());
        return map;
    }

    /**
     * 返回错误信息到页面上
     * @param e
     * @return
     */
    @ExceptionHandler(value = CustomExecption.class)
    public ModelAndView CustomExceptionHandler(CustomExecption e) {
        ModelAndView view = new ModelAndView("error");
        view.addObject("code",e.getCode());
        view.addObject("message",e.getMessage());
        return view;
    }
}
```
处理类中最后一个方法是直接将错误信息返回到error页面上，如果要返回JSON字符串只需要在方法上加上`ResponseBody `注解
# error页面
创建一个error模版，代码如下所示：
```JAVA
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>错误页面</title>
</head>
<body>
    <h1 th:text="${code}"></h1>
    <h1 th:text="${message}"></h1>
</body>
</html>
```
# 控制器
在之前项目的基础上对`UserController`类进行修改，如下面所示：
```JAVA
@Controller
public class UserController {

    @RequestMapping("/user")
    @ResponseBody
    public Object getUser() {
        try {
            int a = 1 / 0;
        }catch (Exception e){
            throw new CustomExecption("10000","除数不能为0");
        }
        return null;
    }

    @RequestMapping("/users")
    @ResponseBody
    public Object getUsers() throws Exception{
        int a = 1 / 0;
        return null;
    }
}
```
为了演示错误，这里直接用最除数为0的这个错误。
# 测试
为了演示错误页面这里不再使用测试类来进行测试而换为浏览器，启动项目然后在浏览器中输入`http://localhost:9090/users`得到如下结果：
```JAVA
{"code":10000,"message":"/ by zero"}
```
访问`http://localhost:9090/user`得到以下结果：![](https://i.loli.net/2018/08/15/5b73f3d3d26a6.png)

本文示例地址：https://github.com/cqjokers/Spring-Boot-Learning