# Spring Boot 如何监控 SQL 运行情况？

Druid 数据库连接池相信很多小伙伴都用过，个人感觉 Druid 是阿里比较成功的开源项目了，不像 Fastjson 那么多槽点，Druid 各方面一直都比较出色，功能齐全，使用也方便，基本的用法就不说了，今天我们来看看 Druid 中的监控功能。

## 1. 准备工作

首先我们来创建一个 Spring Boot 工程，引入 MyBatis 等，如下：

![图片](https://mmbiz.qpic.cn/mmbiz_png/GvtDGKK4uYnlKeRXSyh3zIAlrkH1taibicQ8ZnWTsnrgToVPd6dtrZI9Rpn3SfvZgCzsiao4jw61vsJcPpPNYJhuA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

选一下 MyBatis 和 MySQL 驱动，做一个简单的测试案例。

先来连接一下数据库：

```properties
spring.datasource.username=gzhen
spring.datasource.password=123456
spring.datasource.url=jdbc:mysql:///ds?serverTimezone=Asia/Shanghai
```

创建一个 User 实体类，做一个简单的查询案例，如下：

```java
public class User {
    private Integer id;
    private String username;
    private String address;
    private String password;
    private String email;
    //省略 getter/setter
}
@Mapper
public interface UserMapper {
    List<User> getUserByUsername(String username);
}
@Service
public class UserService {
    @Autowired
    UserMapper userMapper;
    public List<User> getUserByUsername(String username){
        return userMapper.getUserByUsername(username);
    }
}
@RestController
public class UserController {
    @Autowired
    UserService userService;

    @GetMapping("/user")
    public List<User> getUser(String username) {
        return userService.getUserByUsername(username);
    }
}
```

UserMapper.xml 如下：

```xml
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="org.javaboy.druid_monitor.mapper.UserMapper">
    <select id="getUserByUsername" resultType="org.javaboy.druid_monitor.model.User">
        select * from user where username=#{username}
    </select>
</mapper>
```

一个很简单的测试，没啥好说的。

这个环境搭建大家随意，如果你已经有持久化的案例了，那就直接看第二小节引入 Druid。

现在这个工程默认的使用的数据库连接池是 HikariDataSource，这是 Spring Boot 中默认的一个数据库连接池，其实这个也还不错。

## 2. 引入 Druid

接下来我们引入 Druid：

```xml
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid-spring-boot-starter</artifactId>
    <version>1.2.8</version>
</dependency>
```

注意，Spring Boot 引入的 Druid 是上面这个，这个将来配置监控的时候方便一些。

接下来我们在 application.properties 中配置 WebStatFilter，WebStatFilter 用于采集 web-jdbc 关联监控的数据：

```properties
# 启用 WebStatFilter
spring.datasource.druid.web-stat-filter.enabled=true
# 配置拦截规则
spring.datasource.druid.web-stat-filter.url-pattern=/*
# 排除一些不必要的 url，这些 URL 不会涉及到 SQL 查询
spring.datasource.druid.web-stat-filter.exclusions=*.js,*.gif,*.jpg,*.png,*.css,*.ico,/druid/*
# 开启 session 统计功能
spring.datasource.druid.web-stat-filter.session-stat-enable=true
# 缺省 sessionStatMaxCount 是 1000 个，我们可以按需要进行配置
spring.datasource.druid.web-stat-filter.session-stat-max-count=1000
# 配置 principalSessionName，使得 druid 能够知道当前的 session 的用户是谁
# 根据需要，这个参数的值是 user 信息保存在 session 中的 sessionName
#spring.datasource.druid.web-stat-filter.principal-session-name=
# 下面这个配置的作用和上面配置的作用类似，这个是通过 Cookie 来识别用户
#spring.datasource.druid.web-stat-filter.principal-cookie-name=
# 开启 profile 后就能够监控单个 URL 地址调用列表
#spring.datasource.druid.web-stat-filter.profile-enable=
```

我们配置前面五个就可以了，后面三个可以不用配置，各项配置的含义松哥已经在代码中列出来了。

接下来开启 StatViewServlet 的配置，如下：

```properties
# 启用内置的监控页面
spring.datasource.druid.stat-view-servlet.enabled=true
# 内置监控页面的地址
spring.datasource.druid.stat-view-servlet.url-pattern=/druid/*
# 开启 Reset All 功能
spring.datasource.druid.stat-view-servlet.reset-enable=true
# 设置登录用户名
spring.datasource.druid.stat-view-servlet.login-username=gzhen
# 设置登录密码
spring.datasource.druid.stat-view-servlet.login-password=123456
# 白名单（如果allow没有配置或者为空，则允许所有访问）
spring.datasource.druid.stat-view-servlet.allow=127.0.0.1
# 黑名单（deny 优先于 allow，如果在 deny 列表中，就算在 allow 列表中，也会被拒绝）
spring.datasource.druid.stat-view-servlet.deny=
```

配置一下页面地址，配置一下黑白名单即可。

**需要注意的是，reset-enable 属性即使设置为 false，重置按钮也会显示，只是点击该按钮并不会重置而已。**

好啦，这就完事了。

## 3. 测试

好啦，接下来我们启动 Spring Boot 项目进行测试。

Spring Boot 项目启动成功后，首先访问如下地址：

- http://localhost:8080/druid/login.html

此时我们会看到登录认证页面，如下：

![图片](https://mmbiz.qpic.cn/mmbiz_png/GvtDGKK4uYnlKeRXSyh3zIAlrkH1taibicDNXQ4sEzbhzOibIRhcgOMS9SbC7a12tNE7y5e0L9JCkORODbkFf2kibQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

输入我们前面配置的用户名/密码（javaboy/123）进行登录，登录成功后，可以看到如下页面：

![图片](https://mmbiz.qpic.cn/mmbiz_png/GvtDGKK4uYnlKeRXSyh3zIAlrkH1taibicrufUAicuRKgNa9VnvuRj3bnt1J0S1Owya9WXNzEj3s3XuXf3UDQqVzA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

从标题栏就可以看到，数据源、SQL 监控、SQL 防火墙等功能都是一应俱全。

接下来我们访问 `http://localhost:8080/user?username=aaa` 地址，执行一条 SQL，执行完成后，我们来查看 SQL 监控：

![图片](https://mmbiz.qpic.cn/mmbiz_png/GvtDGKK4uYnlKeRXSyh3zIAlrkH1taibicKjMWtnW09cmpiaibet7Al8Q7IRkMhAQHypMxZv66VLicK7oZ6sL80z9gA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

可以看到，此时就有 SQL 执行的监控记录了。

其他的监控数据也都可以看到，我就不一一列举了。如果小伙伴们觉得这里展示的数据不直观，想自己画 HTML 页面，那也是可以的，点击最后面的 JSON API，可以看到每一个监控项的 JSON 地址，拿着 JSON 自己想怎么显示就怎么显示。

## 4. 去广告

如果想直接用这个监控页面，这个上面有阿里的广告，如下图，公司用的话就特别别扭：

![图片](https://mmbiz.qpic.cn/mmbiz_png/GvtDGKK4uYnlKeRXSyh3zIAlrkH1taibicVvRIzrDCoze1TFGekoQIhAP9jqmHbQXq5DuibWYS3rWjXgK565ErbXA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

我们可能想去掉这个广告，这也很容易。

首先，经过分析，我们发现广告是由一个叫做 common.js 的文件构建出来的，该文件位于 `druid-1.2.8.jar!/support/http/resources/js/common.js` 这里，common.js 文件中有如下几行：

```
init : function() {
 this.buildFooter();
 druid.lang.init();
},
buildFooter : function() {
 var html ='';
 $(document.body).append(html);
},
```

大概逻辑就是上面这样，buildFooter 方法负责构建页面末尾的广告，在 init 方法中完成对 buildFooter 方法的调用。

那么想要去除广告，就别调用 buildFooter 方法就行了。

所以我们的去广告思路也很简单，写一个过滤器，拦截下对 common.js 的请求，然后做一点点修改，如下：

```java
@WebFilter(urlPatterns = "/druid/js/common.js")
public class RemoveAdFilter implements Filter {
    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        String text = Utils.readFromResource("support/http/resources/js/common.js");
        text = text.replace("this.buildFooter();", "");
        servletResponse.getWriter().write(text);
    }
}
```

可以看到，这个过滤器就是拦截 `/druid/js/common.js` 请求，拦截到之后，自己去文件中读取 common.js 文件，然后手动替换掉 `this.buildFooter();` 这一句就行了，最后再把文件写出去就行了。

当然，记得在启动类中扫描 Filter，如下：

```java
@SpringBootApplication
@ServletComponentScan("org.javaboy.druid_monitor.filter")
public class DruidMonitorApplication {

    public static void main(String[] args) {
        SpringApplication.run(DruidMonitorApplication.class, args);
    }
}
```