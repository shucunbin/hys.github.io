---
title: Spring Cloud 之 远程调用
toc: true
date: 2021-08-20 15:12:10
tags: 分布式
categories: Spring Cloud
---

# RestTemplate
Spring web 提供了一个很好用的 REST 接口远程调用组件，叫做**RestTemplate**模板组件。该组件提供了多种便捷访问远程 REST 服务的方法，能够大大提高客户端的编写效率。在 SpringBoot 中可以通过以下逻辑获取 RestTemplate 实例：
```java
// 自动配置 spring.factories -> RestTemplateAutoConfiguration -> RestTemplateBuilder
@Resource
private RestTemplateBuilder restTemplateBuilder;

@GetMapping("/user/detail/v1")
public void remoteCall() {
    //...
    // 可以调用 restTemplateBuilder 提供的方法完成配置，最后调用 build 方法获取 RestTemplate 实例完成 rpc 调用
    RestTemplate restTemplate = restTemplateBuilder.build();
    //...    
}
```
本质上，RestTemplate 实现了对 HTTP 请求的封装处理，并且形成了一套模板化的调用方法。

# Feign
Feign 是在 RestTemplate 基础上封装的，使用注解的方式来**声明**一组与服务提供者 REST 接口所对应的本地 Java 接口方法。Feign 将远程 REST 接口抽象成一个**声明式**的 Feign-Clien 客户端，并且负责完成 FeignClient 客户端和服务提供方的 REST 接口绑定。
在 Spring Cloud 中使用 Feign 通过以下几个步骤实现：
```
1. 引入依赖 - spring-cloud-starter-openfeign
2. 在启动类上添加 @EnableFeignClient
3. 编写声明式接口，然后添加 @FeignClient 注解
```

使用自定义  `RequestInterceptor` 接口实现类解决 rpc 调用时的通用请求头透传问题。

# Feign + Ribbon
如果同一个服务存在多个实例，一般的负载均衡方案分为两种：
- 服务端负载均衡，在客户端和服务提供者之间使用硬件（F5）或软件（Nginx）反向代理服务进行负载均衡；
- 客户端负载均衡，客户端自己维护一份从注册中心获取的服务提供者实例列表清单，根据自己配置的负载均衡算法在客户端进行请求的分发。Ribbon 就是一个客户端的负载均衡开源组件。

Feign 组件自身不具备负载均衡能力，Feign 通过集成 Ribbon 组件实现客户端的负载均衡。在 Spring Cloud 只需引入依赖 `spring-cloud-starter-netflix-ribbon` 即可开启 Ribbon 的功能。Ribbon 支持的负载均衡策略：
- 随机策略（RandomRule）
- 线性轮询策略（RoundRobinRule）
- 响应时间权重策略（WeightedResponseTimeRule），响应时间越长，权重越小
- 最少连接策略（BestAvailableRule），选取当前连接数最少的 Provider，内部有一个 LoadBalancerStat 成员变量，存储所有 Provider 的运行状况和连接数。
- 重试策略（RetryRule），每次选取后会对选取的 Provider 进行判断，如果为 null 或者 not alive，就会在一定时限内进行循环重试。
- 可用过滤策略（AvailabilityFilteringRule），扩展了线性轮询策略，首先通过线性轮询策略选取一个 Provider，然后再判断 Provider 是否超时可用，当前连接数是否超过限制，如果都符合要求，就成功返回。
- 区域过滤策略（ZoneAvoidanceRule），该类扩展了线性轮询策略，除了过滤超时和连接数过多的 Provider 之外，还会过滤掉不符合要求的 Zone 区域内的所有节点。Spring Cloud 默认使用此策略。

Spring Cloud 中配置负载均衡策略的方式：
```
# 配置文件方式
ribbon.NFLoadBalancerRuleClassName: com.netflix.loadbalancer.BestAvailableRule - 全局配置
provider-name.ribbon.NFLoadBalancerRuleClassName = com.netflix.loadbalancer.RandomRule - 针对某个服务提供者配置

# Java 代码配置方式
// 全局
@Configuration
public class RibbonConfiguration {
    @Bean
    public IRule defaultLBStrategy() {
        return new BestAvailableRule();
    }
}

// 针对某个服务提供者
@Configuration
@RibbonClient(name = "provider-name", configuration=com.netflix.loadbalancer.RandomRule.class)
public class RibbonConfiguration {
    @Bean
    public IRule defaultLBStrategy() {
        return new RandomRule();
    }
}
```

Ribbon 的其它配置，针对全局或某个服务提供者的逻辑跟配置负载均衡策略一致，其中重试相关配置起效须添加spring-retry 依赖：
```
# 请求链接的超时时间
ribbon.ConnectTimeOut
# 请求处理的超时时间
ribbon.ReadTimeout:
# 对所有操作都重试（默认只对 GET 请求重试）
ribbon.OkToRetyrOnAllOperations: true
# 切换实例的重试次数（不包含第一个实例）
ribbon.MaxAutoRetyiesNextServer: 1
# 对当前实例的重试次数（不包含第一次）
ribbon.MaxAutoRetries: 1
# 对特定的 HTTP 响应码进行重试
ribbon.retryableStatusCodes: 400,401
# 从注册中心刷新 Provider 的时间间隔
ribbon.ServerListRefreshInterval: 2000
```

# Feign + Hystrix
Hystrix 是一个服务降级、限流、熔断组件，主要用于在远程 Provider 服务异常时，对**对消费端的 RPC 进行保护**，避免引起雪崩效应。
在 Spring Cloud 中通过以下几个步骤开启 Hystrix 功能：
```
1. 引入依赖：spring-cloud-starter-netflix-hystrix
2. 在应用属性配置文件中开启 Feign 对 Hystrix 的支持：feign.hystrix.enabled=true
3. 在服务启动类上添加 `@EnableCircuitBreaker` 或 ` EnableHystrix` 注解
```
Fallback 与 FallbackFactory 的区别：Fallback 的方式会屏蔽触发熔断的异常，而 FallbackFactory 可以保留的异常信息。

Hystrix 熔断器的工作机制：统计最近 RPC 调用发生错误的次数，然后根据统计值中的成功、失败、超时比例等信息来决定是否允许后面的 RPC 调用继续或者快速失败回退。熔断器有 3 中状态，分别为 关闭（closed）、开启（open）、半开启（half-open）。
Hystrix 的配置参数主要分为两类：熔断器相关参数和滑动窗口相关参数。
熔断器相关参数：
```
# 是否启动熔断器，默认为 true
hystrix.command.default.circuitBreaker.enabled

# 设置熔断器触发熔断的最小请求次数，默认值为 20
hystrix.command.default.circuitBreaker.requestVolumeThreshold

# 配置错误率阈值，在时间窗口时间内，当错误率超过此值时，熔断器进入 open 状态，所有请求都会发生失败回退，默认值为 50
hystrix.command.default.circuitBreaker.errorThresholdPercentage

# 配置睡眠窗口的大小，即熔断器打开之后，多久之后允许一次请求尝试执行，默认值为 5000 ms
hystrix.command.default.circuitBreaker.sleepWindownMillseconds

# 强制打开熔断器，默认值为 false
hystrix.command.default.circuitBreaker.forceOpen
```
滑动窗口相关配置：
```
# 滑动窗口的持续时间，默认值为 10000 毫秒
hystrix.command.default.metrics.rollingStats.timeInMilliseconds

# 滑动窗口被划分的时间桶数量
hystrix.command.default.metrics.rollingStats.numBuckets
```

# feign + ribbon + hystrix 完整执行流程
1. 生成 feign 注解标识的接口的动态代理实例，并完成装配；
2. HystrixInvocationHandler 中的 invoke 方法对从 dispatch 获取的 SynchronousMethodHandler 进行封装，返回 HystrixCommand 命令实例。注：HystrixInvocationHandler 实例内部有一个 map 类型 dispatch 成员，它保存这远程方法反射实例和方法处理器（MethodHandler）；
3. 执行 HystrixCommand 实例的 run 方法，内部调用 MethodHandler invoke 方法，如果调用异常，这返回回退结果；
4. 执行 MethodHandler 方法处理器的 invoke 方法，MethodHandler 实例内部有一个 client 成员，默认的 client 通过 JDK 自带的 HttpURLConnection 类完成远程调用；
5. 集成 ribbon 时，client 类型为 LoadBalancerFeignClient 负载均衡客户端时，使用 Ribbon 获取最佳 provider 实例，然后委托内部的 delegate 完成实际的请求；
6. 通过 ribbon 内部的 delegate 的 client 客户端成员完成远程URL请求执行和获取远程结果；

# 超时时间的配置
如果同时启用 feign、ribbon、hystrix 的功能，client 整体的包含逻辑为 hystrix -> retry -> ribbon/feign，以设计意图来说，Feign Client 的配置关注在每次 Http Request 上，Retry 关注的是 Feign Client 出问题以后的 Retry 逻辑，而 Hystrix 关心的是整个调用何时能成功，出错了如何 fallback，所有调用的并发控制，是否要熔断终止调用等问题。
源码入口：`FeignClientFactoryBean` -> `DefaultTargeter` ->  `ReflectiveFeign`  -> `HystrixInvocationHandler` -> `HystrixCommand` -> `LoadBalancerFeignClient` ->`OkHttpClient`。
Feign 超时相关的配置：
```
#连接超时时间
feign.client.config.default.connectTimeout=5000
#读取超时时间
feign.client.config.default.readTimeout=5000    
```

Hystrix 超时相关的配置：
```
hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds=12000
```

Ribbon 超时相关的配置：
```
#说明：同一台实例的最大自动重试次数，默认为1次，不包括首次
ribbon.MaxAutoRetries=1
#说明：要重试的下一个实例的最大数量，默认为1，不包括第一次被调用的实例
ribbon.MaxAutoRetriesNextServer: 1
#说明：是否所有的操作都重试，默认为true
ribbon.OkToRetryOnAllOperations: true
#说明：从注册中心刷新服务器列表信息的时间间隔，默认为2000毫秒，即2秒
ribbon.ServerListRefreshInterval: 2000
#说明：使用Apache HttpClient连接超时时间，单位为毫秒
ribbon.ConnectTimeout: 3000
#说明：使用Apache HttpClient读取的超时时间，单位为毫秒
ribbon.ReadTimeout: 3000
```

如果同时配置了 Feign 与 Ribbon 的超时时间，Feign 与配置会覆盖 Ribbon 的配置，详情见 `RibbonLoadBalancingHttpClient.execute()`。
Hystrix 超时时间与 Feign/Ribbon 超时时间的关系：
> Hystrix的超时时间>=Ribbon的重试次数(包含首次) * (ribbon.ReadTimeout + ribbon.ConnectTimeout)

Ribbon 重试次数：
>Ribbon重试次数(包含首次)= 1 + ribbon.MaxAutoRetries  +  ribbon.MaxAutoRetriesNextServer  +  (ribbon.MaxAutoRetries * ribbon.MaxAutoRetriesNextServer)

# 参考资料
[Feign Ribbon Hystrix 三者关系 | 史上最全, 深度解析](https://www.cnblogs.com/crazymakercircle/p/11664812.html)
[Feign 源码解析](https://xli1224.github.io/2017/09/14/feign-anaylsis/)
