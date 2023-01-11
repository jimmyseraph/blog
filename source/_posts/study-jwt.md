---
title: 全面详解JWT
author: 六道先生
date: 2023-01-09 21:13:48
categories:
- IT
tags:
- Rust
- JWT
+ description: 全面解析什么是JWT，以及怎么实现。
---
# 什么是JWT
JWT（JSON Web Token），JSON格式的网络凭证。

Token这个词现在应该没有人不知道，在前后端分离的时代下，后端接口通常采用Token的方式来验证客户端是否有权限调用。那这个Token怎么生成呢？它需要保证每个用户的Token独一无二，并且第三方截获后无法修改。最容易想到的，自然就是采用不可逆的摘要算法来把用户特征信息进行摘要，生成唯一的一串字符作为Token。这个想法没问题，而且在现实中也确实有很多应用这么做。不过既然在互联网上，就需要有一些统一的标准，确保多方的理解能够一致。所以根据一份开放标准([RFC 7519])，定义了一个格式紧凑并且自我包含的JSON格式数据，用于作为网络间的Token。

# JWT的格式
JWT是由`.`分隔开的三部分组成的，这三部分分别是：
+ 头（Header）
+ 内容（Payload）
+ 签名（Signature）

所以一个JWT格式看起来像是这样的：`xxxxx.yyyyy.zzzzz`

接下来我们分别来研究这三部分。

## 头（Header）
头通常有两部分组成：
1. Token的类型，通常就是`JWT`。
2. 使用的签名算法，比如`HS256`等。

举个例子看下，某一个典型的头：
```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```
然后，对这个JSON格式的数据进行`Base64URL`编码，这就组成了`头`的内容了。

## 内容（Payload）
第二部分是内容（Payload），它包含了一组申明（Claims），申明（Claims）是关于数据实体以及附加信息的语句。

`claims`有三种类型：
+ **注册Claims**：这是一组预先定义好的`claims`，这一组`claims`不是强制要求的，但是建议比较有意义，建议加上。
> `注册claims`有如下这些：<br>
> **iss(issuer)**：发布者<br>
> **sub(subject)**：主题<br>
> **aud(audience)**：接收者<br>
> **exp(expiration time)**：过期时间，数字格式<br>
> **nbf(not before)**：不早于某一时间，如果当前时间早于这里设置的值则不处理token，数字格式<br>
> **iat(issued at)**：发布时间，数字格式<br>
> **jti(JWT ID)**：JWT的ID<br>

+ **公共Claims**：这部分是可以由使用者自己定义的，但是为了避免命名冲突，需要预先在[IANA JSON Web Token Registry]定义。

+  **私有Claims**：不包含在`注册claims`和`公共Claims`中的其他需要交换的信息，可以定义在`私有Claims`中。

来看一个Payload的例子：
```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "admin": true
}
```
然后，对这个JSON格式的数据进行`Base64URL`编码，这就组成了`Payload`的内容了。

## 签名（Signature）
要生成签名，首先要把`header`和`payload`两部分合起来，以前面的的为例，`header`部分经过`base64url`编码后是:
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9
```
`payload`部分经过`base64url`编码后是:
```
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiYWRtaW4iOnRydWV9
```
这两部分合起来，中间用`.`分开，就成了以下的字符串：
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiYWRtaW4iOnRydWV9
```
对这个字符串进行签名，生成一组指定长度的二进制数，然后将这一组二进制数进行`base64url`编码，得到的字符串就是签名了。

JWT所支持的签名算法，主要有以下这么几种：

| 签名算法名称 | 数字签名和MAC算法 |
|------------|----------------|
| HS256 | HMAC using SHA-256 |
| HS384 | HMAC using SHA-384 |
| HS512 | HMAC using SHA-512 |
| RS256 | RSASSA-PKCS1-v1_5 using SHA-256 |
| RS384 | RSASSA-PKCS1-v1_5 using SHA-384 |
| RS512 | RSASSA-PKCS1-v1_5 using SHA-512 |
| PS256 | RSASSA-PSS using SHA-256 and MGF1 with SHA-256 |
| PS384 | RSASSA-PSS using SHA-384 and MGF1 with SHA-384 |
| PS512 | RSASSA-PSS using SHA-512 and MGF1 with SHA-512 |
| ES256 | ECDSA using P-256 and SHA-256 |
| ES384 | ECDSA using P-384 and SHA-384 |
| ES512 | ECDSA using P-512 and SHA-512 |

> 这些算法可以参考前面的介绍：[HS签名算法传送门]，[RS签名算法传送门]，[PS签名算法传送门]，[ES签名算法传送门]。

## 合起来
把前面的三部分合起来，那就是一个标准的JWT了，这里以`HS256`签名算法为例，得到的token为：
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiYWRtaW4iOnRydWV9.-phM0aLbmdaodj7A9Wpi959bNRYVgBySu6LeCKXMTkM
```

> 注意了，JWT的`header`和`payload`两部分并没有加密，仅仅是进行了`base64url`编码，所以一些敏感信息不可以放在JWT中。最后的签名，只是保证`header`和`payload`两部分不可被修改，并不能保证数据安全。

# 代码实现
我们可以自己来根据算法，实现JWT，签名算法介绍过了，`Base64URL`算法也有现成的`crate`可以用，那就没什么阻碍了，实现起来很简单。当然，为了避免麻烦，也可以使用现成的JWT的实现库来生成。

在JWT官网上有推荐的库，各种语言都有，[仓库传送门]。

每一个库，都说明了它实现了哪些签名算法，是否支持`注册claims`的校验。打绿色勾子的表示已经实现，红色叉的表示未实现。这里我选择了名字叫`jsonwebtoken`的一个rust库。

首先添加必要的依赖：
```toml
[dependencies]
jsonwebtoken = "8.2.0" # JWT库
serde = {version = "1.0.147", features = ["derive"] } # 序列化和反序列化库
chrono = "0.4.23"   # 时间库
```
我们这里决定使用`EC384`签名算法，准备一对`ECC`算法的私钥和公钥文件，可以用`openssl`命令来生成：
```shell
# 生成私钥
openssl ecparam -genkey -noout -name secp384r1 | openssl pkcs8 -topk8 -nocrypt -out ec-private.pem

# 使用私钥文件来生成公钥文件
openssl ec -in ec-private.pem -pubout -out ec-public.pem 
```

写代码：
```rust
use serde::{Serialize, Deserialize};
use jsonwebtoken::{encode, decode, Header, Algorithm, Validation, EncodingKey, DecodingKey, errors::ErrorKind};
use chrono::{prelude::{DateTime, Local}, Duration};

// 准备Claim结构体，用于存放JWT的payload部分
#[derive(Debug, Serialize, Deserialize)]
struct Claims {
    // registered claims
    iss: String,
    sub: String,
    exp: usize,

    // public claims
    website: String,

    // private claims
    user_id: String,
}

fn main() {
    // 定义header部分的“alg”参数，也就是算法名称
    let header = Header::new(Algorithm::ES384);

    // 准备过期时间
    let exp: DateTime<Local> = Local::now();
    exp.checked_add_signed(Duration::seconds(60));

    // 准备claims数据
    let claims = Claims {
        iss: "Louis".to_string(),
        sub: "testops.vip".to_string(),
        exp: exp.timestamp() as usize,
        website: "https://testops.vip".to_string(),
        user_id: "123456".to_string(),
    };

    // 从pem文件中读取私钥
    let key = EncodingKey::from_ec_pem(include_bytes!("../ec-private.pem")).unwrap();

    // 将header和claims进行签名，组合后生成token
    let token = encode(&header, &claims, &key).unwrap();

    println!("token: {}", token);

    // 校验token
    let mut validation = Validation::new(Algorithm::ES384);
    // 定义需要校验payload中的iss
    validation.set_issuer(&["Louis"]);
    // 定义需要校验payload中的sub
    validation.sub = Some("testops.vip".to_string());
    // 从公钥文件生成解码的密钥
    let key = DecodingKey::from_ec_pem(include_bytes!("../ec-public.pem")).unwrap();
    // 进行校验，并返回Payload的Claim结构体
    let token_data = match decode::<Claims>(&token, &key, &validation) {
        Ok(c) => c, // 校验通过
        Err(err) => match *err.kind() { //校验不通过
            ErrorKind::InvalidToken => panic!("Token 不存在"), 
            ErrorKind::InvalidIssuer => panic!("Issuer 不存在"),
            ErrorKind::InvalidSubject => panic!("Subject 不存在"),
            _ => panic!("其他错误: {:?}", err),
        },
    };
    println!("{:?}", token_data.claims);
    println!("{:?}", token_data.header);
}
```
看下输出：
```shell
   Compiling study-crypto v0.1.0 (/rustProj/study-crypto)
    Finished dev [unoptimized + debuginfo] target(s) in 0.58s
     Running `target/debug/examples/study_jwt`
token: eyJ0eXAiOiJKV1QiLCJhbGciOiJFUzM4NCJ9.eyJpc3MiOiJMb3VpcyIsInN1YiI6InRlc3RvcHMudmlwIiwiZXhwIjoxNjczNDA5Mjc4LCJ3ZWJzaXRlIjoiaHR0cHM6Ly90ZXN0b3BzLnZpcCIsInVzZXJfaWQiOiIxMjM0NTYifQ.MeK4lANKRN1EgrumcKMgNTFnhxIL_cmVFC_PVGW_Vn5Xfb7wXYEq7TzM9Ib062Kxt_nF3YKnpexJGscxwBK-xh2zsdg8VEfYK0dZ9BUHYZbeCs_WeT1QVxRWTZgz1i33
Claims { iss: "Louis", sub: "testops.vip", exp: 1673409278, website: "https://testops.vip", user_id: "123456" }
Header { typ: Some("JWT"), alg: ES384, cty: None, jku: None, jwk: None, kid: None, x5u: None, x5c: None, x5t: None, x5t_s256: None }
```
其中的
`
eyJ0eXAiOiJKV1QiLCJhbGciOiJFUzM4NCJ9.eyJpc3MiOiJMb3VpcyIsInN1YiI6InRlc3RvcHMudmlwIiwiZXhwIjoxNjczNDA5Mjc4LCJ3ZWJzaXRlIjoiaHR0cHM6Ly90ZXN0b3BzLnZpcCIsInVzZXJfaWQiOiIxMjM0NTYifQ.MeK4lANKRN1EgrumcKMgNTFnhxIL_cmVFC_PVGW_Vn5Xfb7wXYEq7TzM9Ib062Kxt_nF3YKnpexJGscxwBK-xh2zsdg8VEfYK0dZ9BUHYZbeCs_WeT1QVxRWTZgz1i33
`
就是我们这个例子生成的JWT了。

源代码请参考[GitHub仓库]

[GitHub仓库]: https://github.com/jimmyseraph/study-crypto

[RFC 7519]: https://tools.ietf.org/html/rfc7519

[IANA JSON Web Token Registry]: https://www.iana.org/assignments/jwt/jwt.xhtml

[HS签名算法传送门]: https://blog.testops.vip/2023/01/03/study-crypto-hmac/

[RS签名算法传送门]: https://blog.testops.vip/2023/01/05/study-crypto-RSASSA-PKCS1-v1-5/

[PS签名算法传送门]: https://blog.testops.vip/2023/01/05/study-crypto-RSASSA-PSS/

[ES签名算法传送门]: https://blog.testops.vip/2023/01/06/study-crypto-ECDSA/

[仓库传送门]: https://jwt.io/libraries