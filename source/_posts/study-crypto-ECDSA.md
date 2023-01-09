---
title: 签名算法之ECDSA
author: 六道先生
date: 2023-01-06 16:39:38
categories:
- IT
tags:
- Rust
- 签名算法
+ description: 聊聊签名算法，这里介绍ECDSA算法。
---
# 什么是ECDSA
需要分开来解释，首先是ECC（Elliptic Curve Cryptography）椭圆曲线密码算法，是一种密码学技术，用来安全地传输数据。

什么是曲线呢？Wolfram MathWorld给出了非常精准的定义：一条椭圆曲线就是一组被  $y^2=x^3+ax+b$ 定义的且满足 $4a^3+27b^2\ne0$ 的点集。  这个限定条件是为了保证曲线不包含奇点(singularities)。 $y^2=x^3+ax+b$ 这个方程称为椭圆曲线的维尔斯特拉斯标准形式（Weierstrass normal form）。

看不懂？不要紧，我们不研究数学，所以并不需要理解椭圆曲线算法，总之只需要知道是基于这种曲线，来运算加解密的过程。

再看DSA（Digital Signature Algorithm），那么ECDSA的意思就很清楚了—— 椭圆曲线签名算法，就是采用椭圆曲线算法，进行数字签名。

ECDSA和其他数字签名算法（如RSA）相比，有很多优势，包括所需的密钥更小，速度更快，安全性更高，等等。因此，ECDSA在很多领域中被广泛使用，包括区块链技术、互联网安全、数字身份验证等。

# 算法使用过程
在ECDSA算法中，私钥是一个固定长度字节的随机数，公钥是一个由两个大小为同样长度字节的数字组成的元组，表示椭圆曲线上的一个点。

算法包括两个部分：签名生成和签名验证。

在签名生成过程中，需要使用私钥和待签名的信息的摘要值生成数字签名。具体来说，签名生成过程包括以下步骤：

1. 使用摘要函数（例如 SHA-256）计算待签名信息的摘要值。
2. 使用私钥对摘要值进行加密，得到数字签名。

在签名验证过程中，需要使用公钥和待验证的数字签名验证信息的完整性。具体来说，签名验证过程包括以下步骤：

1. 使用公钥对数字签名进行解密，得到摘要值。
2. 使用摘要函数（例如 SHA-256）计算待验证信息的摘要值。
3. 比较摘要值和解密后的摘要值是否相等，如果相等则证明信息完整，否则信息不完整。

# 相关摘要算法
ECSDA需要搭配具体的Hash函数，目前最常用的是`SHA-256`，`SHA-384`，`SHA-512`，并且同时要有对应的椭圆曲线，分别是`P-256`，`P-384`，`P-512`，分别为它们取了不同的简称：

| 算法简称 | 对应算法 |
|---------|---------|
| ES256 | ECDSA using P-256 and SHA-256 |
| ES384 | ECDSA using P-384 and SHA-384 |
| ES512 | ECDSA using P-512 and SHA-512 |

# 代码实现
一般编程语言都有比较成熟的`ECC`和对应的椭圆函数的实现，合起来就能实现对应的算法了。不过`Rust`目前还没有`P-512`的成熟`crates`，我们在此处就不演示`ES512`的实现了。

首先引用依赖：
```toml
# Cargo.toml
[dependencies]
sha2 = { version = "0.10.6", default-features = false, features = ["oid"] }
ecdsa = "0.14.8"
p256 = { version = "0.11.1", default-features = false, features = ["ecdsa"] }
p384 = { version = "0.11.2", default-features = false, features = ["ecdsa"] }
```

接着就可以在代码中实践了，我在此处分别实践了`ES256`、`ES384`两种，将`Hello World`字符串进行签名运算，使用随机函数生成ECDSA算法所需的字节数组来生成ECC的私钥，然后输出16进制形式的签名字符串，并使用`verify`函数进行验证：
```rust
use p256::ecdsa::{SigningKey as SigningKey256, signature::Signer as Signer256};
use p384::ecdsa::SigningKey as SigningKey384;

fn main()  {
    // ES256签名
    // 准备一个线程安全的随机数生成器
    let mut rng = rand::thread_rng();
    // 使用随机函数生成ECDSA的私钥，然后使用私钥生成签名，hash函数使用的是sha256
    let signing_key = SigningKey256::random(&mut rng); 
    // 准备需要签名的数据
    let message = b"Hello World";
    // 生成签名
    let signature = signing_key.sign(message);
    println!("ES256 string: {}", signature);

    // 用一个代码块来写验证的代码，这样可以避免变量名冲突
    {
        use p256::ecdsa::{VerifyingKey, signature::Verifier};
        // 生成验证签名的公钥
        let verifying_key = VerifyingKey::from(&signing_key); 
        assert!(verifying_key.verify(message, &signature).is_ok());
    }
    
    // ES384签名
    // 使用随机函数生成ECDSA的私钥，然后使用私钥生成签名，hash函数使用的是sha384
    let signing_key = SigningKey384::random(&mut rng); 
    // 准备需要签名的数据
    let message = b"Hello World";
    // 生成签名
    let signature = signing_key.sign(message);
    println!("ES384 string: {}", signature);
    // 用一个代码块来写验证的代码，这样可以避免变量名冲突
    {
        use p384::ecdsa::{VerifyingKey, signature::Verifier};
        // 生成验证签名的公钥
        let verifying_key = VerifyingKey::from(&signing_key); 
        assert!(verifying_key.verify(message, &signature).is_ok());
    }
    
}
```

看下输出的结果：
```shell
   Compiling study-crypto v0.1.0 (/Users/luyiyi/rustProj/study-crypto)
    Finished dev [unoptimized + debuginfo] target(s) in 3.65s
     Running `target/debug/examples/study_es`
ES256 string: 88D470F437DF2545FA72A83D106FDEF1A7D36E22B84A0422BAD278ADD8ED3233D6155024B2550ADCF41965F9A8EDF3567C92BDECAF53D278D91FF0FACD5FBA71
ES384 string: 36EEFB8FF045E4B9C58805A8B9ED15280909F3F65C2AAA45FFEDB522774A00D33A62FDF660C43644F69EE5EAB34D54ABB706368B234977DC6994939FCCF4481E0DA0789F824B453538106C662DEABB2C71F3BAD685FF7F6A5F7309E0A98657B3
```

源代码请参考[GitHub仓库]

[GitHub仓库]: https://github.com/jimmyseraph/study-crypto