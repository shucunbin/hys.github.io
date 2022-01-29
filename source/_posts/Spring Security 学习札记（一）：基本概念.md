---
title: Spring Security 学习札记（一）：基本概念
toc: true
date: 2021-11-12 17:30:15
tags: Spring Security
categories: Spring Security
---


# 概述
Spring Security，这是一种基于 Spring AOP 和 Servlet 过滤器的安全框架。它提供全面的安全性解决方案，同时在 Web 请求级和方法调用级处理身份确认和授权。在 Spring Framework 基础上，Spring Security 充分利用了依赖注入（DI，Dependency Injection）和面向切面技术。

# 实现机制
在 Sping Security 中，认证、授权等功能都是基于过滤器来完成的。以下是常见的过滤器，这里的是否默认加载指的是引入 Spring Security 依赖之后，开发者不做任何配置时，会自动加载的过滤器：

| 过滤器                                   | 过滤器作用                                                   | 是否默认加载 |
| ---------------------------------------- | ------------------------------------------------------------ | ------------ |
| ChannelProcessingFilter                  | 过滤请求协议，如 HTTPS 和 HTTP                               | NO           |
| WebAsyncManagerIntegrationFilter         | 将 WebAsyncManager 与 Spring Security 上下文进行集成         | YES          |
| SecurityContextPersistenceFilter         | 在处理请求之前，将安全信息加载到 SecurityContextHolder中以方便后续使用。请求结束后，再擦除 SecurityContextHolder 中的信息 | YES          |
| HeaderWriterFilter                       | 头信息加入响应中                                             | YES          |
| CorsFilter                               | 处理跨域问题                                                 | NO           |
| CsrfFilter                               | 处理 CSRF 攻击                                               | YES          |
| LogoutFilter                             | 处理注销登陆                                                 | YES          |
| OAuth2AuthorizationRequestRedirectFilter | 处理 OAuth2 认证重定向                                       | NO           |
| Saml2WebSsoAuthenticationRequestFilter   | 处理 SAML 认证                                               | NO           |
| X509AuthenticationFilter                 | 处理 X509 认证                                               | NO           |
| AbstractPreAuthenticationFilter          | 处理预认证问题                                               | NO           |
| CasAuthenticationFilter                  | 处理 CAS 单点登陆                                            | NO           |
| OAuth2LoginAuthenticationFilter          | 处理 OAuth2 认证                                             | NO           |
| Saml2WebSsoAuthenticationFilter          | 处理 SAML 认证                                               | NO           |
| UsernamePasswordAuthenticationFilter     | 处理表单登陆                                                 | YES          |
| OpenIDAuthenticationFilter               | 处理 OpenID 认证                                             | NO           |
| DefaultLoginPageGeneratingFilter         | 配置默认登陆页面                                             | YES          |
| DefaultLogoutPageGeneratingFilter        | 配置默认注销页面                                             | YES          |
| ConcurrentSessionFilter                  | 处理 Session 有效期                                          | NO           |
| DigestAuthenticationFilter               | 处理 HTTP 摘要认证                                           | NO           |
| BearerTokenAuthenticationFilter          | 处理 OAuth2 认证时的 Access Token                            | NO           |
| BasicAuthenticationFilter                | 处理 HttpBasic 登陆                                          | YES          |
| RequestCacheAwareFilter                  | 处理请求缓存                                                 | YES          |
| SecurityContextHolderAwareRequestFilter  | 包装原始请求                                                 | YES          |
| JaasApiIntegrationFilter                 | 处理 JAAS 认证                                               | NO           |
| RememberMeAuthenticationFilter           | 处理 RememberMe 登陆                                         | NO           |
| AnonymousAuthenticationFilter            | 配置匿名认证                                                 | YES          |
| OAuth2AuthoricationCodeGrantFilter       | 处理OAuth2 认证中的授权码                                    | NO           |
| SessionManagementFilter                  | 处理 Session 并发问题                                        | YES          |
| ExceptionTranslationFilter               | 处理异常认证/授权中的情况                                    | YES          |
| FilterSecurityInterceptor                | 处理授权                                                     | YES          |
| SwitchUserFilter                         | 处理账户切换                                                 | NO           |






开发者所见到的 Spring Security 提供的功能，都是通过这些过滤器来实现的，这些过滤器按照既定的优先级排列，最终形成一个过滤器链。开发者也可以自定义过滤器，并通过 @Order 注解去调整自定义过滤器在过滤器链中的位置。
需要注意的是，默认过滤器并不是直接放在 Web 项目的原生过滤器链中，而是通过一个 FilterChainProxy 来统一管理。Spring Security 中的过滤器链通过 FilterChainProxy 嵌入到 Web 项目的原生过滤器链中，如下图所示：
![](https://thumbimg.dealmoon.com/dealmoon/1cd/472/5c8/ea13b279a121606b869f254.jpg)

在 Spring Security 中，这样的过滤器链不仅仅只有一个，可能会有多个，当存在多个过滤器链时，多个过滤器链之间要指定优先级，当请求达到后，会从 FilterChainProxy 进行分发，先和哪个过滤器链匹配上，就用哪个过滤器链进行处理。当系统中存在多个不同的认证体系是，那么使用多个过滤器链就非常有效。
![](https://thumbimg.dealmoon.com/dealmoon/1ba/012/2bc/a7cecf16be6783372c37615.jpg)

FilterChainProxy 作为一个顶层管理者，将统一管理 Security Filter。FilterChainProxy 本身将通过 Spring 框架提供的 DelegatingFilterProxy 整合到原生过滤器链中，如下图：
![](https://thumbimg.dealmoon.com/dealmoon/a26/10a/e40/f582706a198ab0a5febfe7e.jpg)

## 过滤器链的初始化
### ObjectPostProcessor
ObjectPostProcessor 是 Spring Security 中使用频率最高的组件之一，它是一个后置处理器，也就是当一个对象创建成功后，如果还有一些额外的事情需要补充，那么就可以通过 ObjectPostProcessor 来进行处理。

### XxxConfigurer
在 Spring Security 中，开发者可以灵活地配置项目中需要哪些 Spring Security 过滤器，一旦选定过滤器之后，每一个过滤器都会有一个对应的配置器，叫做XxxConfigurer，例如 CorsConfigurer、CsrfConfigurer 等。过滤器都是在 XxxConfigurer 中 new 出来的，然后上一节提到的 ObjectPostProcessor 的 postProcessor 方法中处理一遍，将这些过滤器注入到 Spring 容器中。

### SecurityFilterChain
SecurityFilterChain 就是 Spring Security 中的代表**过滤器链**的接口，它唯一的实现类为 DefaultSecurityFilterChain。

### SecurityBuilder
Spring Security 中各种对象的都由某个对应的 SecurityBuilder 实现类创建。SecurityBuilder 有一个较为复杂的层级实现关系，以下简要说明一下各个子类所负责的逻辑：
- AbstractSecurityBuilder，对 SecurityBuilder 接口做了改进，确保 build 只执行一次；
- AbstractConfiguredSecurityBuilder，新增了构建过程的状态的跟踪，新增了一个关键的 configurers 变量及对应的添加删除方法，用来维护所有的配置类；
- ProviderManagerBuilder 接口，扩展了 SecurityBuilder 接口，其构建的对象为 AuthenticationManager 对象；
- AuthenticationManagerBuilder，扩展了 AbstractConfiguredSecurityBuilder，并且实现了ProviderManagerBuilder 接口，构建过程主要涉及的内容包括 parentAuthenticationManager、Authentication 数据源（内存、jdbc等）、authenticationProvider（可多个） 等，构建生成对象是 AuthenticationManager 对象；
- HttpSecurity，它 的主要作用是用来构建一条过滤器链，并反映到代码上，也就是构建一个 DefaultSecurityFilterChain 对象，**一个 DefaultSecurityFilterChain 对象包含一个路径匹配器和多个 Spring 过滤器**，HttpSecurity 中通过收集各种各样的 xxxConfigurer，将 Spring Security 过滤器对应的配置类收集起来，并保存到父类 AbstractConfiguredSecurityBuilder 的 configurers 变量中，在后续的构建过程中，再使用这些 xxxConfigurer 构建为具体的 Spring Security 过滤器，同时添加到 HttpSecurity 的 filter 对象中。每一个过滤器链都会有一个 AuthenticationManager 对象来进行认证操作，这里的 AuthenticationManager 对象在 `beforeConfigure()` 方法中完成构建。
- WebSecurity，相比于 HttpSecurity，它是在一个更大的层面上去构建过滤器。一个 HttpSecurity 对象可以构建一个过滤器链，也就是一个 DefaultSecurityFilterChain 对象，而一个 项目中可以存在多个 HttpSecurity 对象，也就是可以构建多个 DefaultSecurityFilterChain 过滤器链。WebSecurity 负责将 HttpSecurity 所构建的 DefaultSecurityFilterChain（可能有多个），以及其它一些需要忽略的请求，再次重新构建为一个 **FilterChainProxy** 对象，同时添加上 HTTP 防火墙。

### FilterChainProxy
FilterChainProxy  就是我们最终构建出来的代理过滤器链，通过 Spring 提供的 DelegatingFilterProxy 将 FilterChainProxy 集成到 WebFilter 中。所以，Spring Security 中的过滤器链的最终执行，就是在 FilterChainProxy 中。

### SecurityConfigurer
SecurityConfigurer 中有两个核心方法，一个是 init 方法，用来完成配置类的初始化操作，另一个就是 configure 方法，进行配置类的配置。SecurityConfigurer 的实现类比较多，其中包含：
- SecurityConfigurerAdapter，提供一个 SecurityBuilder，用来创建出具体的配置对象。内部还定义了一个 CompositePostProcessor 对象后置处理器。
- UserDetailsAwareConfigurer，此类主要负责配置用户认证相关的组件，其中提供了获取 UserDetailsService 的抽象方法；
- AbstractHttpConfigurer，主要为了给在 HttpSecurity 中使用的配置类添加一个方便的父类，提取共同的操作。
- GlobalAuthenticationConfigurerAdapter，主要用于配置全局 AuthenticationManagerBuilder。
- WebScurityConfigurer，用它来定义 WebSecurity。
- **WebSecurityConfigurerAdapter，它是一个可以方便创建 WebScurityConfigurer 实例的基类，我们可以通过覆盖 WebSecurityConfigurerAdapter 中的方法来完成对 HttpSecurity 和 WebSecurity 的定制**。

## 整合到 Spring Boot 体系中
在 Spring Boot 中使用 Spring Security，初始化就从 Spring Security 的自动化配置类中开始。
### SecurityAutoConfiguration

#### SpringBootWebSecurityConfiguration

#### SecurityDataConfiguration

#### WebSecurityEnablerConfiguration
WebSecurityEnablerConfiguration 配置类中添加了 @EnableWebSecurity 注解，该注解是一个组合注解，它导入了三个配置类：
-  WebSecurityConfiguration，用来配置 WebSecurity
- SpringWebMvcImportSelector，判断当前环境是否存在 Spring MVC，如果存在，则引入相关配置
- OAuth2ImportSelector，判断当前环境是否存在 OAuth2，如果存在，则引入相关配置。
- @EnableGlobalAuthentication，导入配置类 AuthenticationConfiguration



# 参考资料
[Spring Security Reference](http://docs.spring.io/spring-security/site/docs/3.2.3.RELEASE/reference/htmlsingle/#preface)
[Spring Security Tutorial](http://www.mkyong.com/tutorials/spring-security-tutorials/)
