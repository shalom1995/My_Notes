# JWT Cookie/Session OAuth 用户认证 授权管理 (包括但不限于)详解

先来梳理一下用户交互&登陆全流程吧

## 用户登陆&交互全流程

1. 第一步先进行**身份认证**，身份认证有如下几种方式：

    * 账户密码
    * 验证码(邮箱、手机号……)
    * 第三授信方验证(谷歌验证……)
    * 生物验证(人脸识别、指纹解锁……)
    * 二次验证

2. 认证成功后，为了让服务器知道后续操作是谁，就需要**维持上下文状态**，实现方式也有如下几种：

    * 使用 Cookie/Session 来维持
    * 使用 JWT

> 补充⛽️：HTTP也提供了身份认证的标准，不过目前使用不太广泛，🔗[HTTP 身份验证](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Authentication)

***

## 身份认证

身份认证目的是**让服务器知道你到底是谁，随后服务器再根据你的身份，开放相应的权限**。稍微完善的网站，都会提供用户多个登陆途径，以防某单一方法失败，导致用户无法使用整个网站。

### 账户密码

账户密码是最传统的一种方式，这种方式是存在一些风险的，有这么几种情况可能会出现密码泄露：

* 客户端：暴力破解
    > 黑客通过程序不断的穷举，直至成功试出正确的密码。
* 传输层：密码传输过程中被抓包
    > 密码明文传输；或者黑客拥有一个MD5库，可以将简单的密码摘要进行破解。
* 服务端：数据库被攻破，密码被读取或者修改

对应解决方案：

* 客户端：增加密码防护机制，比如连续5次错误后暂停5分钟……

* 传输层：客户端将密钥加密，同时使用加密传输协议，比如[HTTPS](TODO)

* 服务端：

    1. 不使用明文存储密码
    2. 加盐字段
    3. 使用不可逆的加密算法计算密文，存储密文，几种加密算法简介🔗：[加密算法](TODO)
    4. 将密文与账号ID绑定，计算密文的时候fn(密码+盐+账号ID)，这样即使脱库，攻击者也无法通过更换密钥的方式来控制账户

服务端的方案比较抽象，下面举一个案例说明一下。

#### XXX项目是如何存储密钥的？

这是一张存储管理员信息数据库表：

```sql
-- 管理员表
DROP TABLE IF EXISTS `t_admin`;
CREATE TABLE `t_admin` (
    `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    `address` CHAR(42) UNIQUE NOT NULL COMMENT '管理员地址',
    `salt` VARCHAR(18) NOT NULL COMMENT '生成摘要的盐',
    `digest` CHAR(64) NOT NULL COMMENT '摘要',
    PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci;
```

其中：`salt`是盐，`digest`是用来校验的摘要，也就是fn(密码+盐+账号ID)计算出来的结果。

下面是登陆验证时的代码：

```golang
type LoginParam struct {
   Address  string `validate:"required"`
   Password string `validate:"required"`
}

func (l *LoginParam) Verify() (bool, error) {
   m := model.TAdmin{
      Address: l.Address,
   }
   one, err := m.One()
   if err != nil {
      return false, errors.Wrap(err, "TAdmin One error")
   }

   //  计算摘要的方式在这里
   digest := sha256.Sum256([]byte(strconv.Itoa(int(one.ID)) + one.Salt + l.Password))

   return strings.ToUpper(hex.EncodeToString(digest[:])) == strings.ToUpper(one.Digest), nil
}
```

解释一下上述代码最关键的一行：使用sha256摘要算法，对“账号ID+盐+密码”进行字符串拼接后的字符串计算摘要，随后再与数据库中记录的摘要相比较。

#### 这么做的好处在哪？

* 首先，没有存储明文，这样即使数据库被攻击，黑客也无法得到用户的密码。

* 再者，即使黑客将所有账户的摘要替换成他自己账户的摘要，因为验证的时候需要与账户的ID拼接，所以计算出摘要还是不正确的。

* 最后，若黑客将ID和盐都修改，那么原本攻击的那个账户所有的权限与资源就都失效了，因为业务逻辑上是与ID绑定的。

这样，即使黑客获取了数据库用户表的信息，也无法登陆其他账户。

### 验证码认证

验证码主要是依赖手机和邮箱的稀缺性，手机的稀缺性要高于邮箱。

### 第三授信方认证

依赖第三方可以信任的平台登陆，比如：谷歌、GitHub、微信等……

这里有两个概念容易混淆：OAuth 与 身份认证。一般谷歌搜索第三方登陆，都会找到许多OAuth相关的内容，但是OAuth与我们需要的身份认证是有很大的不同的，同时也不太适合用来做身份认证。

OAuth是用来授权的，主体是第三方账户系统。比如谷歌的OAuth，获得账户许可后，可以获得操作用户谷歌云盘的权限。而谷歌第三方登陆则是由另外的接口提供。

链接🔗：

* [谷歌登录文档](https://developers.google.com/identity/gsi/web/guides/overview?hl=en)
    > 注意⚠️：谷歌第三方登录有新旧两个版本，旧版将会停止运营，不要找错了版本。

* [谷歌OAuth文档](https://developers.google.com/identity/protocols/oauth2/web-server?hl=en)

* 使用前，建议先了解其原理：[OAuth 2.0 的一个简单解释 - 阮一峰的网络日志](https://www.ruanyifeng.com/blog/2019/04/oauth_design.html) ｜ [OAuth 2.0 的四种方式 - 阮一峰的网络日志](https://www.ruanyifeng.com/blog/2019/04/oauth-grant-types.html) ｜ [GitHub OAuth 第三方登录示例教程 - 阮一峰的网络日志](https://www.ruanyifeng.com/blog/2019/04/github-oauth.html)

#### 谷歌登陆(Sign in with Google)

TODO

#### OAuth

TODO

##### 谷歌OAuth实践

TODO

### 生物认证

TODO

### 二次验证(双重认证)

二次验证是指用户登录后，需要再次输入动态口令，才可以完成登陆过程。典型的二次验证应用有谷歌的Authenticator应用。许多安全性比较高的网站都会采用类工具来验证登录。

#### 原理

其原理是：通过一致性算法，使客户端与服务端计算出相同的动态口令，并且每30秒更改口令。大多数双重认证都采用TOTP 算法🔗[基于时间的一次性密码算法-WIKI](https://zh.wikipedia.org/wiki/%E5%9F%BA%E4%BA%8E%E6%97%B6%E9%97%B4%E7%9A%84%E4%B8%80%E6%AC%A1%E6%80%A7%E5%AF%86%E7%A0%81%E7%AE%97%E6%B3%95) ，客户端与服务端需要一个安全的通道来共享密钥，并且两者时钟需要同步。

#### Golang服务端代码实现

##### 密钥

###### 生成密钥(基于时间)

步骤如下：

1. 时间戳，精确到微秒，除以1000，除以30（动态6位数字每30秒变化一次）

2. 对时间戳余数 hmac_sha1 编码

3. 然后 base32 encode 标准编码

4. 输出大写字符串，即秘钥

代码：

```golang

type GoogleAuth struct {
}

//  1. 获得时间戳，精确到微秒，除以1000，除以30（动态6位数字每30秒变化一次）
func (g *GoogleAuth) un() int64 {
   return time.Now().UnixNano() / 1000 / 30
}

// 2. 对时间戳余数 hmac_sha1 编码
func (g *GoogleAuth) hmacSha1(key, data []byte) []byte {
   h := hmac.New(sha1.New, key)
   if total := len(data); total > 0 {
      h.Write(data)
   }
   return h.Sum(nil)
}

// 3. 然后 base32 encode 标准编码
func (g *GoogleAuth) base32encode(src []byte) string {
   return base32.StdEncoding.EncodeToString(src)
}

// 获取秘钥，时间相关
func (g *GoogleAuth) GetSecret() string {
   var buf bytes.Buffer
   binary.Write(&buf, binary.BigEndian, g.un())
   // 4. 输出大写字符串，即秘钥
   return strings.ToUpper(g.base32encode(g.hmacSha1(buf.Bytes(), nil)))
}

```

###### 共享密钥

在用户登陆之后，设置开启双重认证时，通过二维码显示用户密钥，用户使用 Google Authenticator 客户端扫码，将此秘钥添加到列表中。

```golang

// 获取密钥二维码内容
func (g *GoogleAuth) GetQrcode(user, secret string) string {
   return fmt.Sprintf("otpauth://totp/%s?secret=%s", user, secret)
}

// 获取密钥二维码图片地址,这里是第三方二维码api
func (g *GoogleAuth) GetQrcodeUrl(user, secret string) string {
   qrcode := g.GetQrcode(user, secret)
   width := "200"
   height := "200"
   data := url.Values{}
   data.Set("data", qrcode)
   return "https://api.qrserver.com/v1/create-qr-code/?" + data.Encode() + "&size=" + width + "x" + height + "&ecc=M"
}

```

##### 实现与Google Authenticator一致的算法

算法太复杂，我也没弄明白，暂时直接贴代码吧：

```golang

// 获取动态码
func (g *GoogleAuth) GetCode(secret string) (string, error) {
   secretUpper := strings.ToUpper(secret)
   // 1. base32 解码
   secretKey, err := g.base32decode(secretUpper)
   if err != nil {
      return "", err
   }
   number := g.oneTimePassword(secretKey, g.toBytes(time.Now().Unix()/30))
   return fmt.Sprintf("%06d", number), nil
}

func (g *GoogleAuth) base32decode(s string) ([]byte, error) {
   return base32.StdEncoding.DecodeString(s)
}

func (g *GoogleAuth) toBytes(value int64) []byte {
   var result []byte
   mask := int64(0xFF)
   shifts := [8]uint16{56, 48, 40, 32, 24, 16, 8, 0}
   for _, shift := range shifts {
      result = append(result, byte((value>>shift)&mask))
   }
   return result
}

func (g *GoogleAuth) toUint32(bts []byte) uint32 {
   return (uint32(bts[0]) << 24) + (uint32(bts[1]) << 16) +
      (uint32(bts[2]) << 8) + uint32(bts[3])
}

func (g *GoogleAuth) oneTimePassword(key []byte, data []byte) uint32 {
   // 对密钥和时间余数进行 hmac_sha1 编码
   hash := g.hmacSha1(key, data)
   offset := hash[len(hash)-1] & 0x0F
   hashParts := hash[offset : offset+4]
   hashParts[0] = hashParts[0] & 0x7F
   number := g.toUint32(hashParts)
   return number % 1000000
}

```

## 维持上下文状态

