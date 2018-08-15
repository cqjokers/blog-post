---
title: Spring Boot (八)拦截器的添加
tags:
  - spring boot
  - interceptor
categories:
  - SpringBoot
abbrlink: 513b7b6a
date: 2018-08-15 15:03:10
---
![](https://i.loli.net/2018/08/15/5b73d301e0daa.jpg)
# 前言
在SSM开发中对拦截器应该都非常熟悉，经常使用拦截器来实现如登录权限验证、日志记录、session验证等，在SpringBoot中使用拦截器与SSM中区别不大，只是配置的方式不一样。
# 编码
工程还是用之前创建好的工程，在工程上新建一个interceptor包，如下所示：![](https://i.loli.net/2018/08/15/5b73c90492fab.png)在包中新建一个`LoginInterceptor`,代码如下所示：
```JAVA
public class LoginInterceptor implements HandlerInterceptor {

    private Logger logger = LoggerFactory.getLogger(LoginInterceptor.class);

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        logger.info(">>>>>>>>>>>>>>preHandle<<<<<<<<<<<<<<<<");
        return true;
    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
        logger.info(">>>>>>>>>>>>>>postHandle<<<<<<<<<<<<<<<<");
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        logger.info(">>>>>>>>>>>>>>afterCompletion<<<<<<<<<<<<<<<<");
    }
}
```
代码中每个方法都用日志进行输出，正式开发中需要根据业务进行相关代码编写。
在工程中创建configure包，并创建一个`InterceptorConfigure`类，其代码如下所示：
<!--more-->
```JAVA
@Configuration
public class InterceptorConfigure extends WebMvcConfigurationSupport {
    @Override
    protected void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new LoginInterceptor()).addPathPatterns("/**")
                .excludePathPatterns("/users");
        super.addInterceptors(registry);
    }
}
```
代码中对所有的路径都进行拦截除了''/users''
# 测试
为了便于区分是否被拦截，所以将之前写的测试类中的查询所有用户信息单独用一个方法，代码如下所示：
```JAVA
@RunWith(SpringRunner.class)
@SpringBootTest(classes = SblApplication.class, webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public class SblApplicationTests {


    private Logger logger = LoggerFactory.getLogger(SblApplication.class);

    @Autowired
    TestRestTemplate template;

    @Test
    public void getUser() {
        //查询单个
        ResponseEntity<TUser> responseEntity = template.getForEntity("http://localhost:9090/user/{id}", TUser.class, 2);
        logger.info(responseEntity.getBody().toString());

    }

    @Test
    public void getUsers() {

        //查询所有
        ResponseEntity<List<TUser>> responseEntity1 = template.exchange("http://localhost:9090/users", HttpMethod.GET, null, new ParameterizedTypeReference<List<TUser>>() {
        });
        logger.info(responseEntity1.getBody().toString());
    }
}
```
首先运行“getUser”方法，查看控制台的输出，如是出现了拦截器中打印的日志则代码该方法所访问的路径被拦截到了，如下所示：![](https://i.loli.net/2018/08/15/5b73cf026d4f2.png)
现在把控制台清空，然后运行"getUsers"方法，如果控制台中未出现拦截器中的日志输出则该方法访问的路径未被拦截，也验证了我们上面代码的正确性。

本文示例代码地址：https://github.com/cqjokers/Spring-Boot-Learning