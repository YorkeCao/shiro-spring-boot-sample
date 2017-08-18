# Spring Boot + Shiro 集成

Shiro 是一个流行的 Java 安全框架。

其实 Spring 有一个 Spring Security 的安全框架，我试用了一下，太复杂了。同样的安全需求，用 Shiro 要简单、快捷得多，也利于理解。

本手册的源码托管在 GitHub 上：

[YorkeCao/shiro-spring-boot-sample](https://github.com/YorkeCao/shiro-spring-boot-sample)

下面主要介绍在 Spring Boot 项目中引入 Shiro，对应用进行安全管控。

方法

## 集成

可以利用 Shiro 启动器来完成与 Spring Boot 的集成。

这里为了简单，尽量少做配置。实际上 Shiro 有很多自定义选项。详细介绍请移步[官网](http://shiro.apache.org/)。



1. **引入 Shiro 启动器**

   Shiro 官方提供了 Spring Boot 启动器：[shiro-spring-boot-starter](http://mvnrepository.com/artifact/org.apache.shiro/shiro-spring-boot-starter)，在 `pom.xml` 文件中引入：

   ```xml
   <!-- https://mvnrepository.com/artifact/org.apache.shiro/shiro-spring-boot-starter -->
   <dependency>
       <groupId>org.apache.shiro</groupId>
       <artifactId>shiro-spring-boot-starter</artifactId>
       <version>1.4.0</version>
   </dependency>
   ```

   ​

2. **配置 Shiro**

   这里我们提供一个最简单的 Java Class 配置。

   里面用到了一个自定义的 Realm：CustomRealm。

   ```java
   package io.yorkecao.sample.config;

   import io.yorkecao.sample.shiro.CustomRealm;
   import org.apache.shiro.spring.LifecycleBeanPostProcessor;
   import org.apache.shiro.spring.web.ShiroFilterFactoryBean;
   import org.apache.shiro.web.mgt.DefaultWebSecurityManager;
   import org.springframework.context.annotation.Bean;
   import org.springframework.context.annotation.Configuration;

   /**
    * @author Yorke
    */
   @Configuration
   public class ShiroConfig {

       @Bean(name = "customRealm")
       public CustomRealm customRealm() {
           return new CustomRealm();
       }

       @Bean(name = "securityManager")
       public DefaultWebSecurityManager defaultWebSecurityManager(CustomRealm customRealm) {
           DefaultWebSecurityManager  securityManager = new DefaultWebSecurityManager ();
           securityManager.setRealm(customRealm);
           return securityManager;
       }

       @Bean(name = "lifecycleBeanPostProcessor")
       public LifecycleBeanPostProcessor lifecycleBeanPostProcessor() {
           return new LifecycleBeanPostProcessor();
       }

       @Bean(name = "shiroFilter")
       public ShiroFilterFactoryBean shiroFilterFactoryBean(DefaultWebSecurityManager securityManager) {
           ShiroFilterFactoryBean shiroFilterFactoryBean = new ShiroFilterFactoryBean();
           shiroFilterFactoryBean.setSecurityManager(securityManager);
           return shiroFilterFactoryBean;
       }
   }
   ```

   ​

3. **实现自定义 Realm**

   Realm 是控制认证和授权的核心部分，也是开发人员必须自己实现的部分。

   实现自定义 Realm 最快捷的方式是继承 AuthorizingRealm 类。

   ```java
   package io.yorkecao.sample.shiro;

   import io.yorkecao.sample.dao.ShiroSampleDao;
   import org.apache.shiro.authc.*;
   import org.apache.shiro.authz.AuthorizationInfo;
   import org.apache.shiro.authz.SimpleAuthorizationInfo;
   import org.apache.shiro.realm.AuthorizingRealm;
   import org.apache.shiro.subject.PrincipalCollection;
   import org.springframework.beans.factory.annotation.Autowired;

   import java.util.Set;

   /**
    * @author Yorke
    */
   public class CustomRealm extends AuthorizingRealm {

       @Autowired
       private ShiroSampleDao shiroSampleDao;

       /**
        * 认证
        */
       @Override
       protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken authenticationToken) throws AuthenticationException {
           UsernamePasswordToken token = (UsernamePasswordToken) authenticationToken;
           String username = token.getUsername();
           String password = this.shiroSampleDao.getPasswordByUsername(username);
           return new SimpleAuthenticationInfo(username, password, getName());
       }

       /**
        * 授权
        */
       @Override
       protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principalCollection) {
           String username = (String) super.getAvailablePrincipal(principalCollection);
           SimpleAuthorizationInfo authorizationInfo = new SimpleAuthorizationInfo();
           Set<String> roles = shiroSampleDao.getRolesByUsername(username);
           authorizationInfo.setRoles(roles);
           roles.forEach(role -> {
               Set<String> permissions = this.shiroSampleDao.getPermissionsByRole(role);
               authorizationInfo.addStringPermissions(permissions);
           });
           return authorizationInfo;
       }
   }
   ```

   这里用到了一个 DAO：shiroSampleDao。应该要通过它来从我们的数据库中获取用户、角色等信息。但是为了方便，我只是模拟了一下。

   ```java
   package io.yorkecao.sample.dao;

   import org.springframework.stereotype.Repository;

   import java.util.*;

   /**
    * @author Yorke
    */
   @Repository
   public class ShiroSampleDao {

       public Set<String> getRolesByUsername(String username) {
           Set<String> roles = new HashSet<>();
           switch (username) {
               case "zhangsan":
                   roles.add("admin");
                   break;
               case "lisi":
                   roles.add("guest");
                   break;
           }
           return roles;
       }

       public Set<String> getPermissionsByRole(String role) {
           Set<String> permissions = new HashSet<>();
           switch (role) {
               case "admin":
                   permissions.add("read");
                   permissions.add("write");
                   break;
               case "guest":
                   permissions.add("read");
                   break;
           }
           return permissions;
       }

       public String getPasswordByUsername(String username) {
           switch (username) {
               case "zhangsan":
                   return "zhangsan";
               case "lisi":
                   return "lisi";
           }
           return null;
       }
   }
   ```

   ​

4. **实现 login、logout 接口**

   在 Shiro 框架中实现登录、登出很容易。

   这里我也提供了一个 read 和 write 的接口，这两个接口在 Service 实现。

   ```java
   package io.yorkecao.sample.web;

   import io.yorkecao.sample.service.ShiroSampleService;
   import org.apache.shiro.SecurityUtils;
   import org.apache.shiro.authc.UsernamePasswordToken;
   import org.apache.shiro.subject.Subject;
   import org.springframework.beans.factory.annotation.Autowired;
   import org.springframework.web.bind.annotation.GetMapping;
   import org.springframework.web.bind.annotation.RestController;

   /**
    * @author Yorke
    */
   @RestController
   public class ShiroSampleController {

       @Autowired
       private ShiroSampleService shiroSampleService;

       @GetMapping("/login")
       public void login(String username, String password) {
           UsernamePasswordToken token = new UsernamePasswordToken(username, password);
           token.setRememberMe(true);
           Subject currentUser = SecurityUtils.getSubject();
           currentUser.login(token);
       }

       @GetMapping("/logout")
       public void logout() {
           Subject currentUser = SecurityUtils.getSubject();
           currentUser.logout();
       }

       @GetMapping("/read")
       public String read() {
           return this.shiroSampleService.read();
       }

       @GetMapping("/write")
       public String write() {
           return this.shiroSampleService.write();
       }
   }
   ```

   ​

5. **通过注解设置访问资源所需权限**

   可以通过 `@RequiresPermissions` 等注解设置访问资源所需的权限。

   ```java
   package io.yorkecao.sample.service;

   import org.apache.shiro.authz.annotation.RequiresPermissions;
   import org.springframework.stereotype.Service;

   /**
    * @author Yorke
    */
   @Service
   public class ShiroSampleService {

       @RequiresPermissions("read")
       public String read() {
           return "reading...";
       }

       @RequiresPermissions("write")
       public String write() {
           return "writting...";
       }
   }
   ```

   ​

## 测试

- 在登录之前，访问 `/read` 和 `/write` 接口都无效
- 用 lisi 登录（`GET http://localhost:8080/login?username=lisi&password=lisi`）后，可以访问 `/read`，不能访问 `/write`
- 用 zhangsan 登录（`GET http://localhost:8080/login?username=zhangsan&password=zhangsan`）后，`/read` 和 `/write` 都可以访问
- 登出后，访问 `/read` 和 `/write` 接口都无效

