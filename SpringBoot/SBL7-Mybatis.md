---
title: Spring Boot (七)Mybatis的整合
tags:
  - spring boot
  - mybatis
categories:
  - SpringBoot
abbrlink: 8b2a3900
date: 2018-08-14 16:52:33
---
![](https://i.loli.net/2018/08/14/5b729814549fb.jpg)
# 前言
MyBatis 是一款优秀的持久层框架，它支持定制化 SQL、存储过程以及高级映射。MyBatis 避免了几乎所有的 JDBC 代码和手动设置参数以及获取结果集。MyBatis 可以使用简单的 XML 或注解来配置和映射原生信息，将接口和 Java 的 POJOs(Plain Old Java Objects,普通的 Java对象)映射成数据库中的记录。
# 项目创建
在之前的[工程创建](https://cqjokers.top/note/2018/08/08/170f521.html)基础上勾选上MySql与Mybatis如图所示：![](https://i.loli.net/2018/08/14/5b728838bcee0.png)项目创建好后发现POM.xml中多了以下依赖：
```JAVA
<dependency>
	<groupId>org.mybatis.spring.boot</groupId>
	<artifactId>mybatis-spring-boot-starter</artifactId>
	<version>1.3.2</version>
</dependency>

<dependency>
	<groupId>mysql</groupId>
	<artifactId>mysql-connector-java</artifactId>
	<scope>runtime</scope>
</dependency>
```
按照平时开发的套路创建项目结构，如下图所示：![](https://i.loli.net/2018/08/14/5b7289b2205fd.png)这里并没有创建dao层，后面将进行说明。
<!--more-->
# MyBatis Generator配置
## pom配置
首先在pom.xml中添加插件配置，如下所示：
```JAVA
<plugin>
	<groupId>org.mybatis.generator</groupId>
	<artifactId>mybatis-generator-maven-plugin</artifactId>
	<version>1.3.5</version>
	<dependencies>
		<dependency>
			<groupId> mysql</groupId>
			<artifactId> mysql-connector-java</artifactId>
			<version> 5.1.39</version>
		</dependency>
		<dependency>
			<groupId>org.mybatis.generator</groupId>
			<artifactId>mybatis-generator-core</artifactId>
			<version>1.3.5</version>
		</dependency>
	</dependencies>
	<executions>
		<execution>
			<id>Generate MyBatis Artifacts</id>
			<phase>package</phase>
			<goals>
				<goal>generate</goal>
			</goals>
		</execution>
	</executions>
	<configuration>
		<!--允许移动生成的文件 -->
		<verbose>true</verbose>
		<!-- 是否覆盖 -->
		<overwrite>true</overwrite>
		<!-- mybatis-generator配置文件路径 -->
		<configurationFile>
			src/main/resources/mybatis-generator/mybatis-generator.xml</configurationFile>
	</configuration>
</plugin>
```
## mybatis-generator配置文件创建
在resources目录中新建一个mybatis-generator目录并在其中创建一个`mybatis-generator.xml`文件，其内容如下：
```JAVA
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE generatorConfiguration
        PUBLIC "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN"
        "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd">

<generatorConfiguration>
    <properties resource="mybatis-generator/mybatisGenerator.properties"/>
    <context id="memberTables" targetRuntime="MyBatis3">
        <!-- 自动识别数据库关键字，默认false，如果设置为true，根据SqlReservedWords中定义的关键字列表；
            一般保留默认值，遇到数据库关键字（Java关键字），使用columnOverride覆盖 -->
        <property name="autoDelimitKeywords" value="true"/>
        <!-- 生成的Java文件的编码 -->
        <property name="javaFileEncoding" value="utf-8"/>
        <!-- beginningDelimiter和endingDelimiter：指明数据库的用于标记数据库对象名的符号，比如ORACLE就是双引号，MYSQL默认是`反引号； -->
        <property name="beginningDelimiter" value="`"/>
        <property name="endingDelimiter" value="`"/>

        <!-- 格式化java代码 -->
        <property name="javaFormatter" value="org.mybatis.generator.api.dom.DefaultJavaFormatter"/>
        <!-- 格式化XML代码 -->
        <property name="xmlFormatter" value="org.mybatis.generator.api.dom.DefaultXmlFormatter"/>
        <plugin type="org.mybatis.generator.plugins.SerializablePlugin"/>

        <plugin type="org.mybatis.generator.plugins.ToStringPlugin"/>
        <commentGenerator>
            <!-- 是否去除自动生成的注释 true：是 ： false:否 -->
            <property name="suppressAllComments" value="true"/>
        </commentGenerator>
        <!--数据库连接的信息：驱动类、连接地址、用户名、密码 -->
        <jdbcConnection driverClass="${jdbc_driver}"
                        connectionURL="${jdbc_url}"
                        userId="${jdbc_user}"
                        password="${jdbc_password}">
        </jdbcConnection>

        <!-- 默认false，把JDBC DECIMAL 和 NUMERIC 类型解析为 Integer，为 true时把JDBC DECIMAL 和
                NUMERIC 类型解析为java.math.BigDecimal -->
        <javaTypeResolver>
            <property name="forceBigDecimals" value="false"/>
        </javaTypeResolver>

        <!-- targetProject:生成PO类的位置 -->
        <javaModelGenerator targetPackage="${po_package}" targetProject="${project}">
            <!-- enableSubPackages:是否让schema作为包的后缀 -->
            <property name="enableSubPackages" value="false"/>
            <!-- 从数据库返回的值被清理前后的空格 -->
            <property name="trimStrings" value="true"/>
        </javaModelGenerator>

        <!-- targetProject:mapper映射文件生成的位置 -->
        <sqlMapGenerator targetPackage="mappers" targetProject="${resource}">
            <!-- enableSubPackages:是否让schema作为包的后缀 -->
            <property name="enableSubPackages" value="false"/>
        </sqlMapGenerator>

        <!-- targetPackage：mapper接口生成的位置 -->
        <javaClientGenerator type="XMLMAPPER" targetPackage="${mapper_package}" targetProject="${project}">
            <!-- enableSubPackages:是否让schema作为包的后缀 -->
            <property name="enableSubPackages" value="true"/>
        </javaClientGenerator>
        <table tableName="t_user" enableCountByExample="false" enableUpdateByExample="false"
               enableDeleteByExample="false"
               enableSelectByExample="false" selectByExampleQueryId="false"/>

    </context>
</generatorConfiguration>
```
在文件首部引用了一个`mybatisGeneratorinit.properties`文件，会使用其中的属性值来设置配置中相关属性，配置中的table是需要自动生成代码的数据表，可以添加多个。
## mybatisGenerator.properties创建
在resources目录中的mybatis-generator目录中创建一个`mybatisGenerator.properties`文件其内容如下所示：
```JAVA
project=src/main/java
resource=src/main/resources
po_package=top.cqjokers.sbl.pojo
mapper_package=top.cqjokers.sbl.mapper
jdbc_driver=com.mysql.jdbc.Driver
jdbc_url=jdbc:mysql://localhost:3306/sbl?tinyInt1isBit=false
jdbc_user=root
jdbc_password=root
```
该文件主要是配置了自动生成的实体类所在位置与mapper的位置及数据库连接信息。
## 生成代码与mapper的xml
创建mybatisGenerator的运行配置，如下图所示：![](https://i.loli.net/2018/08/14/5b728dd6e9cfc.png)![](https://i.loli.net/2018/08/14/5b728e1773050.png)创建好后直接运行若出现下图示例则代表代码生成成功：![](https://i.loli.net/2018/08/14/5b728e909b5a1.png)这时会发现工程的结构有所变化，多了pojo、mapper包，resources目录中多了一个mappers目录。其工程结构如下所示：![](https://i.loli.net/2018/08/14/5b728f1190669.png)到此自动生成代码就结束。
# application.yml的配置
代码生在好后在使用mybatis之前需要对其进行相关配置，如下面所示：
```JAVA
server:
  port: 9090
spring:
  profiles:
    active: dev
  datasource:
    driver-class-name: com.mysql.jdbc.Driver
    url: jdbc:mysql://localhost:3306/sbl?autoReconnect=true&useUnicode=true&characterEncoding=utf-8
    username: root
    password:
mybatis:
  type-aliases-package: top.cqjokers.sbl.mapper
  mapper-locations: classpath:mappers/*Mapper.xml
  configuration:
    call-setters-on-nulls: true
```
# 编码
在`TUserMapper`中添加一个`selectUsers`方法这里使用mybatis的注解方式，目的是想与xml配置方式区别开来，如下所示：
```JAVA
@Mapper
public interface TUserMapper {
    int deleteByPrimaryKey(Integer id);

    int insert(TUser record);

    int insertSelective(TUser record);

    TUser selectByPrimaryKey(Integer id);

    int updateByPrimaryKeySelective(TUser record);

    int updateByPrimaryKey(TUser record);

    @Select("select * from t_user")
    List<TUser> selectUsers();
}
```
在顶部加入了`Mapper`注解，如果不使用它则需要在启动类上加入`MapperScan`注解并填入Mapper包的路径
Service代码如下所示：
```JAVA
@Service
public class UserServiceImpl implements UserService {

    @Autowired
    private TUserMapper userMapper;

    @Override
    public List<TUser> selectUsers() {
        return userMapper.selectUsers();
    }

    @Override
    public TUser selectUser(int userId) {
        return userMapper.selectByPrimaryKey(userId);
    }
}
```
controller代码如下：
```JAVA
@RestController
public class UserController {

    @Autowired
    private UserService userService;

    @RequestMapping("/users")
    public Object getUsers() {
        List<TUser> userList = userService.selectUsers();
        return userList;
    }

    @RequestMapping("/user/{id}")
    public Object getUser(@PathVariable("id") int userId){
        return userService.selectUser(userId);
    }

}
```
# 测试
编写测试类如下所示：
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
        ResponseEntity<TUser> responseEntity =  template.getForEntity("http://localhost:9090/user/{id}",TUser.class,2);
        logger.info(responseEntity.getBody().toString());

        //查询所有
        ResponseEntity<List<TUser>> responseEntity1 = template.exchange("http://localhost:9090/users", HttpMethod.GET, null, new ParameterizedTypeReference<List<TUser>>() {
        });
        logger.info(responseEntity1.getBody().toString());
    }

}
```
运行测试类如果在控制台看到以下日志则代码成功：
```JAVA
top.cqjokers.sbl.SblApplication:TUser [Hash = 441260727, id=2, username=测试, password=123, tel=11321316532132, nickname=123, serialVersionUID=1]
top.cqjokers.sbl.SblApplication:[TUser [Hash = 2054332292, id=1, username=cqjokers, password=123, tel=12345678901, nickname=大叔, serialVersionUID=1], TUser [Hash = 507944445, id=2, username=测试, password=123, tel=11321316532132, nickname=123, serialVersionUID=1]]
```
# 注意
如果mapper的xml文件不是放在resources中的话，那么在打包的时候一定要将xml一并打包进去否则打包后运行的项目会出现问题

该文的示例地址：https://github.com/cqjokers/Spring-Boot-Learning
