---
title: 签名算法之RSASSA-PKCS1-v1_5
author: 六道先生
date: 2023-01-05 09:39:21
categories:
- IT
tags:
- Rust
- 签名算法
+ description: 聊聊签名算法，这里介绍RSASSA-PKCS1-v1_5算法。
---
# 什么是RSASSA-PKCS1-v1_5
分开来解释，首先是RSA，RSA加密算法是一种非对称加密算法，由罗纳德·李维斯特（Ron Rivest）、阿迪·萨莫尔（Adi Shamir）和伦纳德·阿德曼（Leonard Adleman）在1977年一起提出的，RSA就是他们三人姓氏开头字母拼在一起组成的。

RSASSA（RSA Signature Scheme with appendix）RSA填充签名，而PKCS1（Public-Key Cryptography Standards）是一组公钥加密标准，由美国密码学研究所（RSA Laboratories）制定。PKCS1-v1_5是PKCS1的一个版本，它定义了使用RSA算法生成数字签名的格式和算法。

具体来说，PKCS1-v1_5定义了将摘要值填充为特定格式（如填充"00 01 FF FF FF … FF 00"）后使用RSA私钥进行加密生成数字签名的算法流程。

所以，RSASSA-PKCS1-v1_5算法就是使用RSA算法和PKCS1-v1_5标准生成数字签名的算法。

# 算法使用过程
该算法由两个部分组成：签名生成和签名验证。

在签名生成过程中，需要使用RSA私钥和待签名的信息的摘要值生成数字签名。这个过程包括以下步骤：

1. 使用摘要函数计算待签名信息的摘要值。
2. 对摘要值进行填充，使其满足一定的格式要求。
3. 使用RSA私钥对填充后的摘要值进行加密，生成数字签名。

在签名验证过程中，需要使用RSA公钥和待验证的信息的摘要值来验证数字签名的有效性。这个过程包括以下步骤：

1. 使用摘要函数计算待验证信息的摘要值。
2. 对摘要值进行填充，使其满足与签名生成时相同的格式要求。
3. 使用RSA公钥对数字签名进行解密，得到填充后的摘要值。

将解密后的填充后的摘要值与步骤1中计算得到的摘要值进行比较，如果相同，则证明数字签名是有效的。

# 相关摘要算法
RSASSA-PKCS1-v1_5需要搭配具体的Hash函数，目前最常用的是`SHA-256`，`SHA-384`，`SHA-512`，分别为它们取了不同的简称：

| 算法简称 | 对应算法 |
|---------|---------|
| RS256 | RSASSA-PKCS1-v1_5 using SHA-256 |
| RS384 | RSASSA-PKCS1-v1_5 using SHA-384 |
| RS512 | RSASSA-PKCS1-v1_5 using SHA-512 |

# 代码实现
一般编程语言都有比较成熟的`RSA`和`SHA`的实现，合起来就能实现对应的算法了。首先引用依赖：
```toml
# Cargo.toml
[dependencies]
sha2 = { version = "0.10.6", default-features = false, features = ["oid"] }
hex = "0.4.3"
rsa = "0.7.2"
```

接着就可以在代码中实践了，我在此处分别实践了`RS256`、`RS384`、`RS512`三种，将`Hello World`字符串进行签名运算，使用随机函数生成RSA算法所需的字节数组来生成RSA的私钥，然后输出16进制形式的签名字符串，并使用`verify`函数进行验证：
```rust
use rsa::RsaPrivateKey;
use rsa::pkcs1v15::{SigningKey, VerifyingKey};
use rsa::signature::{RandomizedSigner, Signature, Verifier};
use sha2::{Sha256, Sha384, Sha512};
use hex;


fn main() {
    // 准备一个线程安全的随机数生成器
    let mut rng = rand::thread_rng();

    // 定义私钥长度
    let bits = 2048;
    // 生成RSA私钥
    let private_key = RsaPrivateKey::new(&mut rng, bits).expect("failed to generate a key");

    // 需要进行签名的数据
    let data = b"Hello World";

    // RS256签名
    let signing_key: SigningKey<Sha256> = SigningKey::<Sha256>::new_with_prefix(private_key.clone());
    let verifying_key: VerifyingKey<Sha256> = (&signing_key).into();
    let signature = signing_key.sign_with_rng(&mut rng, data);
    let signature_string = hex::encode(signature.as_bytes());
    println!("RS256 String is: {}", signature_string);

    // RS256签名验证
    verifying_key.verify(data, &signature).expect("failed to verify");

    // RS384签名
    let signing_key: SigningKey<Sha384> = SigningKey::<Sha384>::new_with_prefix(private_key.clone());
    let verifying_key: VerifyingKey<Sha384> = (&signing_key).into();
    let signature = signing_key.sign_with_rng(&mut rng, data);
    let signature_string = hex::encode(signature.as_bytes());
    println!("RS384 String is: {}", signature_string);

    // RS384签名验证
    verifying_key.verify(data, &signature).expect("failed to verify");

    // RS512签名
    let signing_key: SigningKey<Sha512> = SigningKey::<Sha512>::new_with_prefix(private_key.clone());
    let verifying_key: VerifyingKey<Sha512> = (&signing_key).into();
    let signature = signing_key.sign_with_rng(&mut rng, data);
    let signature_string = hex::encode(signature.as_bytes());
    println!("RS512 String is: {}", signature_string);

    // RS512签名验证
    verifying_key.verify(data, &signature).expect("failed to verify");

}
```

看下输出的结果：
```shell
    Finished dev [unoptimized + debuginfo] target(s) in 0.18s
     Running `target/debug/examples/study_rs`
RS256 String is: 51440042bb6bb48d378edb8894667a08e96c640c0999c0e7074bd48fe2a80b71f2bd757ff4e50f8da9274d75ded0fbe8cafc840a6d0b7e58d9be4e5c9a14059b564993e0294a0de5c4edde8562e386eb8d9cec7fee82ae3c823c51943b65e665b458b5ba43c9dc7d9e5f529d90fe9c849624729b521f5e040c0e3b5ff98cba831ce2595225170538b496d3d103cf5bacdf9248db27fb93ad70c769ca695c86a745ee8b5af3c4befe33a1699658291092ca46f6f2d8f2ce23fac09b443fc77d6495a0e8418e90fa710b2aee1534d5797870ab6a981a7b385cbe4202bbbefdcea00ada66a36d365cc4165d5c731cab49ec13e86f0134366617003a1c3af1f374fb

RS384 String is: 7ad2e5afc1ea2222d2728bcba4f33f51a081f454dce30f5f7e9ab88ee3fdc497b1c0e33a6fddca38b3693ce41f50a2074562e19410c1e43a61485ad1f224d635d350cf51dc44d2e5085d3848188258e4133a7e41ff0aa6cac1a12ec3e38f2720616abb3936be898d6856c9328e49a078e0514b0d11514b5782149621a76d4bb22418de5fe61026b52277eecd1035bc6298ad72afbb122a6357af2216f5859640662b307ddce31a70da67adfa8c03bd8d68cebca6a7f76e1c302d68f2e0d396fdc5f442e11cc42e848f5fabbcc30ab0fce6d5ffffa8bc21fdd0abeb379a43ddf5e930e0ed047dc64b5eef9c9195d40cb92d3e93edf92d4949e6210203144ae3fd

RS512 String is: 9d6095e6a7391bee093e0ee6841529362e7ac4f3f27ebb952f97b4fa520bd6b6c448bd4c678f21334a6f01374d0f0e7eecc1bc6ac0f12e6493cc3756fcf57a376dd05cc3aa27336808882f2db448fc3a63454dd4398eac8d21549e5b1040b5604acfb8ebb4ec27f4492a56526548c4aba0a9d552cd8e79422e5f600ad812e4dcc98e899c13cfedfdac0ad094cd92b2bedbfcf8861f29742a035e5c63d57f8b11ee5a464e7afb10121b23ff4078df4f6a910e1828203b8ca4d59b85068693274495619e552b5bc6fbb52905be1318014e9aa5b0b85107f7fc96dcd2eec7c5427b087317493a91b30bbd024254f0bb741b43e964d510dbca66b18655755d2dfb23
```

源代码请参考[GitHub仓库]

[GitHub仓库]: https://github.com/jimmyseraph/study-crypto