---
title: Netty 前置知识点之事件循环机制
toc: true
date: 2020-11-12 10:23:10
tags: 事件循环
categories: Netty
---

[原文链接](http://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/Netty%20%E6%A0%B8%E5%BF%83%E5%8E%9F%E7%90%86%E5%89%96%E6%9E%90%E4%B8%8E%20RPC%20%E5%AE%9E%E8%B7%B5-%E5%AE%8C/04%20%E4%BA%8B%E4%BB%B6%E8%B0%83%E5%BA%A6%E5%B1%82%EF%BC%9A%E4%B8%BA%E4%BB%80%E4%B9%88%20EventLoop%20%E6%98%AF%20Netty%20%E7%9A%84%E7%B2%BE%E9%AB%93%EF%BC%9F.md)
# 基本概念
事件循环（EventLoop）这个概念其实并不是 Netty 独有的，它是一种事件等待和处理的程序模型，可以解决多线程资源消耗高的问题。例如 Node.js 就采用了 EventLoop 的运行机制，不仅占用资源低，而且能够支撑了大规模的流量访问。
下图展示了 EventLoop 通用的运行模式。每当事件发生时，应用程序都会将产生的事件放入事件队列当中，然后 EventLoop 会轮询从队列中取出事件执行或者将事件分给相应的事件监听者执行。事件执行的方式通常分为立即执行、延后执行、定期执行几种。
![](https://thumbimg.dealmoon.com/dealmoon/3a2/688/621/7927f790413d80d513ade32.png)

# Netty 如何实现 EventLoop
在 Netty 中 EventLoop 可以理解为 Reactor 线程模型的事件处理引擎，每个 EventLoop 线程都维护一个 Selector 选择器和任务队列 taskQueue。它主要负责处理 IO 事件、普通任务和定时任务。
Netty 中推荐使用 NioEventLoop 作为实现类，其实现核心逻辑在 NioEventLoop 的 run() 方法中。
```java
protected void run() {
    for(;;) {
        // 1. 轮询 IO 事件
        slelect(wokenUp.getAndSet(false));
        // 2. 处理 IO 事件
        processSelectedKeys();
        // 3. 处理其它非 IO 事件
        runAllTasks();
    }
}
```
上述源码的结构比较清晰，NioEventLoop 每次循环的处理流程都包含事件轮询 select、事件处理 processSelectedKeys、任务处理 runAllTasks 几个步骤，是典型的 Reactor 线程模型的运行机制。而且 Netty 提供类一个 ioRatio，可以调整 IO 事件处理和任务处理的时间比例。

## IO 事件处理
![](https://thumbimg.dealmoon.com/dealmoon/5c5/d8e/63b/a8481886b17aaf1d4e767f4.png)
EventLoop 事件处理机制最核心的是采用**无锁串形化的设计思路**：
- BossEventLoopGroup 和 WorkerEventLoopGroup 包含一个或者多个 NioEventLoop。BossEventLoopGroup 负责监听客户端的 Accept 事件，当事件触发时，将事件注册至 WorkerEventLoopGroup 中的一个 NioEventLoop 上。每新建一个 Channel， 只选择一个 NioEventLoop 与其绑定。所以说 Channel 生命周期的所有事件处理都是线程独立的，不同的 NioEventLoop 线程之间不会发生任何交集。
- NioEventLoop 完成数据读取后，会调用绑定的 ChannelPipeline 进行事件传播，ChannelPipeline 也是线程安全的，数据会被传递到 ChannelPipeline 的第一个 ChannelHandler 中。数据处理完成后，将加工完成的数据再传递给下一个 ChannelHandler，整个过程是串行化执行，不会发生线程上下文切换的问题。

NioEventLoop 无锁串形化的设计不仅使系统吞吐量达到最大化，而且降低了用户开发业务逻辑的难度，不需要花太多精力关心线程安全问题。虽然单线程执行避免了线程切换，但是它的缺陷就是不能执行时间过长的 I/O 操作，一旦某个 I/O 事件发生阻塞，那么后续的所有 I/O 事件都无法执行，甚至造成事件积压。**在使用 Netty 进行程序开发时，我们一定要对 ChannelHandler 的实现逻辑有充分的风险意识**。
Netty 解决  JDK Epoll 空轮询的方式：
1. 每次执行 Select 操作之前记录当前时间 currentTimeNanos
2. time - TimeUnit.MILLISECONDS.toNanos(timeoutMillis) >= currentTimeNanos，如果事件轮询的持续时间大于等于 timeoutMillis，那么说明是正常的，否则表明阻塞时间并未达到预期，可能触发了空轮询的 Bug。（如何探测）
3. Netty 引入了计数变量 selectCnt。在正常情况下，selectCnt 会重置，否则会对 selectCnt 自增计数。当 selectCnt 达到 SELECTOR_AUTO_REBUILD_THRESHOLD（默认512） 阈值时，会触发重建 Selector 对象。（重建策略）

## 其它任务处理
NioEventLoop 不仅负责处理 I/O 事件，还要兼顾执行任务队列中的任务。任务队列遵循 FIFO 规则，可以保证任务执行的公平性。NioEventLoop 处理的任务类型基本可以分为三类。
1. 普通任务：通过 NioEventLoop 的 execute() 方法向任务队列 taskQueue 中添加任务。例如 Netty 在写数据时会封装 WriteAndFlushTask 提交给 taskQueue。taskQueue 的实现类是多生产者单消费者队列 MpscChunkedArrayQueue，在多线程并发添加任务时，可以保证线程安全。
2. 定时任务：通过调用 NioEventLoop 的 schedule() 方法向定时任务队列 scheduledTaskQueue 添加一个定时任务，用于周期性执行该任务。例如，心跳消息发送等。定时任务队列 scheduledTaskQueue 采用优先队列 PriorityQueue 实现。
3. 尾部队列：tailTasks 相比于普通任务队列优先级较低，在每次执行完 taskQueue 中任务后会去获取尾部队列中任务执行。尾部任务并不常用，主要用于做一些收尾工作，例如统计事件循环的执行时间、监控信息上报等。

NioEventLoop 处理任务的逻辑源码如下所示：
```java
protected boolean runAllTasks(long timeoutNanos) {
        // 1. 获取即将可执行的定时任务，放入普通任务队列中
        fetchFromScheduledTaskQueue();

        // 2. 从普通任务队列中取出任务
        Runnable task = pollTask();
        if (task == null) {
            afterRunningAllTasks();
            return false;
        }

        // 3. 计算任务处理超时时间
        final long deadline = ScheduledFutureTask.nanoTime() + timeoutNanos;
        long runTasks = 0;
        long lastExecutionTime;
        for (;;) {
            // 4. 执行任务（直接执行任务的 run 方法）
            safeExecute(task);

            runTasks ++;

            // 5. 每执行 64 个任务，检查一下是否超时
            if ((runTasks & 0x3F) == 0) {
                lastExecutionTime = ScheduledFutureTask.nanoTime();
                if (lastExecutionTime >= deadline) {
                    break;
                }
            }

            task = pollTask();
            if (task == null) {
                lastExecutionTime = ScheduledFutureTask.nanoTime();
                break;
            }
        }

        // 6. 执行 tailTasks 队列中的任务，tailTasks 相比于普通任务队列优先级较低，主要用于做一些收尾工作，例如统计事件循环执行时间、监控信息上报等。
        afterRunningAllTasks();
        this.lastExecutionTime = lastExecutionTime;
        return true;
    }
```

## 关于 EventLoop 一些好的实践
实际开发中，一些好的 EventLoop 实践：
1. 网络连接建立过程中三次握手、安全认证的过程会消耗不少时间，建议采用 Reactor 主从线程模型，MainReactor 线程处理客户端请求接入，SubReactor 线程：数据读取、I/O 事件的分发与执行。
2. 由于 Reactor 线程模式适合处理耗时短的任务场景，对于耗时较长的 ChannelHandler 可以考虑维护一个**业务线程池**，将编解码后的数据封装成 Task 进行异步处理，避免 ChannelHandler 阻塞而造成 EventLoop 不可用。
3. 如果业务逻辑执行时间较短，建议直接在 ChannelHandler 中执行。例如编解码操作，这样可以避免过度设计而造成架构的复杂性。
4. 不宜设计过多的 ChannelHandler。对于系统性能和可维护性都会存在问题，在设计业务架构的时候，需要明确业务分层和 Netty 分层之间的界限。不要一味地将业务逻辑都添加到 ChannelHandler 中。

# 参考资料
[JDK Epoll 空轮询 bug](https://www.jianshu.com/p/3ec120ca46b2)
