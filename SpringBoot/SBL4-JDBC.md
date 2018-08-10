---
title: Spring Boot (四)JdbcTemplate的整合
tags:
  - spring boot
categories:
  - SpringBoot
abbrlink: ec5b638f
date: 2018-08-10 16:29:18
---
![](https://i.loli.net/2018/08/10/5b6d4cb1ac2b0.jpg)
# 导入依赖
在`pom.xml`中添加一些依赖来配置数据源，我们在pom.xml中加入以下依赖：
```JAVA
<!--mysql包-->
<dependency>
	<groupId>mysql</groupId>
	<artifactId>mysql-connector-java</artifactId>
</dependency>
<!--Spring JDBC 的依赖包-->
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-jdbc</artifactId>
	<version>2.0.4.RELEASE</version>
</dependency>
```
# 配置
```JAVA
server:
  port: 9090
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/SBL?autoReconnect=true&useUnicode=true&characterEncoding=utf-8
    username: root
    password: root
    driver-class-name: com.mysql.jdbc.Driver
#    type: com.zaxxer.hikari.HikariDataSource
```
上面的type是配置连接池的类型，如果没有配置的话默认是使用hikari的，可以通过控制来看，如下面所示：
<!--more-->
```JAVA
o.s.j.e.a.AnnotationMBeanExporter        : Bean with name 'dataSource' has been autodetected for JMX exposure
o.s.j.e.a.AnnotationMBeanExporter        : Located MBean 'dataSource': registering with JMX server as MBean [com.zaxxer.hikari:name=dataSource,type=HikariDataSource]
o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 9090 (http) with context path ''
```
也可以通过源代码来查看，主要代码位于`DataSourceAutoConfiguration`中的`PooledDataSourceConfiguration`方法处，如下面所示：
```JAVA
@Configuration
@Conditional({DataSourceAutoConfiguration.PooledDataSourceCondition.class})
@ConditionalOnMissingBean({DataSource.class, XADataSource.class})
@Import({Hikari.class, Tomcat.class, Dbcp2.class, Generic.class, DataSourceJmxConfiguration.class})
protected static class PooledDataSourceConfiguration {
	protected PooledDataSourceConfiguration() {
	}
}
```
可以看出SpringBoot默认支持Hikari、Tomcat、Dbcp2、Generic这五种数据源,所以可能通过`spring.datasource.type`来指定其它种类的连接池。假如这5种都不想用，换用阿里的druid数据源。首先是加入依赖
```JAVA
<dependency>
	<groupId>com.alibaba</groupId>
	<artifactId>druid</artifactId>
	<version>1.0.14</version>
</dependency>
```
然后修改配置
```JAVA
spring.datasource.type=com.alibaba.druid.pool.DruidDataSource
```
除了上面的配置方式外，还有另外一种方式就是通过代码来完成，不过首先也得添加依赖。代码如下面所示：
```JAVA
@Bean
public DataSource dataSource() {
	DruidDataSource dataSource = new DruidDataSource();
	dataSource.setDriverClassName("com.mysql.jdbc.Driver");
	dataSource.setUrl("jdbc:mysql://localhost:3306/SBL?autoReconnect=true&useUnicode=true&characterEncoding=utf-8");
	dataSource.setUsername("root");
	dataSource.setPassword("root");
	dataSource.setMinIdle(60);
	dataSource.setMaxActive(100);
	return dataSource;
}
```
上面方法的值都是直接写入的，当然也可以通过`Environment`来读取配置中的变量的值来填 入推荐使用`Environment`方式。相对于配置方式更加推荐使用代码来完成，因为在代码中还可以设置数据源其它信息，而配置中则不能。
# 数据表创建
```JAVA
CREATE TABLE `t_user`  (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `username` varchar(50) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT '用户名',
  `password` varchar(50) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT '密码',
  `tel` varchar(19) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT '电话号码',
  `nickname` varchar(50) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT '昵称',
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 1 CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci ROW_FORMAT = Compact;
```
这里创建一张简单用户表供后面进行数据的操作。
# 编码
按照平时开发中的套路来，controller中获取参数传给service层中进行业务逻辑处理，然后由dao层负责与数据库交互，所以就有下面的的结构图：![](https://i.loli.net/2018/08/10/5b6d3d2086817.png)
## service
在service类中我们通常通过@service注解来标示这是一个service，由spring在启动时自动扫描进容器中
```JAVA
public interface UserService {
    UserInfo getUserById(int id);
    List<UserInfo> getAllUser();
}

@Service
public class UserServiceImpl implements UserService {

    @Autowired
    private UserDao userDao;

    @Override
    public UserInfo getUserById(int id) {
        return userDao.getUser(id);
    }

    @Override
    public List<UserInfo> getAllUser() {
        return userDao.queryUsers();
    }
}

```
## DAO
```java
public interface UserDao {

    UserInfo getUser(int id);
    List<UserInfo> queryUsers();
}

@Repository
public class UserDaoImpl implements UserDao {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    @Override
    public UserInfo getUser(int id) {
        String sql = "select * from t_user where id= ?";
        UserInfo userInfo = jdbcTemplate.queryForObject(sql,new Object[]{id},new BeanPropertyRowMapper<>(UserInfo.class));
        return userInfo;
    }

    @Override
    public List<UserInfo> queryUsers() {
        String sql = "select * from t_user";
        List<UserInfo> userInfoList = jdbcTemplate.query(sql,new Object[]{},new BeanPropertyRowMapper<>(UserInfo.class));
        return userInfoList;
    }
}
```
## controller
这里使用`RestController`注解而没有使用`Controller`注解，因为这里不需要页面跳转。
```JAVA
@RestController
public class UserController {

    @Autowired
    private UserService userService;

    @GetMapping("/user/{id}")
    public UserInfo getUser(@PathVariable("id") int id) {
        UserInfo userInfo = userService.getUserById(id);
        return  userInfo;
    }

    @GetMapping("users")
    public List<UserInfo> getUsers(){
        List<UserInfo> userInfoList = userService.getAllUser();
        return userInfoList;
    }
}
```
# 测试
测试直接用项目里的测试类进行，也可以通过测试工具postman或者浏览器里直接访问来看结果。
```JAVA
@RunWith(SpringRunner.class)
@SpringBootTest(classes=SblApplication.class,webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public class SblApplicationTests {

    private Logger logger = LoggerFactory.getLogger(SblApplication.class);

    @Autowired
    TestRestTemplate template;

    @Test
    public void getUser() {
        //查询单个
        ResponseEntity<UserInfo> responseEntity =  template.getForEntity("http://localhost:9090/user/{id}",UserInfo.class,2);
        logger.info(responseEntity.getBody().toString());

        //查询所有
        ResponseEntity<List<UserInfo>> responseEntity1 = template.exchange("http://localhost:9090/users", HttpMethod.GET, null, new ParameterizedTypeReference<List<UserInfo>>() {
        });
        logger.info(responseEntity1.getBody().toString());
    }
}
```
测试结果如下：
```JAVA
top.cqjokers.sbl.SblApplication: UserInfo{id=2, username='测试', password='123', nickname='123'}
top.cqjokers.sbl.SblApplication: [UserInfo{id=1, username='cqjokers', password='123', nickname='大叔'}, UserInfo{id=2, username='测试', password='123', nickname='123'}]
```
这里直接用的toString方法，所以重写下UserInfo的toString方法即可。