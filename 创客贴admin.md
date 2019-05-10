[TOC]

# 目录结构如下


admin 目录结构如下：
``` 
src/        
├── main
│   ├── java
│   │   └── com
│             ├── EyuankuWebAdminApplication.java
│             ├── M                                       【service、controller层都写在此】
│             ├── bean                                    【存放枚举、bo对象、vo对象】
│             ├── config                                  【springmvc相关配置、静态资源、权限】                       
│             ├── filter                                  【已弃用】
│             └── interceptor
│   └── resources
│       ├── application.properties
│       ├── applicationContext.xml
│       ├── applicationContext_quartz.xml
│       ├── applicationContext_rocketMq.xml
│       ├── applicationContext_viewResolver.xml
│       ├── logback.xml
│       ├── profiles
│       ├── static
│       └── templates
└── test
    ├── java
    │   └── com
    └── resource
```

# maven 构建

## 参数构建

maven核心构建配置如下：
```
<profiles>
	<profile>
		<id>dev</id>
		<properties>
			<env>dev</env>
		</properties>
		<activation>
			<activeByDefault>true</activeByDefault>
		</activation>
	</profile>
	<profile>
		<id>prod</id>
		<properties>
			<env>prod</env>
		</properties>
	</profile>
</profiles>
<build>
	<plugins>
		<plugin>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-maven-plugin</artifactId>
		</plugin>
	</plugins>
	<resources>
		<resource>
			<directory>src/main/resources/profiles/${env}</directory>
			<filtering>true</filtering>
			<includes>
				<include>*.properties</include>
			</includes>
		</resource>
		<resource>
			<directory>src/main/resources</directory>
			<filtering>true</filtering>
			<includes>
				<include>*.properties</include>
				<include>*.xml</include>
				<include>templates/**</include>
			</includes>
		</resource>
		<resource>
			<directory>src/main/resources</directory>
			<filtering>false</filtering>
			<includes>
				<include>static/**</include>
			</includes>
		</resource>

	</resources>
</build>

```
主要参数介绍：

> profiles:配置自定义的属性，可配置除了profile属性外的所有pom.xml中的属性，并在构建时将profile中的属性替换掉pom.xm相应的属性

> resource : 指定加载的资源文件（java源文件除外），配合profiles,配置不同环境（prod,dev,test）的属性

## spring配置文件

```xml
<bean id="propertyConfigurer"
		class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
	<property name="locations">
		<list>
			<value>classpath:eyuanku-web-admin-dbconfig.properties</value>
			<value>classpath:eyuanku-web-admin-pay.properties</value>
			<value>classpath:eyuanku-web-admin-cache-key.properties</value>
			<value>classpath:eyuanku-web-admin-application.properties</value>
		</list>
	</property>
</bean>
```

> PropertyPlaceholderConfigurer : spring提供的属性加载扩展点，动态替换属性（通常是Java的Properties文件），
在运行时动态替换bean中的配置项（通过@Value或xml属性注入）

# 多数据源配置

本质：因为是基于mybatis的，替换就是将mybatis 依赖的DataSource替换掉，
即：将SqlSessionFactory中的数据源做动态替换。
spring 同时注入多个数据源，根据aop（注解+反射实现）

# 全局配置

## GlobalConfig配置
```java
@SpringBootApplication
@ImportResource(locations = {"classpath:applicationContext.xml"})
public class EyuankuWebAdminApplication {

    public static void main(String[] args) {
        ApplicationContext context = SpringApplication.run(EyuankuWebAdminApplication.class, args);
        GlobalConfig.setContext(context);
    }
}
```
> 本项目通过springBoot启动项目时的返回值进行配置（😝以前还真没注意这个返回值，哈哈）

> 这里给出第二种方式,实现ApplicationContextAware接口（这种在bean初始化方法的时候会赋值invokeAwareMethods()方法中会循环赋值）

```
@Component
public class GlobalConfig implements ApplicationContextAware {}
```

## oss 配置

采用名称注入：applicationContext.xml  中配置

实体类两种：

种类 | 描述 | 对应bean |
---|---|---
OssImgStorage  | 图片处理        | publicImageStorage privateImageStorage
OssFileStorage | 普通文件处理   | publicFileStorage privateFileStorage

> 注:使用直接@AutoWired 注入 

## redis 配置

applicationContext.xml 定义redisSource信息(类似于myql的DataSource),

`redisCache` 类

- 1）引入RedisSource，初始化Jedis配置
- 2）实现ICache（定义了redis的各种操作的接口）

`CacheDef` 类 extends `redisCache`

- 1）对redis 数据库，不同存储区间进行划分（db0 - db19,当然现在只是用到了0-10 及15）
- 2）static静态代码块中对不同区间的存储,RedisCache做了实例化

## quartz 定时任务  `ing`
配置是resource 根目录下，在applicationContext_quartz.xml ,暂时未发现quartz 的持久化任务配置，走的是内存存储


# 后台权限框架 
  采用springBoot 提供的Session机制的权限框架WebSecurityConfigurerAdapter

方式如下：
      
```
自定义WebSecurityConfigurerAdapter，复写方法
示例：
        @Override
        UserDetailsService userDetailsService()         提供用户的登录信息，主要是获取密码
        
        @Override
        protected void configure(HttpSecurity http)     配置登录路由
        {
            String loginPage="/index/index/auth.do";
            http.csrf().disable();
            http.exceptionHandling()
                    .authenticationEntryPoint(new EykAdminAuthEntryPoint(loginPage));
            http
                    .authorizeRequests()
                        .antMatchers("/api/**")
                        .permitAll()//API接口任意访问
                    .and()
                        .authorizeRequests()
                        .antMatchers("/index/index/downloadTemplateThumb.do")
                        .permitAll()
                    .and()
                    .authorizeRequests()
                        .regexMatchers(".*\\.do")
                        .authenticated() //任何.do请求,登录后可以访问
                    .and()
                        .formLogin()
                        .loginPage(loginPage)
                        .loginProcessingUrl("/login.do")
                        .successHandler(new EykAdminAuthSuccessHandler())
                        .failureHandler(new EykAdminAuthFailureHandler())
                        .permitAll() //登录页面用户任意访问
                    .and()
                        .logout()
                        .logoutUrl("/logOutSecrity.do")
                        .logoutSuccessUrl("/index/index/auth.do")
                        .deleteCookies("JSESSIONID");
            http.addFilterBefore(eykAdminFilterSecurityInterceptor, FilterSecurityInterceptor.class);
        }
        
         @Override
        protected void configure(AuthenticationManagerBuilder auth) throws Exception {
            // AuthenticationManager使用我们的 EykAdminUserDetailsService 来获取用户信息
            // EykAdminPassEncoder 密码加密方式自定义
            auth.userDetailsService(userDetailsService()).passwordEncoder(new EykAdminPassEncoder());
        }
```

```java
public class EykAdminUserDetailsService implements UserDetailsService {
    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException
}
```


```xml
以上代码介绍：
    UserDetailsService  :必须覆写，
        主要是：
            1）提供用户密码，用于验证授权  
            2）提供用户基本信息，用于全局访问登录用户基本信息（诸如id、username等）

    
    UserDetails需要实现，主要提供用户名、密码、认证权限列表
    UserDetails接口方法
            |- getAuthorities               获取认证信息
            |- getPassword                  获取密码
            |- getUsername                  获取用户名
            |- isAccountNonExpired          账号是否过期
            |- isAccountNonLocked           账号是否被锁定
            |- isCredentialsNonExpired      token/sessionId 是否过期
            |- isEnabled                    用户是否禁用，禁用用户无法进入系统
```

> 注：使用此WebSecurityConfigurerAdapter进行轻量级权限认证，用户名必须是username、密码必须是password,暂时未发现自定义扩展项

# 代码风格

## 不同层model划分

- DO    与数据库表对应
- DTO   与mybatis查询结果对应
- BO    service层转化使用的的对象
- VO    视图层，页面渲染需要、进行的封装

## Mybatis 操作(mysql)

- sql语句中避免出现or (可能导致索引失效)
- <if test= "" /> 避免出现


# 参见

[spring扩展点之PropertyPlaceholderConfigurer](https://leokongwq.github.io/2016/12/28/spring-PropertyPlaceholderConfigurer.html)

[maven多环境配置](https://segmentfault.com/a/1190000003908040)

[spring源码分析专栏，博主牛逼](https://blog.csdn.net/qq_21434959/article/category/8228773)

[springBoot官方文档 - 30.1 MVC Security](https://docs.spring.io/spring-boot/docs/2.1.3.RELEASE/reference/htmlsingle/#boot-features-security-mvc)

[Spring Security(三)--核心配置解读](http://blog.didispace.com/xjf-spring-security-3/)

[quartz 定时任务,博主牛逼](https://blog.csdn.net/qq_21434959/article/category/8228780)