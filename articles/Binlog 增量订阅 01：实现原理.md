目前有两款流行的 MySQL binlog 订阅工具，分别是阿里巴巴的 Canal 以及 zendesk 的 Maxwell，Canal 的功能相较于 Maxwell 更为全面，社区也较为活跃；利用 binlog 订阅工具，我们可以读取 binlog 并结合业务做如下几个典型功能：

1. 数据库实时备份；
2. 数据异构；
3. 业务缓存刷新、业务数据处理；

为了尝试深入订阅工具的实现原理，在阅读部分源码以及查阅官方文档后，笔者使用 Spring Boot 与 Netty，基于 MySQL 5.7 以上的版本，自行实现了轻量级的 binlog 订阅工具（[neptune-binlog-thief](https://github.com/notayessir/neptune-binlog-thief)），这里做一些总结回顾。

### 实现原理

![implementation](https://github.com/notayessir/blog/blob/main/images/binlog/implementation.png)

如图所示，订阅工具利用 MySQL 主从同步的机制，模拟 slave 接收来自 master 推送的 binlog 事件，并完成 binlog 事件到各个中间件的分发。

### 协议步骤

![negotiation](https://github.com/notayessir/blog/blob/main/images/binlog/negotiation.png)

如上图所示，要成功模拟 slave 需要完成几个步骤：

1. slave 连接上 master 节点后，master 首先发出 handshake request packet，即握手请求，这个包会携带数据库的版本、特性、随机加密数据等信息；
2. slave 收到 handshake request packet 后，需要快速回复一个已授权的账号信息（若没有及时响应，服务器会返回一个超时提醒并主动断开连接），并用 1 步骤里返回的随机加密数据以及对应的加密算法对密码进行加密，打包进 handshake response packet 回复；
3. 不同的 MySQL 版本会提供不同的密码加密方式，常见的有（按安全性排序） old_password < native_password < caching_sha2_password 等，master 收到握手响应之后会根据授权账号所使用的加密方式进行验证对比，若验证成功，master 返回 ok packet，进行下一步；
4. slave 发送 register slave packet，这个包携带一个 slave 节点 id 信息，表示自己要注册成为一个 slave 节点；
5. master 收到 register slave packet，响应 ok packet；
6. slave 发送 binlog dump packet，表示已准备好接收 binlog 事件，这个包会携带需要接收的 binlog 文件以及文件中的起始位置，我们的增量实现也是借此特点进行实现；
7. master 收到 binlog dump packet，开始推送 binlog 事件至 slave 节点；

以上 7 步是最理想的交互步骤，若一切顺利，事件会主动从 master 同步到 slave。理想之外，还会出现几种情况：

1. 主从节点加密协议不对而出现 auth switch 的步骤，这时 slave 需要切换加密方式再次验证账号密码；
2. 若要支持 ssl，需要额外的交互步骤；

针对 1，笔者只对 native_password 加密方式进行了支持，所以在创建授权账号时需要指定加密方式；

针对 2，暂未实现；

### 参考文档

Canal ：https://github.com/alibaba/canal

Maxwell：https://github.com/zendesk/maxwell

MySQL 主从连接：https://dev.mysql.com/doc/internals/en/connection-phase.html

Authentication Methods：https://dev.mysql.com/doc/dev/mysql-server/latest/page_protocol_connection_phase_authentication_methods.html

