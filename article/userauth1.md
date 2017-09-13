# 用户认证鉴权系统改造梳理（1）

近期在对用户系统改造升级，因为之前系统中用户登陆授权，鉴权相关服务都是调用集团提供的接口，一旦有客服反馈有
用户登陆不了，常常要一层一层去找集团相关部门，例如用户组，安全组，风控组等，
最后可能查的结果是**用户在使用集团另一个产品线的产品时设置的密码太简单，被安全组
设置为高危用户了**，排查过程响应慢，且常常没有结果，严重影响到了我们产品的登陆体验。更
关键是我们的产品正和竞品打的正酣，运营或产品同学
好不容易争取过来的一个用户，不能就这么流失了。


扯了这么多，说回正事。在用户系统的改造过程中，用户的登陆授权，鉴权是非常重要的一部分，毕竟涉及到系统安全，用户隐私等，
考虑的点也非常多，下面梳理了下做设计方案时候要考虑的一些问题和处理思路。

----
## 讨论假设

为了方便讨论问题，我们假设该系统目前架构如下图：


![](http://7xpnzc.com1.z0.glb.clouddn.com/auth.png)

上图说明： 用户在手机上填写用户名+密码提交后，客户端发送http(s)请求到系统接入层，接入层对request请求进行一些
基本处理，然后调用url对应的RPC服务，如上图中的认证授权服务进行校验，授权成功后，会返回一个**认证成功令牌token**给客户端，此后用户再访问需要登录的接口时候，只需要带上这个令牌token即可，而不要再填写用户名密码进行验证，同时http接入层会完成对携带token的校验，而不用每个业务模块都调用授权认证模块进行校验token是否正确。这里用户的每一次请求都是无状态的，服务器端拓展业务也无需考虑类似session共享、同步问题。

-----
## 如何生成令牌token

验证完用户名+密码后，我们返回给客户端的token格式如何构造呢？
> 一个随机的字符串，例如uuid。

用户访问的时候，接入层拿到uuid，这时候为了验证这个uuid是系统自己生成的，就需要调用RPC服务去数据库，或则redis查询，是否有这个值，及是否过期。
咋一看满足了需求，实现也简单，但可能有以下潜在问题，需要注意：
1. 由于令牌是随机字符串，拿到令牌就会泄露用户信息，所以要注意接口放刷，避免被人不停尝试。
2. 通过字符串我们无法直接得知丢应用户信息，及是否是系统生成的和是否过期，也就是说每次进行鉴权时候都要
查询至少一次后台服务，当系统访问量大时候，会对系统带宽，数据存储系统造成不必要压力。
3. 有随机数碰撞问题。
4. 依赖问题，如果鉴权模块出现问题，会导致所有需要登录才能访问的业务模块都受到不必要牵连。

> RSA非对称加密

验证完用户名+密码后，我们把用户的信息例如:
* 用户uid
* 令牌过期时间expireTime
* 其他信息字段


 通过RSA算法，加密，得到一个加密后的字符串。这样一来，我们验证时候，只需要在接入层进行RSA解密操作，然后
 对解密后的内容进行校验就行了，不需要再每次都去数据库查询了，减少了系统IO.
 但是这样又产生了一些问题：
 1. 如果密匙泄露了或者被破解了怎么办？破解难度可能会大些（例如，加入一些随机字符串，增加难度）我们可忽略，但泄露就太容易了，
 因为生成token的系统可能只有一个，但要进行token鉴权的系统即上图的接入层可能就不止一个了，例如app端一个，后台系统一个，商户系统，IM系统等等，这些**客户端**（使用我们封装鉴权服务的内部业务线服务），人多混杂，水平参差，很容易泄露，一不小心返回给页面，例如 input hidden中。
2. 该算法本身是一个相对耗CPU的操作。
3. 没法及时踢出用户，即黑名单功能。当用户因为某些行为被拉黑后，由于鉴权时候接入层不和授权认证系统进行交互，没法第一时间就阻值用户继续访问系统，只能等到token过期，重新授权时候才能踢出用户。





