---
title: Springboot整合Mybatis
date: 2023-01-19 14:14:15 +0800 
categories: [java, spring, Mybatis]
tags: [history, spring, mybatis] 
---

### Springboot整合Mybatis
#### 1. Springboot模块化开发
##### 1.1 创建项目和模块
**方法**：新建Maven项目--->在项目名右击新建Module---->选择Maven（若需要Spring依赖，通过maven添加）
**注意**：选择Module是千万不要选择`Spring Initializer`,这是新建一个springboot项目，非模块；当然，自己修改POM.xml,也可改成模块，**不建议**。
完成后，项目的`POM.xml`文件如下：会显示`modules`标签
```xml
<modules>
    <module>api</module>
    <module>service</module>
    <module>domain</module>
</modules>
```
而模块的`POM.xml`文件，`parent`标签会改：
```xml
<parent>
    <artifactId>ModuleTest</artifactId>
    <groupId>com.ymm.crm</groupId>
    <version>1.0-SNAPSHOT</version>
</parent>
```
这就给项目划分好模块了。
##### 1.2 依赖的版本号管理
通过项目的`POM.xml`文件，`dependencyManagement`可以对的各个模块的依赖的版本进行统一管理。
```xml
<properties> <!--在此统一更改-->
    <spring.version>2.4.5</spring.version>
    <junit.version>4.12</junit.version>
    <java.version>1.8</java.version>
</properties>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId><!--dependencies中都是版本号，可以点进去看-->
            <version>${spring.version}</version>   <!--获取propertise的值-->
            <type>pom</type>
            <scope>import</scope>  <!--子模块必须用这个版本-->
        </dependency>
   ...
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>${junit.version}</version>
            <scope>test</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```
> **注意：** maven 依赖的两个原则：最短路径优先原则和最先声明原则。

#### 2. springboot整合Mybatis
##### 2.1 导入mybatis相关依赖
```xml
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
</dependency>

<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>2.1.3</version>
</dependency>
```

##### 2.2 编写dao数据库操作接口，以及在对应的mapper.xml中编写相应的sql语句（也可以写在注解中，不建议）
> dao接口

```java
import org.apache.ibatis.annotations.Mapper;

import org.apache.ibatis.annotations.Param;
import org.springframework.stereotype.Component;
import java.util.List;

@Mapper
@Component
public interface SeaRelationDao {
    //@Select(sql语句) 第二种写法
    List<String> querySalersByCustomer2(@Param("id") long id, @Param("custType") long custType);
}
```

> ***.xml

```xml
<?xml version="1.0" encoding="UTF-8" ?>

<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd" >
<mapper namespace="com.ymm.crm.dao.SeaRelationDao">
    <select id="querySalersByCustomer2" resultType="java.lang.String">
        select name_list from sea_relation
        where id = #{param1} and cust_type = #{param2};
    </select>
</mapper>
```
> 注意：**接口文件名** 和 **对应的xml文件名** 必须完全一致

##### 2.3 在`application.properties`添加数据库连接配置信息
> 数据库连接

```properties
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
spring.datasource.url=jdbc:mysql://localhost:3306/mydb01?use。。。。e&serverTimezone=Asia/Shanghai&useSSL=false

spring.datasource.username=root
spring.datasource.password=123456
```

> mybatis的xml文件位置

```properties
mybatis.mapper-locations=classpath*:mapper/**/*.xml  
# mybatis的映射的实体类的位置
mybatis.type-aliases-package=com.ymm.crm.entity

# mybatis.config-location=classpath:mybatis-config.xml
# 因为引入mybatis依赖是starter启动器，所以可以在 applicaion.properties 进行配置
mybatis.configuration.call-setters-on-nulls=true
mybatis.configuration.map-underscore-to-camel-case=true
```
> mybatis配置详解 **官方网址**：[https://mybatis.org/mybatis-3/zh/configuration.html](https://mybatis.org/mybatis-3/zh/configuration.html)

>**注意**，mybatis配置也可用java类用@Configuration标签进行配置，同时也可以用xml文件进行配置；

##### 2.4 maven静态资源过滤问题
如果需要不过滤java和resources文件下xml、properties文件，在maven项目pom.xml加配置
- directory：指定文件所在的目录，目录地址是相对pom.xml而言
- includes：指定要包含哪些文件
- filtering：false不过滤

```xml
<resources>
    <resource>
        <directory>src/main/java</directory>
        <includes>
            <include>**/*.properties</include>
            <include>**/*.xml</include>
        </includes>
        <filtering>false</filtering>
    </resource>
    <resource>
        <directory>src/main/resources</directory>
        <includes>
            <include>**/*.properties</include>
            <include>**/*.xml</include>
        </includes>
        <filtering>false</filtering>
    </resource>
</resources>
```
写在build标签下。
完成基本配置，可以通过接口查询数据库。
 
### 数据库连接池
#### 1. C3p0连接池
1.  添加依赖
```xml
<dependency>
    <groupId>com.mchange</groupId>
    <artifactId>c3p0</artifactId>
    <version>0.9.5.5</version>
</dependency>
```

> **_注意_**： 注意此依赖的 `gruopId`，有过更改，特别注意

1. 配置`application.properties`文件。
```properties
# c3p0连接池, c3p0配置没有提示
# 定义c3p0的配置，没有提示可以使用，数据库连接地址
c3p0.jdbcUrl=jdbc:mysql://localhost:3306/mydb01?useUnicode=true&characterEncoding=utf8&allowMultiQueries=true&serverTimezone=Asia/Shanghai&useSSL=false
# 数据库用户名
c3p0.user=root
# 数据库密码
c3p0.password=123456
# 数据库驱动程序
c3p0.driverClass=com.mysql.cj.jdbc.Driver
# 最小连接数
c3p0.minPoolSize=2
# 最大连接数
c3p0.maxPoolSize=10
# 最大等待时间
c3p0.maxIdleTime=3000
# 初始化连接数
c3p0.initialPoolSize=5
```

> 虽然配置了`c3p0`的属性，但是不会生效。因为默认是使用的java.sql.Datasource的类来获取属性的，有些属性datasource没有。如果我们想让配置生效，需要创建c3p0的配置类。

2. `c3p0`的配置类

```java
import org.springframework.boot.context.properties.ConfigurationProperties;

import org.springframework.boot.jdbc.DataSourceBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import javax.sql.DataSource;
/**
 * @author jiakai.wu
 */
@Configuration
public class C3p0Config {

    @Bean(name = "datasource")
    @ConfigurationProperties(prefix = "c3p0")
    public DataSource dataSource() {
        return DataSourceBuilder.create()
                .type(com.mchange.v2.c3p0.ComboPooledDataSource.class)
                .build();
    }
}
```

> **c3p0**属性详解 **官网：**[https://www.mchange.com/projects/c3p0/#configuration_properties](https://www.mchange.com/projects/c3p0/#configuration_properties)

#### 2. Druid连接池
##### a. 使用启动器
- 添加依赖

```xml
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid-spring-boot-starter</artifactId>
    <version>1.1.10</version>
</dependency>
```

> **_注意_** ：使用的`version`是 `1.1.10`高版本会导致`druid`的监控界面无法显示的报`404`错误的问题

- 在`application.properties`进行配置连接池

```properties
spring.datasource.druid.url=jdbc:mysql://localhost:3306/mydb01?useUnicode=true&characterEncoding=utf8&allowMultiQueries=true&serverTimezone=Asia/Shanghai&useSSL=false
spring.datasource.druid.driver-class-name=com.mysql.cj.jdbc.Driver
spring.datasource.druid.username=root
spring.datasource.druid.password=123456
# 下面为连接池的补充设置，应用到上面所有数据源中
# 初始化大小，最小，最大
spring.datasource.druid.initial-size=5
spring.datasource.druid.min-idle=5
spring.datasource.druid.max-active=20
# 配置获取连接等待超时的时间
spring.datasource.druid.max-wait=60000
# 配置间隔多久才进行一次检测，检测需要关闭的空闲连接，单位是毫秒
spring.datasource.druid.time-between-eviction-runs-millis=60000
# 配置一个连接在池中最小生存的时间，单位是毫秒
spring.datasource.druid.min-evictable-idle-time-millis=300000
spring.datasource.druid.validation-query=SELECT 1 FROM DUAL
spring.datasource.druid.test-while-idle= true
spring.datasource.druid.test-on-borrow=false
spring.datasource.druid.test-on-return=false

# 打开PSCache，并且指定每个连接上PSCache的大小
spring.datasource.druid.pool-prepared-statements=true
#   配置监控统计拦截的filters，去掉后监控界面sql无法统计，'wall'用于防火墙
spring.datasource.druid.max-pool-prepared-statement-per-connection-size=20
spring.datasource.druid.filters=stat,wall
spring.datasource.druid.use-global-data-source-stat=true
# 通过connectProperties属性来打开mergeSql功能；慢SQL记录;键值对不会表示
#spring.datasource.druid.connect-properties=druid.stat.mergeSql=true;druid.stat.slowSqlMillis=5000

# 配置监控服务器
spring.datasource.druid.stat-view-servlet.login-username=admin
spring.datasource.druid.stat-view-servlet.login-password=123456
spring.datasource.druid.stat-view-servlet.reset-enable=false
spring.datasource.druid.stat-view-servlet.url-pattern=/druid/*
## 添加IP白名单
#spring.datasource.druid.stat-view-servlet.allow=127.0.0.1
## 添加IP黑名单，当白名单和黑名单重复时，黑名单优先级更高
#spring.datasource.druid.stat-view-servlet.deny=127.0.0.1

# 添加过滤规则
spring.datasource.druid.web-stat-filter.url-pattern=/*
# 忽略过滤格式
spring.datasource.druid.web-stat-filter.exclusions="*.js,*.gif,*.jpg,*.png,*.css,*.ico,/druid/*"
```

>相关配置详细信息见官网：[https://github.com/alibaba/druid/tree/master/druid-spring-boot-starter](https://github.com/alibaba/druid/tree/master/druid-spring-boot-starter)

>**PS:** 也可以使用配置类进行配置，如**不使用启动器**的配置类

##### b.不使用启动器
- 添加依赖

```xml
<!--添加 依赖-->
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid</artifactId>
    <version>1.1.16</version>
</dependency>
```

- 在`application.properties`的数据库连接部分加上type类型，指定连接池

```properties
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver

spring.datasource.url=jdbc:mysql://localhost:3306/mydb01?.......=false
spring.datasource.username=root
spring.datasource.password=123456
spring.datasource.type=com.alibaba.druid.pool.DruidDataSource
# 以上有代码自动提示

# 最小连接数
spring.datasource.minPoolSize=2
# 最大连接数
spring.datasource.maxPoolSize=10
# 最大等待时间
spring.datasource.maxIdleTime=3000
# 初始化连接数
spring.datasource.initialPoolSize=5
```

> 虽然我们配置了druid连接池的其它属性，但是不会生效。因为默认是使用的java.sql.Datasource的类来获取属性的，有些属性datasource没有。如果我们想让配置生效，需要创建Druid的配置文件。

- 手动创建配置文件

```java
/**
 * @author jiakai.wu
 */
@Configuration
public class DruidDataSourceConfig {

    @Bean(name = "datasource")
    @ConfigurationProperties(prefix = "spring.datasource") // 前缀
    public DataSource dataSource() {
        return new DruidDataSource();
    }
}
```
结果
`2021-05-20 16:52:22.004  INFO 13216 --- [nio-8080-exec-1] com.alibaba.druid.pool.DruidDataSource   : {dataSource-1} inited`

**Druid的优势**：**在于它有着强大的监控**
```java
/**
     * 配置监控服务器
     * @return 返回监控注册的servlet对象
     * @author SimpleWu
     */
@Bean
public ServletRegistrationBean  servletRegistration() {
    ServletRegistrationBean servletRegistrationBean = new ServletRegistrationBean(new StatViewServlet(),
                                                                                  "/druid/*");
    // 添加IP白名单
    servletRegistrationBean.addInitParameter("allow", "127.0.0.1");
    // 添加IP黑名单，当白名单和黑名单重复时，黑名单优先级更高
    //servletRegistrationBean.addInitParameter("deny", "127.0.0.1");
    // 添加控制台管理用户
    servletRegistrationBean.addInitParameter("loginUsername", "admin");
    servletRegistrationBean.addInitParameter("loginPassword", "123456");
    // 是否能够重置数据
    servletRegistrationBean.addInitParameter("resetEnable", "false");
    return servletRegistrationBean;
}

/**
     * 配置服务过滤器
     * @return 返回过滤器配置对象
     */
@Bean
public FilterRegistrationBean statFilter() {
    FilterRegistrationBean filterRegistrationBean = new FilterRegistrationBean(new WebStatFilter());
    // 添加过滤规则
    filterRegistrationBean.addUrlPatterns("/*");
    // 忽略过滤格式
    filterRegistrationBean.addInitParameter("exclusions", "*.js,*.gif,*.jpg,*.png,*.css,*.ico,/druid/*,");
    return filterRegistrationBean;
}
```

配置完后我们启动SpringBoot程序访问：`http://localhost:8080/druid/`就可以来到我们的登录页面面就是我们上面添加的控制台管理用户，
我们可以在上面很好的看到运行状况.
**druid监控**
![druid监控](/assets/img/2023-01-19-Springboot整合Mybatis/druid监控.png)
 
