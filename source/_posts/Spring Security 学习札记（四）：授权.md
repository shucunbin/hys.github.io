---
title: Spring Security 学习札记（四）：授权
toc: true
date: 2021-11-14 17:30:15
tags: Spring Security
categories: Spring Security
---


# 支持的授权方式
Spring Security 支持的授权模式包括：
- 基于 URL 的请求授权
- 支持方法访问授权
- 支持域对象安全（ACL）
- 支持动态权限配置
- 支持 RBAC 权限模型

从技术上讲，Spring Security 中提供的权限管理功能主要有两种类型：
- 基于过滤器的权限管理（FilterSecurityInterceptor）
- 基于 AOP 的权限管理（MethodSecurityInterceptor）

基于过滤器的权限管理主要用来拦截 HTTP 请求，拦截下来之后，根据 HTTP 请求地址进行权限校验。基于 AOP 的权限管理则主要用来处理方法级别的权限问题。当需要调用某一个方法时，通过 AOP 将操作拦截下来，然后判断用户是否具备相关的权限，如果具备，则允许方法调用，否则禁止方法调用。


# 关键概念及流程

## 权限和角色
权限和角色，权限就是一些具体的操作，角色则是某些权限的集合。
针对不同的场景，权限系统的设计方案有两种：
- 用户 <=> 权限 <=> 资源
- 用户 <=> 角色 <=> 权限 <=> 资源

Spring Security 中对应的类：GrantedAuthority、SimpleGrantedAuthority、Role、RoleHierarchy

## 两种不同的权限管理实现方式
从技术上讲，Spring Security 中提供的权限管理功能主要有两种类型：
- 基于过滤器的权限管理（FilterSecurityInterceptor）
- 基于 AOP 的权限管理（MethodSecurityInterceptor）

不论是哪种，都涉及一个前置处理器和后置处理器。
基于过滤器的权限管理中，请求首先到达过滤器 FilterSecurityInterceptor，在其执行过程中，首先会由前置处理器去判断发起当前请求的用户是否具备相应的权限，如果具备，则请求继续向下走，到达目标方法并执行完毕。在响应时，又会经过 FilterSecurityInterceptor 过滤器，此时由后置处理器再去完成其它收尾工作。在基于过滤器的权限管理中，后置处理器一般是不工作的。因为基于过滤器的权限管理，实际上就是拦截请求 URL 地址，这种权限管理方式粒度较粗，而且过滤器中拿到的是响应的 HttpServeltResponse 对象，对其所返回的数据做二次处理并不方便。

基于方法的权限管理中，目标方法的调用会被 MethodSecurityInterceptor 拦截下来，实现原理是 AOP 机制。当目标方法的调用被 MethodSecurityInterceptor 拦截下之后，在其 invoke 方法中首先会由前置处理器去判断用户是否具备调用目标方法所需的权限，如果具备，则继续执行目标方法。当目标方法执行完毕并给出返回结果后，在 MethodSecurityInterceptor#invoke 方法中，由后置处理器再去对目标方法的返回结果进行过滤或者鉴权，然后在 invoke 方法中将处理后的结果返回。

相关 API：
- ConfigAttribute，**用户**请求一个资源（网络接口或者 Java 方法）所需要的角色会被封装成一个 ConfigAttribute 对象，在 ConfigAttribute 中只有一个 getAttribute 方法，该方法返回一个 String 字符串，就是角色的名称（通常都带有一个 ROLE_ 前缀）；
- SecurityMetadataSource，安全元数据源，存储当前访问的 URL 或者方法所需的权限。如果权限信息存储在数据库中，则只需重新对应的方法即可。
- 受保护的对象，如果受保护的对象是 URL 地址，则用 FilterInvocation 对象表示，如果受保护对象是一个方法，则使用 MethodInvocation 对象表示。
- 投票器，投票器由 AccessDecisionVoter 定义，其所做的事情就是比较用户所具备的角色和请求某个资源所需的 ConfigAttribute 之间的关系。
- 决策器，决策器由 AccessDecisionManager 定义， AccessDecisionManager 会同时管理多个投票器，由 AccessDecisionManager 调用投票器进行投票，然后根据投票结果作出相应的决策。
