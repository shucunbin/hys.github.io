---
title: Spring Security 学习札记（二）：登陆用户数据
toc: true
date: 2021-11-13 17:30:15
tags: Spring Security
categories: Spring Security
---



# 保存登陆用户
如果使用 Spring Security 这一类安全管理框架，大部分的开发者可能会将登陆用户数据保存在 Session 中，事实上，Spring Security 也是这么做的，并在此基础上做了一些改进，其中最主要的一个变化就是线程绑定。
当用户登陆成功后，Spring Security 会将登陆成功的用户信息保存到 SecurityContextHolder 中，SecurityContextHolder 中的数据保存默认是通过 ThreadLocal 来实现的，使用 ThreadLocal 创建的变量只能被当前线程访问，不能被其它线程访问和修改，也就是用户数据和请求线程绑定在一起。当登陆请求处理完毕后，Spring Security 会将 SecurityContextHolder 中的数据拿出来保存到 Session 中，同时将 SecurityContextHolder 中的数据清空。以后每当有请求到来时，Spring Security 就会从 Session 中取出用户登陆数据，保存到 SecurityContextHolder 中，方便在该请求的后续护理过程中使用，同时在请求结束时将 SecurityContextHolder 中的数据拿出来保存到 session 中，然后将 SecurityContextHolder 的数据清空。
这种策略在子线程中想要获取用户登陆数据时比较麻烦，Spring Security 对此也提供来相应的解决方案，如果开发者使用 `@Async` 注解来开启异步任务的话，那么只需要添加如下配置，使用 Spring Security 提供的异步任务代理，就可以在异步任务中从 SecurityContextHolder 里边获取当前登陆用户的信息：
```java
@Configuration
public class ApplicationConfiguration extends AsyncConfigurerSupport {
    @Override
    public Executor getAsyncExecutor() {
        return new DelegatingSecurityContextExecutorService(Executors.newFixedThreadPool(5));
    }
}
```

# 获取登陆用户
登陆用户获取主要分为两种方式：
- 从 SecurityContextHolder 中获取
- 从当前请求对象中获取

## 从 SecurityContextHolder 中获取
### Authentication
登陆用户信息使用 Authentication 对象表示，Authentication 对象主要有两方面的功能：
- 作为 AuthenticationManager 的输入参数，提供用户认证的凭证，当它作为一个输入参数时，它的 isAuthenticated 方法返回 false，表示用户还未认证。
- 代表已经经过身份认证的用户，此时的 Authentication 可以从 SecurityContext 中获取。

一个 Authentication 对象主要包含三个方面的信息：
- principal：定义认证的用户，如果用户使用用户名/密码的方式登陆，principal 通常就是一个 UserDetails 对象；
- credentials：登陆凭证，一般就是指密码。当用户登陆成功之后，登陆凭证会被自动擦除，以防止泄漏；
- authorities：用户被授予的权限信息。

常见的 Authentication 实现类：

| 类名                                | 作用                                                         |
| ----------------------------------- | ------------------------------------------------------------ |
| AbstractAuthenticationToken         | 实现了 Authentication、CredentialsContainer 两个接口，其中 CredentialsContainer 提供了登陆凭证擦除方法。一般登陆成功后，为了防止用户信息泄漏，可以将登陆凭证擦除。 |
| RememberMeAuthenticationToken       | 使用 RememberMe 登陆时，存储登陆信息                         |
| TestingAuthenticationToken          | 单元测试时封装的用户对象                                     |
| AnonymousAuthenticationToken        | 匿名登录时封装用户对象                                       |
| RunAsUserToken                      | 替换验证身份时封装的用户对象                                 |
| UsernamePasswordAuthenticationToken | 表单登陆时封装的用户对象                                     |
| JaasAuthenticationToken             | JAAS 认证时封装的用户对象                                    |
| PreAuthenticatedAuthenticationToken | Pre-Authentication 场景下封装的用户对象                      |



### SecurityContextHolder
SecurityContextHolder 中存储的是 SecurityContext，SecurityContext 中存储的是 Authentication。SecurityContext 的存储方式由 SecurityContextHolderStrategy 决定，目前存在三种类型的 SecurityContextHolderStrategy，分别为：
- ThreadLocalSecurityContextHolderStrategy （默认）
- InheritableThreadLocalSecurityContextHolderStrategy
- GlobalSecurityContextHolderStrategy

还有一个 与SecurityContext 相关的重要过滤器  ` SecurityContextPersistenceFilter`，它的作用是为了存储 SecurityContext 而设计的，主要做两件事情：
- 请求来的时候，从 HttpSession 中获取 SecurityContext 并存例入 SecurityContextHolder 中，这样在同一个请求的后续处理过程中，开发者始终可以通过 SecurityContextHolder 获取当前登陆用户信息。
- 当一个请求处理完毕时，从 SecurityContextHolder 中获取 SecurityContext 并存入 HttpSession 中（主要针对异步 Servlet），方便下一个请求到来时，再从 HttpSession 中拿出来使用，同时擦除 SecurityContextHolder 中的登陆用户信息。

上述两部操作均有 `SecurityContextRepository` 接口完成，它的三个实现类为：
- NullSecurityContextRepository（什么都没做）
- TestSecurityContextRepository（为单元测试提供支持）
- HttpSessionSecurityContextRepository（默认）

## 从当前请求对象中获取
直接在 Controller 的请求参数中放入 Authentication 或 Principal 对象获取，本质上，这些对象都是 HttpServletRequest 带来的（通过 `ServletRequestMethodArgumentResolver`），在使用了 Spring Security 框架时，HttpServletRequest 的实现类为 `Servlet3SecurityContextHolderAwareRequestWrapper`。
Spring Security 使用 `SecurityContextHolderAwareRequestFilter `过滤器将默认的请求对象转化为 Servlet3SecurityContextHolderAwareRequestWrapper。

# 定义登陆用户的其它方式
Spring Security 支持多种用户定义的方式，其抽象接口为 `UserDetailsService`，其不同的实现类表示不同的用户定义方式，将配置好的 UserDetailsService 提供给 AuthenticationManagerBuilder，系统再将 UserDetailsService 提供给 AuthenticationProvider 使用。

## 基于内存
我们使用配置文件配置用户时，本质上就是基于内存，对应的实现类为 `InMemoryUserDetailsManager`。

## 基于 JdbcUserDetailsManager
JdbcUserDetailsManager 支持将用户数据持久化到数据库，同时它还封装类一系列操作用户的方法，例如用户的添加、更新、查找等。
Spring Security 中为 `JdbcUserDetailsManager` 提供了数据库脚本（无法自定义用户表、角色表等，这意味着这种方式的局限性很大），位置在 org/springframework/security/core/userdetails/jdbc/users.ddl。

## 基于 MyBatis
使用 MyBatis 做数据持久化是目前大多数企业应用采用的方案，Spring Security 中结合 MyBatis 可以灵活地定制用户表以及角色表。
