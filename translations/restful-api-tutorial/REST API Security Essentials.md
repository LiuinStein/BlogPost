### REST API Security Essentials

安全永远不是事后才考虑的。它必须是任何开发项目以及REST API的组成部分。有多种方法来保卫RESTful API，例如[basic auth](http://howtodoinjava.com/resteasy/jax-rs-resteasy-basic-authentication-and-authorization-tutorial/), [OAuth](https://oauth.net/)等等。但是要记住一件事就是RESTful API必需是无状态的，所以请求的认证和授权必须不能依赖cookie或者session。相反，每个API请求必须带有一些授权信息，服务端每次收到请求后都要去验证这些信息。

### REST Security Design Principles

这篇论文[“The Protection of Information in Computer Systems” by Jerome Saltzer and Michael Schroeder](http://web.mit.edu/Saltzer/www/publications/protection/)提出了8条用于保持计算机系统内信息安全的原则，如下：

1. **最小权限（Least Privilege）：**一个实体应当仅有其操作所需的最小权限。只有当实体需要某个权限的时候才授权给它，当其不需要某个权限的时候要即时撤销权限
2. **安全设置缺省值（Fail-Safe Defaults）:**  一个用户对系统内任意资源的默认访问权限都应为拒绝，除非他们明确被授予某个特定的权限
3. **机制经济（Economy of Mechanism）：**设计应当越简单越好。所有的组件接口以及两者之间的交互应当足够简单以便理解
4. **完全控制（Complete Mediation）：** 一个系统应当去验证对其所有资源的访问权限而不是单单依赖于缓存过的权限模型。如果对某一个资源的访问权限已经被撤销，但是并没有被相应的反应到权限模型中，那么系统就会变得不安全
5. **开放设计（Open Design）：**这个原则强调了以公开方式建立一个系统的重要性，比如不使用保密的算法
6. **特权分离（Separation of Privilege）：**不应当单单基于一种情况来对实体授予权限，最好使用基于资源类型的条件组合
7. **最不常用机制（Least Common Mechanism）：** 这个考虑的是在不同的组件之间共享状态的风险。如果其中一个可以破坏共享状态，那么他就可以同时破坏所依赖于它的所有的组建
8. **心理可接受性（Psychological Acceptability）：** 安全机制不应当让资源变得难以获取。简要来说，安全机制不应当影响用户体验

### Best Practices to Secure REST APIs

下面给出了几个观点可以作为设计REST API时的安全机制的检查清单

#### 保持简单（Keep it Simple）

使一个API或者系统保持安全——它需要的安全性。每当你使你的解决方案变得“不必要”复杂的时候，你同样可能留下一个漏洞。

#### 始终使用HTTPS（Always Use HTTPS）

通过使用[SSL](https://www.digicert.com/ssl.htm)，认证凭证可以被简化为一个随机生成的访问token保存在HTTP Basic Auth字段里。SSL用起来也相对简单，同时你还能享受到其带来的其他安全特性。

如果你使用[HTTP 2](https://http2.github.io/)的话，为了提高性能，你可以通过一个连接来发送多个请求，这样也避免了后面的几次传输中TCP和SSL握手的时间。

#### 使用密码哈希（Use Password Hash）

密码必需被hash以保护系统的安全（或者风险最小化）即使它在一些黑客攻击中收到了损害。有许多[哈希算法](https://howtodoinjava.com/security/how-to-generate-secure-password-hash-md5-sha-pbkdf2-bcrypt-examples/)可以用来很好地保护密码安全，诸如MD5, SHA, PBKDF2, bcrypt以及scrypt等等。

#### 不要将信息暴露在URL中（Never expose informtion on URLs）

用户名，密码，session token，以及API key这些不应当出现在URL中，因为这样可以直接在web浏览记录中捕获，使得系统易于被攻击。

```
https://api.domain.com/user-management/users/{id}/someAction?apiKey=abcd123456789  //Very BAD !!
```

上述URL中就包含API key，永远不要用这种方式来确保系统安全性。

#### 考虑OAuth（Consider OAuth）

对于大多数API来说，如果实现正确的话，[basic auth](https://en.wikipedia.org/wiki/Basic_access_authentication)就已经足够了，不过如果你能考虑[OAuth](https://tools.ietf.org/html/rfc6749)那就更好了。OAuth 2.0验证框架可以确保一个第三方的应用对HTTP服务进行有限的访问，不管是通过资源拥有者与HTTP服务之间的批准交互还是允许第三方应用以自己的名义获取资源。

#### 在请求中加入时间戳（Consider Adding Timestamp in Request）

跟其他参数一块，你可以在HTTP自定义API请求头中加入请求时间戳。服务端将会比较现在的时间戳与请求中的时间戳，并且仅接受在一小段合理时间段内的请求（可能是1-2分钟）。

这可以抵御基本的[重放攻击](https://en.wikipedia.org/wiki/Replay_attack)，当这些人试图通过[暴力破解](https://en.wikipedia.org/wiki/Brute-force_attack)的方法攻入你的系统而不改变这个时间戳。

#### 请求参数验证（Input Parameter Validation）

验证请求参数往往放在第一步，在进行应用程序逻辑之前。做强的验证检查并立刻拒绝验证失败的请求。在API响应中，发送相关的错误信息并指出示例的正确数据可以提升用户体验。