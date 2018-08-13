---
title: Spring Boot (六)Servlet、Filter、Listener配置
tags:
  - spring boot
  - servlet
  - filter
  - listener
categories:
  - SpringBoot
abbrlink: '38067409'
date: 2018-08-13 17:41:27
---
![](https://i.loli.net/2018/08/13/5b7151d0be8bb.jpg)
SpringBoot对Servlet、Filter、Listener的配置有两种方式：
1、注解方式
2、代码注册方式
# 注解方式
通过注解方式，需要在启动类上加入`@ServletComponentScan`注解，具体配置项如下：
```JAVA
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import({ServletComponentScanRegistrar.class})
public @interface ServletComponentScan {
    @AliasFor("basePackages")
    String[] value() default {};

    @AliasFor("value")
    String[] basePackages() default {};

    Class<?>[] basePackageClasses() default {};
}
```
## Servlet
新建一个IndexServlet，代码如下所示：
```JAVA
@WebServlet(name = "IndexServlet",urlPatterns = "/index")
public class IndexServlet extends HttpServlet {
    private Logger logger = LoggerFactory.getLogger(IndexServlet.class);
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        logger.info(">>>>>>>>>>>>>进入IndexServlet<<<<<<<<<<<<<");
        resp.setCharacterEncoding("UTF-8");
        PrintWriter pw = resp.getWriter();
        pw.print("这是由IndexServlet返回");
        pw.flush();
        pw.close();
    }
}
```
需要在类上加上`WebServlet`注解并对相关配置项进行设置
## Filter
新建一个LoginFilter,代码如下所示：
```JAVA
@WebFilter(urlPatterns = "/*",filterName = "loginFilter")
public class LoginFilter implements Filter {
    private Logger logger = LoggerFactory.getLogger(LoginFilter.class);
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        logger.info(">>>>>>>>>>>LoginFilter init...<<<<<<<<<<<<<");
    }

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        logger.info(">>>>>>>>>>>LoginFilter doFilter...<<<<<<<<<<<<<<");
        filterChain.doFilter(servletRequest,servletResponse);
    }

    @Override
    public void destroy() {

    }
}
```
需要在类上加上`WebFilter`注解并对相关配置项进行设置
## Listener
新建一个IndexListener，代码如下所示：
```JAVA
@WebListener
public class IndexListener implements ServletContextListener {
    private Logger logger = LoggerFactory.getLogger(IndexListener.class);
    @Override
    public void contextInitialized(ServletContextEvent servletContextEvent) {
        logger.info(">>>>>>>>>>>>>>>IndexListener contextInitialized<<<<<<<<<<<<<<<<<<<");
    }

    @Override
    public void contextDestroyed(ServletContextEvent servletContextEvent) {
        logger.info(">>>>>>>>>>>>>>>IndexListener contextDestroyed<<<<<<<<<<<<<<<<<<<");
    }
}
```
需要在类上加上`WebListener`注解
<!--more-->
# 代码注册方式
SpringBoot提供了三种Bean，FilterRegistrationBean、ServletRegistrationBean、ServletListenerRegistrationBean分别对应配置原生的Filter、Servlet、Listener。
## Servlet
```JAVA
@Configuration
public class ServletConfigure {

    @Bean
    public ServletRegistrationBean indexServlet() {
        ServletRegistrationBean bean = new ServletRegistrationBean();
        bean.setServlet(new IndexServlet());
        bean.addUrlMappings("/index");
        return  bean;
    }
}
```
## Filter
```JAVA
@Configuration
public class FilterConfigure {

    @Bean
    public FilterRegistrationBean loginFilter() {
        FilterRegistrationBean bean = new FilterRegistrationBean();
        bean.addUrlPatterns("/*");
        bean.setFilter(new LoginFilter());
        return bean;
    }

}
```
## Listener
```JAVA
@Configuration
public class ListenerConfigure {

    @Bean
    public ServletListenerRegistrationBean indexListener() {
        ServletListenerRegistrationBean bean = new ServletListenerRegistrationBean();
        bean.setListener(new IndexListener());
        return bean;
    }
}
```
正常的情况下通过代码方式得到的结果应该与注解方式一样。最后附上项目的结构图：
![](https://i.loli.net/2018/08/13/5b714d05e3ab5.png)
# 总结
通过运行结果可以知道，这三者处理顺序是listener->filter->servlet,通过注解方式其实最终都会转换成这三种bean的FilterRegistrationBean、ServletRegistrationBean、ServletListenerRegistrationBean。