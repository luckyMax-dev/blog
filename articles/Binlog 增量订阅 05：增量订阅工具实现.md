有了前面几节的知识，可以自己实现一个 binlog 增量订阅的工具，于是便有了这个实践项目 [neptune-binlog-thief](https://github.com/notayessir/neptune-binlog-thief)，它有几个功能点：

1. 能够订阅 MySQL 5.7 以上的 binlog 事件；
2. 实现了日志推送到 Redis、Rocket MQ、Kafka；
3. 使用 Raft 协议实现高可用，基于 Apache Ratis；

### 应用架构

![architecture](https://github.com/notayessir/blog/blob/main/images/binlog/architecture.png)

### 模块切分

项目分成 6 个模块。

#### neptune-binlog-thief-common

定义了被共同使用的类，如多个 event 类型、column 类型、以及应用的属性配置；

#### neptune-binlog-thief-connector

基于 netty 的 slave 的实现，负责协议交互、协议解析，然后将解析好的协议、事件帧通过 Spring Listener 机制发布到 neptune-binlog-thief-processor 模块；

#### neptune-binlog-thief-processor

neptune-binlog-thief-processor 模块负责解析各类 event ，并将 event 按需要推送到各个中间件；

#### neptune-binlog-thief-ha

使用 Apache Ratis 实现应用的高可用；

#### neptune-binlog-thief-bootstrap

在 Spring Boot 中嵌入订阅工具，根据 spring 的生命周期启动与停止 binlog 应用；

### 参考文档

Spring Boot：https://spring.io/projects/spring-boot

Disruptor：https://lmax-exchange.github.io/disruptor/

Netty：https://netty.io/
