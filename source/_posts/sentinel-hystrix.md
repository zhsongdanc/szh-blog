---
title: sentinel
author: demus
top: false
cover: false
mathjax: false
toc: true
abbrlink: 103
date: 2023-10-08 10:30:07
categories:
tags:
img:
coverImg:
password:
summary:
keywords:
---

📚 微服务容错与流量治理核心机制笔记
时间：2025年11月20日

一、注册中心的心跳机制：为什么需要双向通信？
1. 客户端 → 注册中心（心跳续约）
   目的：证明服务实例存活，防止被误剔除。
   实现：客户端定期（如 Eureka 默认 30s）发送心跳。
   优点：轻量、简单、注册中心压力小。
2. 注册中心 → 客户端（主动探测）
   目的：
   应对客户端异常退出（如 kill -9）；
   防止“假活实例”（进程卡死但心跳正常）；
   提高服务状态准确性。
   典型实现：
   Consul：主动 HTTP/TCP 健康检查；
   Nacos：对持久化实例进行 TCP 探测；
   ZooKeeper：通过 Session 机制隐式探测。
3. 为什么 Eureka 不主动探测？
   设计哲学：AP 系统，优先保证高可用；
   风险：可能保留“僵尸实例”；
   应对：可集成 Spring Boot Actuator，让客户端在健康检查失败时主动停止心跳。

二、Eureka 的自我保护机制（Self-Preservation）
1. 设计目标
   防止网络分区或瞬时故障导致大量健康实例被误删，避免雪崩。
2. 触发条件（默认）
   每分钟期望心跳数 = 实例数 × 2（因默认 30s 心跳一次）；
   若实际心跳数 < 期望值 × 85%（renewal-percent-threshold），则进入自我保护模式。
3. 自我保护期间行为
   ❌ 停止剔除过期实例；
   ✅ 允许新服务注册；
   ⚠️ 暂停集群间数据同步；
   ✅ 继续提供服务发现（返回所有实例，含可能失效的）；
   🟥 控制台显示红色警告。
4. 关键配置
   yaml
   eureka:
   server:
   enable-self-preservation: true # 默认开启（生产勿关！）
   renewal-percent-threshold: 0.85
   instance:
   lease-renewal-interval-in-seconds: 30 # 心跳间隔
   lease-expiration-duration-in-seconds: 90 # 失效等待时间
   💡 最佳实践：生产环境务必开启；配合 Actuator 健康检查提升准确性。

三、Sentinel 核心机制详解
1. 功能概览
   ✅ 限流（QPS/线程数/关联/链路）
   ✅ 熔断降级（慢调用比例、异常比例）
   ✅ 系统负载保护
   ✅ 热点参数限流
   ✅ 实时监控 + 动态规则（支持 Nacos/Apollo）
2. 熔断如何工作？
   Sentinel 自动统计以下指标（基于滑动窗口）：
   请求总数
   异常数 / 异常比例
   平均 RT（响应时间）
   慢调用比例（耗时 > 阈值的请求占比）
   熔断策略：
   类型 触发条件示例
------ -------------
慢调用比例 10秒内 ≥5 次调用，60% 耗时 > 500ms
异常比例 10秒内 ≥5 次调用，错误率 ≥ 50%
🔸 注意：只有抛出异常的调用才计入错误；返回错误码不算。
3. Sentinel 是轻量级的吗？
   ✅ 是！默认在当前线程执行，无额外线程开销；
   通过 AOP 式拦截 + 内存滑动窗口统计；
   对比 Hystrix 的线程池模型，性能更高、资源占用更少。
4. fallback 需要自己实现
   java
   @SentinelResource(
   value = "myService",
   blockHandler = "handleBlocked" // 限流/熔断时调用
   )
   public String myService() { ... }

public String handleBlocked(BlockException ex) {
return "系统繁忙，请稍后再试";
}
📌 Sentinel 只负责“拦”，不负责“兜”。

四、Hystrix vs Sentinel：熔断与限流对比

能力 Hystrix Sentinel
------ -------- ---------
熔断（错误率） ✅ 支持 ✅ 支持（+ 慢调用比例）
QPS 限流 ❌ 不支持 ✅ 支持
并发控制 ✅（线程池 / 信号量） ✅（线程数流控）
热点参数限流 ❌ ✅
动态规则 困难 ✅（Nacos 等）
轻量级（无额外线程） ❌（默认线程池） ✅（默认当前线程）
💡 Hystrix = 熔断器 + 隔离器，不是限流器。

五、JDBC 查询为何无法被线程中断？
1. 根本原因
   JDBC 使用阻塞 I/O（socket read/write）；
   Java 的 Thread.interrupt() 对阻塞 I/O 无效；
   线程会一直等待数据库返回，直到：
   查询完成，或
   socket/database 超时。
2. 正确中断 JDBC 的方法
   ✅ 方案1：Statement.cancel()
   驱动向数据库发送取消请求（如 MySQL 的 KILL QUERY）；
   需持有原 Statement 对象。
   ✅ 方案2：setQueryTimeout(seconds)
   java
   PreparedStatement ps = conn.prepareStatement(sql);
   ps.setQueryTimeout(3); // 最多执行3秒
   驱动内部启动监控线程，超时后自动调用 cancel()；
   强烈推荐在所有可能慢的 SQL 上设置！

六、Hystrix 超时与熔断的真实机制
1. 熔断 ≠ 线程中断
   熔断：基于错误率的状态机（CLOSED → OPEN）；
   进入 OPEN 状态后，直接拒绝请求，不执行 run() 方法；
   与线程中断无关。
2. 超时才涉及线程中断（仅线程池模式）
   主线程调用 future.get(timeout)；
   超时后：
   主线程立即走 fallback；
   同时调用 future.cancel(true) 尝试 interrupt 工作线程；
   但若 run() 中是 JDBC 查询，interrupt 无效 → 工作线程仍卡住。
3. 为什么主线程还能返回？
   主线程和工作线程解耦；
   主线程只关心“是否在 timeout 内拿到结果”，不等待工作线程真正结束；
   用户早已收到 fallback 响应，但工作线程仍在后台执行（直到 DB 返回）。
4. 风险
   Hystrix 线程池可能被慢查询占满；
   数据库连接未释放，可能导致连接池耗尽；
   数据库仍在执行“已取消”的查询，浪费资源。
   ✅ 解决方案：必须配合 setQueryTimeout()，从源头控制查询耗时。

七、总结与建议

场景 推荐方案
------ --------
服务注册发现 Eureka（AP，高可用） / Nacos（灵活） / Consul（CP，强一致）
熔断 + 限流一体化 ✅ Sentinel（轻量、功能全、动态规则）
仅需熔断（老系统） Hystrix（注意线程开销）或 Resilience4j
数据库防慢查询 所有 SQL 设置 setQueryTimeout() + 监控慢日志
生产环境稳定性 注册中心开启自我保护 + 客户端健康检查 + 服务端熔断限流
🌟 核心思想：
注册中心：宁可多留，不可错删（AP）；
流量治理：既要防“太多请求”，也要防“下游太慢”；
数据库访问：永远不要信任 SQL 执行时间，必须设超时！

📝 本笔记可作为微服务容错设计的参考手册。建议结合实际项目配置实践，并持续监控指标（QPS、RT、错误率、线程池状态）。

✅ 完
