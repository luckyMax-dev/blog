笔者在上家的工作内容之一，是维护平台的登录注册系统，除了公司自身的账密登录方式外，还需要接入国内外主流的第三方登录方式，例如 Google、Apple、微信等，接入之后，玩家点击几个按钮就可以快速的进入游戏，从而免去公司自身账号体系手动输入信息的步骤。笔者曾经统计过公司某款游戏刚开服时一段时间内，账号类型的分布情况，账密与第三方类型占比为 3:7，以至于后续某些游戏客户端 SDK 版本中，第三方登录会放在更显目的位置，减少玩家进入游戏的成本。

在接入各个第三方登录时，服务端经常做的一件事就是将客户端传过来的 auth code，传到第三方登录验证服务进行验证，从返回的信息中，获取玩家在该第三方的唯一 id，用于后续登录注册逻辑。这一步骤即是 OAuth2.0 授权协议的一环，OAuth2.0 用于让**用户授权给第三方软件访问受限资源**，也有其他形式的解释，这里不做展开。本篇总结 OAuth2.0 的几种授权许可类型的**流程及使用场景**。

### 授权许可类型

#### 授权码许可

使用场景：第三方应用有自己的前后端，具有安全验证 auth code 的能力，该第三方属于非官方旗下的应用。授权码许可的安全性较高，所以也是市面上最常用的许可方式。

![authcode](https://github.com/notayessir/blog/blob/main/images/oauth2/authcode.png)

#### 资源拥有者凭据许可

使用场景：官方旗下的应用，因此用户可以信赖并提供账号密码。

![password](https://github.com/notayessir/blog/blob/main/images/oauth2/password.png)

#### 客户端凭据许可

使用场景：访问的是与用户无关的资源，比如对第三方公开的信息。

![client_credentials](https://github.com/notayessir/blog/blob/main/images/oauth2/client_credentials.png)

#### 隐式许可

使用场景：第三方软件没有后端服务，操作均在前端完成。由于 access token 直接返回到前端，该类型的安全性最低。

![token](https://github.com/notayessir/blog/blob/main/images/oauth2/token.png)

### 补充说明

- 在请求授权服务时，四种授权类型的 grant_type 分别为 authcode、password、client_credentials、token。
- 当第三方应用没有后端参与时，还可以使用 PKCE 协议来提高授权的安全性，降低攻击面。
- 上面没有介绍 refresh token 相关机制，简单说，refresh token 用来刷新 access token 的有效期，refresh token 也有自己的有效期，过期就需要用户重新授权。
- OAuth2.0 中的 access token 通常使用 JWT 这种具有结构化的数据结构来实现，通过 JWT 携带一些信息可以减少通信步骤。
- OIDC 是目前各大互联网使用的用户身份认证的开放标准，OIDC 在 OAuth2.0 的基础上，加入了身份认证的功能。

### 参考资料

OAuth2.0 官方：https://oauth.net/2/

理解 OAuth 2.0：https://www.ruanyifeng.com/blog/2014/05/oauth_2_0.html

JWT：https://jwt.io/