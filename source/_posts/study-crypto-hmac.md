---
title: 签名算法之HMAC
author: 六道先生
date: 2023-01-03 20:01:08
categories:
- IT
tags:
- Rust
- 签名算法
+ description: 聊聊签名算法，这里介绍HMAC算法。
---

# 什么是HMAC
HMAC（Hash-based Message Authentication Code）是一种消息认证码算法，它使用一个密钥和哈希函数来生成一个消息的认证标记。HMAC通常用于认证消息的完整性和身份验证。所以，我们一般不认为这种算法是加密算法，因为它是不可逆的过程，也就是说只有“加密”，没有“解密”。

# HMAC算法
整个计算过程如下：

1. 在密钥K后面添加0 或者 对密钥K用Hash函数进行处理 来创建一个字长为B的字符串。(例如，如果K的字长是20字节，B＝64字节，则K后会加入44个零字节0x00；如果K的字长是120字节，B＝64字节，则会用Hash函数作用于K后产生64字节的字符串)

2. 将上一步生成的B字长的字符串与ipad做异或运算。

3. 将需要处理的数据text填充至第二步的结果字符串后面。

4. 用Hash函数作用于第三步生成的数据。

5. 将第一步生成的B字长字符串与opad做异或运算。

6. 再将第四步的结果填充进第五步的结果字符串后面。

7. 用Hash函数作用于第六步生成的数据流，输出最终结果。

> ipad和opad是定长的常量，通常ipad的值为`0x36`，opad的值为`0x5c`，并将这两个数重复，扩充到固定字节长度（和B一样长，比如64个字节）

# HMAC算法的具体实现
HMAC需要搭配具体的Hash函数，目前最常用的是`SHA-256`，`SHA-384`，`SHA-512`，分别为它们取了不同的简称：

| 算法简称 | 对应算法 |
|---------|---------|
| HS256 | HMAC using SHA-256 |
| HS384 | HMAC using SHA-384 |
| HS512 | HMAC using SHA-512 |

# 代码实现

代码实现上，一般语言都有比较成熟的`HMAC`和`SHA`的实现，合起来就能实现对应的算法了。首先引用依赖：
```toml
# Cargo.toml
[dependencies]
sha2 = "0.10.6"
hmac = "0.12.1"
hex = "0.4.3"
```

接着就可以在代码中实践了，我在此处分别实践了`HS256`、`HS384`、`HS512`三种，将`Hello World`字符串进行运算，使用的`key`为一个简单的字符串`my secret and secure key`，最后输出16进制形式的字符串：
```rust
use sha2::{Sha256, Sha384, Sha512};
use hmac::{Hmac, Mac};
use hex;

// 定义相关别名
type HmacSha256 = Hmac<Sha256>;
type HmacSha384 = Hmac<Sha384>;
type HmacSha512 = Hmac<Sha512>;

fn main() {

    // HMAC-SHA256
    let mut mac = HmacSha256::new_from_slice(b"my secret and secure key")
        .expect("HMAC can take key of any size");
    mac.update(b"Hello World");
    let result = mac.finalize();
    let hs256_string = hex::encode(result.into_bytes());
    println!("HS256 String is: {}", hs256_string);

    // HMAC-SHA384
    let mut mac = HmacSha384::new_from_slice(b"my secret and secure key")
        .expect("HMAC can take key of any size");
    mac.update(b"Hello World");
    let result = mac.finalize();
    let hs384_string = hex::encode(result.into_bytes());
    println!("HS384 String is: {}", hs384_string);

    // HMAC-SHA512
    let mut mac = HmacSha512::new_from_slice(b"my secret and secure key")
        .expect("HMAC can take key of any size");
    mac.update(b"Hello World");
    let result = mac.finalize();
    let hs512_string = hex::encode(result.into_bytes());
    println!("HS512 String is: {}", hs512_string);
}
```
看下输出的结果：
```shell
   Compiling study-crypto v0.1.0 (/Users/luyiyi/rustProj/study-crypto)
    Finished dev [unoptimized + debuginfo] target(s) in 3.58s
     Running `target/debug/examples/study_hmac`
HS256 String is: 5731eb2136aeb2c69cc4261e4f113538fa772b9056482232709051c981c06979
HS384 String is: c56548daa49c437fb6fc2f052e6323473e06cb33c4ce7deb78c7aa92d02aa8e72ea4f031ef803a08361178d97dd1e8e9
HS512 String is: d01268077c496aafda4c910e61583634e195f12ef8faef220d3cb1ae8395b835ebcf1b297fbb22c7fdb52679096b9ed11f4e3316fc5f183977963c6598ac421f
```

源代码请参考[GitHub仓库]

[GitHub仓库]: https://github.com/jimmyseraph/study-crypto