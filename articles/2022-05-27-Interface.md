# 写好一个接口

接口是前后端交互的重要桥梁。在厂里打工近 3 年，见过具备高可读性、高性能的接口，也见过写得毫无章法的接口设计；自己在开发中也耳濡目染，挖了一些坑，又填了一些坑；于是在这做个总结，**如何写好一个接口**，或者说，**一个好的接口应该具备哪些特性**。下面会列举一些基于 Spring 框架开发的正反例来说明。

### 1、参数与校验

**尽可能的定义一个 bean 来接收参数**。在日常开发中，经常看到接口的第一个参数就是 request，接着就是在方法内进行 request.getParameter("param")，当接口接收的参数少时，尚可接受，一旦入参超过 10 个，该方法便占领了半个屏幕，非常不利用阅读和做字段注释，如果接口依赖 Swagger 等工具生成的文档，会带来较大阻碍。因此，当接收的参数超过一定数量时，使用 bean 来接收参数。

**在 bean 中做好参数校验**。做好参数校验可以避免 NPE，有了校验，接口不会出现莫名其妙的错误。可以定义一个参数检查接口，让所有 bean 去实现该参数检查的方法。

反例：

```java
@RequestMapping("/signIn")
public ResultBean<?> signIn(HttpServletRequest request, HttpServletResponse response) {
    String gameCode = request.getParameter("gameCode");
    String accessToken = request.getParameter("oauth_token");
    String accessTokenSecret = request.getParameter("oauth_token_secret");
    String language = request.getParameter("language");
    if(StringUtils.isBlank(gameCode) || StringUtils.isBlank(accessToken)){
        // 参数检查错误
	}
    ...
}

@RequestMapping("/signIn")
public ResultBean<?> signIn(String gameCode, String accessToken, String accessTokenSecret, String language) {
    if(StringUtils.isBlank(gameCode) || StringUtils.isBlank(accessToken)){
        // 参数检查错误
	}
    ...
}
```

正例：

```java
@RequestMapping("signIn")
public ResultBean<?> signIn(ThirdLoginParams params)  {
    if(params.invalid()){
        // 参数检查错误
    }
    ...
}
```

如果接口接收的参数较少，可以打破这种规则。

### 2、细分返回码

**用返回码告诉调用者发生了什么**。例如在一个登录接口，我们针对账号不存在、密码错误、第三方验证错误、参数错误、系统维护等所有情况进行返回码细分，这样做的好处是，调用端可以根据你给的文档清楚的知道当前发生了什么，当反馈到负责人时，也能够快速的定位到原因。**更重要的一个好处是，可以结合监控系统及时告警**。例如采集接口的返回码，再根据特定的 code 进行告警，比如密码错误次数返回的 code 为 110，当触发次数过多时，发出告警，相关人员收到后，可以确认是否有人在尝试盗号。

正例：

```java
@RequestMapping("signIn")
public ResultBean<?> signIn(ThirdLoginParams params)  {
    if(params.invalid()){
        return Result.build("1001", "params is empty.");
    }
    User user = userService.find(params.getUsername());
    if(Objects.isNull(user)){
    	return Result.build("1002", "username or password incorrect.");
    }
    if(StringUtils.equal(params.getPassword(), user.getPassword())){
    	return Result.build("1003", "username or password incorrect.");
    }
    ...
}
```



### 3、限制访问次数

**保护受限的资源**。当开发的接口涉及数据增改的时候，需要注意接口的防刷，例如评论接口、注册接口，常见的做法是，若接口需要登录状态才能访问，就针对 uid 做次数限制；当接口不需要登录状态时，也可以使用设备号、IP 做次数限制；尤其是一些付费资源，例如短信接口，需要更进一步的限制，例如在发送手机验证码前，增加一层行为验证码来抵御脚本攻击，例如拖动、旋转、点选验证码，虽不能说 100% 挡住攻击，但提高了许多门槛，攻防是一直在变化的；

### 4、限制地区访问

若公司存在多个地区的运营中心，需要考虑是否需要做访问地域的限制，IP 归属地判断。例如港澳台的中心，不允许大陆 IP 访问；日本中心不允许除日本外的 IP 访问；如果攻击者发起 CC 攻击，那他需要有足够多的相应地区的肉鸡，这一点也增加了攻击者的成本。我们可以在网关层、或接口前置拦截器做好拦截判断，快速的返回，减少命中接口的重逻辑。

### 5、缓存与异步

**使用缓存能提高接口响应速度**。抛开缓存带来的数据一致性问题，结合场景将数据库查到的数据放到缓存中，通常访问缓存只需要 1 ms，这样可以缓解数据库压力。在拥有良好的缓存框架和延迟双删的系统下，缓存能够给接口带来更大的响应能力；

将一些逻辑异步化也能降低响应时间。例如在账号登录过程中，玩家的登录记录不需要同步插入，可以通过 MQ 发布到对应的消费者中完成入库操作。

### 6、注意外部调用

**防止线程被耗尽以致无法响应**。当接口内部存在 RPC 调用的时候，要评估调用的耗时，以及对方服务的稳定性。Tomcat 线程是有限的，如果线程因为外部调用被一直占用，会使整个 Tomcat 的响应能力降低，依赖当前服务的上游也会出现问题。随着微服务的流行，一些 RPC 框架在设计的时候也具备熔断、超时的机制，可以充分利用这些特性保证服务的稳定性。有一些没有上微服务的系统可要小心了，最低限度也要设置一个超时，以释放线程资源。

有这几点的考虑，接口安全性是有了一定的保证。有一些特定的场景，需要定制化的访问策略，可以视情况而定。
