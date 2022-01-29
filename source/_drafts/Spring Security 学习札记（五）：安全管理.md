---
title: Spring Security 学习札记（四）：安全管理
toc: true
date: 2021-11-14 17:30:15
tags: Spring Security
categories: Spring Security
---


# Http 防火墙
在 servlet 容器规范中，HttpServletRequest 定义了一些属性，如 contextPath、servletPath、pathInfo、queryString 等，规范中没有定义这些属性可以包含哪些值，各容器实现处理的方式比较混乱，可能会造成安全隐患。
Spring Security 中通过 HttpFirewall 来检查请求路径以及参数是否合法，如果合法，才会进入到过滤器链中进行处理。

# CSRF 攻击与防御
CSRF（Cross-Site Request Forgery，跨站请求伪造），也可成为一键式攻击（one-click attack），通常缩写为 CSRF 或者 XSRF。
CSRF 攻击是一种挟持用户在当前**已登陆的浏览器**上发送恶意请求攻击的方法。相对于 XSS 利用用户对指定网站的信任，CSRF 则是利用网站对用户网页浏览器的信任。简单来说，CSRF 是攻击者通过一些技术手段欺骗用户的浏览器，去访问一个用户曾经已经通过认证的网站，然后执行恶意请求（发送邮件、发送消息、转账、购买商品等）。由于客户端（浏览器）已经在该网站上认证过，所以该网站会认为是真正的用户在操作而执行请求，而这个请求实际上并不是用户的本意。

CSRF 攻击的根源在于浏览器默认的身份验证机制（第三方发起的请求会自动携带当前网站的 Cookie 信息），这种机制虽然可以保证请求是来自用户的某个浏览器，但是无法确保该请求是用户授权发送的。攻击者和用户发送的请求一模一样，这意味着我们没有办法直接拒绝这里的某一个请求。如果能**在合法请求中额外携带一个攻击者无法获取的参数**，就可以成功区分出两种不同的请求，进而拒绝恶意请求。

Spring 中提供来两种机制俩防御 CSRF 攻击：
- 令牌同步模式，CSRF 令牌放在 cookie 中
- 在 Cookie 上指定 SameSite 属性

无论是哪种方式，前提都是请求方法幂等，即 HTTP 请求中的 GET、HEAD、OPTIONS、TRACE 方法不应该改变应用的状态。
对于前后端分离的应用，目前有什么好的办法呢？

# HTTP 响应头处理
