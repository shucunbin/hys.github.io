---
title: Spring Security 学习札记（三）：认证
toc: true
date: 2021-11-14 17:30:15
tags: Spring Security
categories: Spring Security
---


# 支持的认证机制
Spring Security  集成的主流认证机制包括：
- 表单认证
- OAuth2.0 认证
- HTTP Basic 认证
- HTTP Digest 认证
- OpenID 去中性化认证
- RememberMe 自动认证
- SAML 2.0 认证
- CAS 认证
- JAAS 认证
- Pre-Authentication Scenarios 认证
- X509 认证
- 自定义认证逻辑

# 关键概念及流程
Authentication、AuthenticationProvider、AuthenticationManager（实现类为 ProviderManager）。
在一次完整的认证流程中，可能会同时存在多个 AuthenticationProvider，多个 AuthenticationProvider 统一由 ProviderManager 来管理。同时 ProviderManager 具有一个 parent，它是一个备选认证方式，当所有认证都失败了，就由 parent 出场收拾残局。

## Authentication
Authentication 对象主要有两方面的功能：
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


## SecurityContextHolder
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

## AuthenticationManager
AuthenticationManager 是一个认证管理器，它定义了 Spring Security 过滤器要如何执行认证操作。AuthenticationManager 在认证成功后，会返回一个 Authentication 对象，这个 Authentication 对象会被设置到 SecurityContextHolder 中。

ProviderManager  是 AuthenticationManager 的一个重要实现类。ProviderManager 和 AuthenticationProvider 之间的关系如下图：
![](https://thumbimg.dealmoon.com/dealmoon/7ef/fc3/4a4/061532f33222e1bdd60a807.jpg)
多个 AuthenticationProvider 将组成一个列表，这个列表将由 ProviderManager 代理，换而言之，在 ProviderManager 中存在一个 AuthenticationProvider 列表，在 ProviderManager 中遍历列表中的每一个 AuthenticationProvider 去执行身份认证，最终得到认证结果。
ProviderManager 本身也可以再配置一个 AuthenticationProvider 作为 parent，这样当 ProviderManager 失败之后，就可以进入 parent 中再次进行认证。ProviderManager 的 parent 可以是任意类型的 AuthenticationManager。
ProviderManager 本身也可以有多个，多个 ProviderManager 共用一个 parent，当存在多个过滤器链的时候非常有用。当存在多个过滤器链时，不同的路径可能对应不同的认证方式，但是不同路径可能又会同时存在一些共有的认证方式，这些共有的认证方式可以在 parent 中统一处理。

### 全局 ProviderManager 与 局部 ProviderManager 的配置
一个系统中，我们可以配置多个 HttpSecurity，而mei每一个HttpSecurity 都有一个对应的 AuthenticationManager，我们暂且称这种 AuthenticationManager 为局部 AuthenticationManager，这些局部的 AuthenticationManager 实例都有一个共同的 parent，那就是全局的 AuthenticationManager。
待补充


## AbstractAuthenticationProcessingFilter
作为 Spring Security 过滤器链中的一环，`AbstractAuthenticationProcessingFilter` 将 Authentication、AuthenticationManager（ProviderManager）、AuthenticationProvider 这些组件与登陆关联起来，它可以用来处理任何提交给它的身份认证。下图是 `AbstractAuthenticationProcessingFilter` 在整个通用架构中所处的位置：
![](https://thumbimg.dealmoon.com/dealmoon/325/61d/58b/2b6f4481e468d79eb9660e6.jpg)

`AbstractAuthenticationProcessingFilter` 作为一个抽象类，使用不同的方式登陆，对应不同的实现类。比如，如果使用用户名/密码的方式登陆，那么它对应的实现类是 `UsernamePasswordAuthenticationFilter`，构造出来的 Authentication 对象则是 `UsernamePasswordAuthenticationToken`。其大致流程如下：
- 当用户提交登陆请求时，UsernamePasswordAuthenticationFilter 会从当前请求 HttpServletRequest 中提取出登陆用户名/密码，然后创建出一个 UsernamePasswordAuthenticationToken 对象。
- UsernamePasswordAuthenticationToken 对象将被传入 ProviderManager 中进行具体的认证操作。
- 如果认证失败，则 SecurityContextHolder 中相关信息将被清除，登陆失败回调也会被调用。
- 如果认证成功，则会进行登陆信息存储、Session 并发处理、登陆成功事件发布以及登陆成功方法回调等。

## RememberMe
RememberMe 是一种服务器端的行为，具体的思路就是通过 Cookie 来记录当前用户身份。当用户登陆成功后，会通过一定的算法，将用户信息、时间戳等进行加密，加密完成后，通过响应头带回前端存储在 Cookie 中，当浏览器关闭之后重新打开，如果再次访问该网站，会自动将 Cookie 中的信息发送给服务器，服务器对 Cookie 中的信息进行校验分析，进而确定用户身份。

为了提高普通令牌的安全性，在其基础上的改进方案包括：持久化令牌、二次校验

### 持久化令牌
在普通令牌的基础上，新增了 series 和 token 两个校验参数，当使用用户名/密码的方式登陆时，series 才会自动更新，而一旦有了新的会话，token 就会重新生成。所以，如果令牌被别人盗用，对方使用 RememberMe 登陆成功后，就会生成新的 token，你自己的登陆令牌就会失效，这样就能及时发现账户泄漏并作出处理，比如清除自动登陆令牌、通知账户泄漏等。
建立新会话的场景：关闭浏览器再重新打开、重启服务器

### 二次校验
二次校验就是将系统中的资源分为敏感的和不敏感的，如果用户使用了 RememberMe 的方式登陆，这访问敏感资源时会自动跳转到登陆页面，要求用户重新登陆；如果使用了用户名/密码的方式登陆，则可以访问所有资源。这种方式相当于牺牲了一点用户体验换取系统安全性。

### 原理关键 API
- RememberMeServices，定义自动登陆逻辑、设置 cookie 逻辑
- RememberMeConfigurer，开启 RememberMe 功能的配置类，init() 方法中集成了 RememberMeServices、RememberMeAuthenticationProvider，configure() 方法中配置了 RememberAuthenticationFilter。
- RememberMeAuthenticationFilter，将 RememberMe 功能集成到系统的 Filter。
- RememberMeAuthenticationToken，RememberMeAuthenticationProvider 认证成功后返回的 Authentication 实现类。

>什么是 HttpOnly？
如果 cookie 中设置了 HttpOnly 属性，那么 JS 脚步将无法读取到 cookie 信息，这样能有效防止 XSS 攻击，获取 cookie 内容。XSS 全称 Cross SiteScript，跨站脚本攻击，是 Web 程序中常见的漏洞，XSS 属于被动式且用于客户端的攻击方式，其原理是攻击者向有 XSS 漏洞的网站中输入恶意的 HTML 代码，当其它用户浏览该网站时，这段 HTML 代码会自动执行，从而达到攻击的目的，如盗取用户 Cookie、破坏页面结构、重定向到其它网站等。

## 会话管理
当浏览器调用登陆接口成功后，服务器端会和浏览器之间建立一个会话（Session），浏览器在每次发送请求时都会携带一个 sessionId通过 cookie或 url 重写），服务端则根据这个 sessionId 来确定用户的身份。当浏览器关闭后，服务端的 Session 并不会自动销毁，需要开发者手动在服务端调用 Session 销毁方法，或者等 Session 过期自动销毁。

### 会话并发管理
会话并发管理就是在当前系统中，同一个用户可以同时创建多少个会话，如果一台设备对应一个会话，那么也可以简单理解为同一个用户可以同时在多少台设备进行登陆。默认情况下，同一用户在多少台设备上登陆并没有限制，开发者可执行配置。

通过配置可以实现相同的用户同时只可以登陆一台设备：相互踢或者禁止后面的设备登陆。

### 原理关键 API
- SessionInformation，记录 Spring Security 框架内的会话信息。
- SessionRegistry，用来维护 SessionInformation 实例
- SessionAuthenticationStrategy，用作在用户登陆成功后，对 HttpSession 进行处理，它有很多作用各异的实现类。
    - CsrfAuthenticationStrategy，与 CSRF 攻击有关，该类主要负责在身份验证后删除旧的 CstfToken 并生成一个新的 CsrfToken；
    - ConcurrentSessionControlAuthenticationStrategy，处理 Session 并发问题；
    - RegisterSessionAuthenticationStrategy，用于在认证成功后将 HttpSession 信息记录到 SessionRegistry 中；
    - CompositeSessionAuthenticationStrategy，复合策略，Spring Security 使用的就是该类的实例；
    - NullAuthenticationSessionStrategy：空实现；
    - AbstractSessionFixationProtectionStrategy：处理会话固定攻击的基类；
    - ChangeSessionIdAuthenticationStrategy：通过修改 sessionId 来防止会话攻击；
    - SessionFixationProtectionStrategy：通过创建一个新的会话来防止会话固定攻击。
- SessionManagementFilter，用来处理 RememberMe 登陆时的会话管理，即如果用户使用了 RememberMe 的方式进行认证，则认证成功后需要进行会话管理，相关的管理操作通过 SessionManagementFilter 过滤器触发。
- ConcurrentSessionFilter，处理会话并发管理
- SessionManagementConfigurer，完成 SessionManagementFilter、ConcurrentSessionFilter 的配置。
- AbstractAuthenticationProcessingFilter，认证成功后，调用 sessionStrategy 的 onAuthentication方法

整个流程总结：用户通过用户名／密码发起一个认证请求，当认证成功后，在 AbstractAuthenticationProcessingFilter#doFilter方法中触发了Session并发管理。默认的 sessionStrategy是Composite SessionAuthenticationStrategy，它一共代理了三个 SessionAuthenticationStrategy，分别是ConcurrentSessionControlAuthenticationStrategy、ChangeSessionIdAuthenticationStrategy以及RegisterSessionAuthenticationStrategy。当前请求在这三个SessionAuthenticationStrategy中分别走一圈，第一个用来判断当前登录用户的Session数是否已经超出限制，如果超出限制就根据配置好的规则作出处理；第二个用来修改sessionId（防止会话固定攻击）；第三个用来将当前Session注册到SessionRegistry中。使用用户名／密码的方式完成认证，将不会涉及ConcurrentSessionFilter和SessionManagementFilter两个过滤器。如果用户使用了RememberMe的方式来进行身份认证，则会通过SessionManagementFilter#doFilter方法触发Session并发管理。当用户认证成功后，以后的每一次请求都会经过ConcurrentSessionFilter，在该过滤器中，判断当前会话是否已经过期，如果过期就执行注销登录流程；如果没有过期，则更新最近一次请求时间。

### 会话固定攻击与防御
会话固定攻击（Session fixation attacks）是一种潜在的风险，恶意攻击者有可能通过访问当前应用程序创建会话，然后诱导用户**以相同的会话 ID 登陆**，通常是将会话ID作为参数放在请求链接中，然后诱导用户去点击，进而获取用户的登陆身份。

三个方面入手防范会话固定攻击：
- Spring Security 中默认自带了 Http 防火墙，如果 sessionid 放在地址栏中，这个请求就会直接被拦截下来；
- 在 Http 响应的 set-cookie 字段中设置 httpOnly 属性，避免通过 XSS 攻击来获取包含sessionid 的 Cookie 信息；
- 用户登陆成功后，改变会话 sessionid，Spring Security 默认实现来该种方案。

### 会话共享
集群环境下的会话共享解决方案：
- Session 复制：多个服务之间互相复制 session 信息，这样每个服务中都包含来所有的 session 信息，Tomcat 通过 IP 组播对这种方案提供支持。但是这种方案占用宽带、有时延，服务数量越多效率越低，所以这种方案使用较少；
- Session 粘滞：也叫会话保持，就是在 Nginx 上通过一致性 Hash，将 Hash 结果相同的请求总是分发到一个服务上去。这种方案可以解决一部分集群会话带来的问题，但是无法解决集群中的会话并发管理问题。
- Session 共享：Session 共享就是将不同服务的会话统一放在一个地方，所有的服务共享一个会话。一般使用redis 等key-value 数据库存储 session。

使用 redis 维护 session 的信息关键类是 spring-session 的 SessionRepositoryFilter。

# 参考文章
[session 会话管理](https://www.cnblogs.com/zongmin/p/13783348.html)
[@EnableRedisHttpSession原理简析](https://www.cnblogs.com/54chensongxia/p/12096493.html)
