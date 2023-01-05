---
title: 签名算法之RSASSA-PSS
author: 六道先生
date: 2023-01-05 16:28:40
categories:
- IT
tags:
- Rust
- 签名算法
+ description: 聊聊签名算法，这里介绍RSASSA-PSS算法。
---
# 什么是RSASSA-PSS
分开来解释，首先是RSASSA请参考前面介绍`RSASSA-PKCS1-v1_5`算法的文章，这里不再重复了。

PSS（Probabilistic Signature Scheme）概率签名算法，它使用随机数来生成签名，并通过使用一些掩码（mask）和哈希函数来保证签名的安全性。计算掩码的算法通常是MGF1（Mask Generation Function 1）。

所以，RSASSA-PSS算法就是使用RSA算法结合PSS概率算法生成数字签名的算法。

# 算法使用过程
该算法由两个部分组成：签名生成和签名验证。

在签名生成过程中，需要使用 RSA 私钥和待签名的信息的摘要值生成数字签名。具体来说，签名生成过程包括以下步骤：

1. 使用摘要函数（例如 SHA-256）计算待签名信息的摘要值。
2. 对摘要值进行填充，使其满足一定的格式要求。
3. 使用 MGF1 算法和摘要函数（例如 SHA-256）生成掩码。
4. 对填充后的摘要值进行异或操作，得到签名信息。
5. 使用 RSA 私钥对签名信息进行加密，得到最终的数字签名。

在签名验证过程中，需要使用 RSA 公钥和待验证的数字签名验证信息的完整性。具体来说，签名验证过程包括以下步骤：

1. 使用 RSA 公钥对数字签名进行解密，得到签名信息。
2. 使用 MGF1 算法和摘要函数（例如 SHA-256）生成掩码。
3. 对签名信息进行异或操作，得到填充后的摘要值。
4. 删除填充后的摘要值中的填充内容，得到原摘要值。
5. 使用摘要函数（例如 SHA-256）计算待验证信息的摘要值。
6. 比较原摘要值和计算得到的摘要值是否相等，如果相等则证明信息完整，否则信息不完整。

# 相关摘要算法
RSASSA-PSS需要搭配具体的Hash函数，目前最常用的是`SHA-256`，`SHA-384`，`SHA-512`，分别为它们取了不同的简称：

| 算法简称 | 对应算法 |
|---------|---------|
| PS256 | RSASSA-PSS using SHA-256 and MGF1 with SHA-256 |
| PS384 | RSASSA-PSS using SHA-384 and MGF1 with SHA-384 |
| PS512 | RSASSA-PSS using SHA-512 and MGF1 with SHA-512 |

# 代码实现
一般编程语言都有比较成熟的`RSA`和`SHA`的实现，合起来就能实现对应的算法了。首先引用依赖：
```toml
# Cargo.toml
[dependencies]
sha2 = { version = "0.10.6", default-features = false, features = ["oid"] }
hex = "0.4.3"
rsa = "0.7.2"
```

接着就可以在代码中实践了，我在此处分别实践了`PS256`、`PS384`、`PS512`三种，将`Hello World`字符串进行签名运算，使用随机函数生成RSA算法所需的字节数组来生成RSA的私钥，然后输出16进制形式的签名字符串，并使用`verify`函数进行验证：
```rust
use rsa::RsaPrivateKey;
use rsa::pss::{BlindedSigningKey, VerifyingKey};
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

    // PS256签名
    let signing_key: BlindedSigningKey<Sha256> = BlindedSigningKey::<Sha256>::new(private_key.clone());
    let verifying_key: VerifyingKey<Sha256> = (&signing_key).into();
    let signature = signing_key.sign_with_rng(&mut rng, data);
    let signature_string = hex::encode(signature.as_bytes());
    println!("PS256 String is: {}", signature_string);

    // PS256签名验证
    verifying_key.verify(data, &signature).expect("failed to verify");

    // PS384签名
    let signing_key: BlindedSigningKey<Sha384> = BlindedSigningKey::<Sha384>::new(private_key.clone());
    let verifying_key: VerifyingKey<Sha384> = (&signing_key).into();
    let signature = signing_key.sign_with_rng(&mut rng, data);
    let signature_string = hex::encode(signature.as_bytes());
    println!("PS384 String is: {}", signature_string);

    // PS384签名验证
    verifying_key.verify(data, &signature).expect("failed to verify");

    // PS512签名
    let signing_key: BlindedSigningKey<Sha512> = BlindedSigningKey::<Sha512>::new(private_key.clone());
    let verifying_key: VerifyingKey<Sha512> = (&signing_key).into();
    let signature = signing_key.sign_with_rng(&mut rng, data);
    let signature_string = hex::encode(signature.as_bytes());
    println!("PS512 String is: {}", signature_string);

    // PS512签名验证
    verifying_key.verify(data, &signature).expect("failed to verify");

}
```

看下输出的结果：
```shell
   Compiling study-crypto v0.1.0 (/Users/luyiyi/rustProj/study-crypto)
    Finished dev [unoptimized + debuginfo] target(s) in 0.86s
     Running `target/debug/examples/study_ps`
PS256 String is: a0f8f05a7271344d753df8e29cde525e10a7e7adf5a458e0d57b2107a079c3fcba091476b4740504b8acbcd4632158540be954e11c0c968b23708301e493058381d06f7154baae811f1d2d0208e21c64c0e0553b95ad779f17906ed2bd996aa3f5a956a865fe5a732bd0eeba7926c5f194ac7236ad5f15004f23414fb9ad379652801235a0fe77157a8a735927500258c1ce576d0c67b41931e72699fcd6273a04cbcd96ad7c232df5ee8515a725d9174e8d010a93bf11759e12a4c5b3ae7556239b0a8d35da53b1503283b28b6318226faa97a5e8385fa05ff28c33725b7786a18a2d9289adb14923d4023d846cdeb3b890ee40548857a1cdda7fc45b1a70f4
PS384 String is: 87c879e5410b87cb96c19b33f92f662ff726c012a06a78478419d08badab3aa76eb80cbc873113473bd2a842625220cc2fe158582dd46b1fbef2fb49e6cd41e06e3f20fdeeb8c25438ee280b9665c69d3770f4d046ec424e38c992769b011dbc306124adf2f837a3324ffd077686109f3784f9eb380cc81f7db68c3b3fb50e0987f17c60bee868bf9850e7da9a26238a4f880d658622d38af42effc59f5808a65bb452a45942124d6b0002227ea5c2263790ee06c04311b00579cb8e6906c71a94ea3ff5b9becc98e7238bc0dc0f53892a4f406ddc15b52b3b203de8e3b0832bdbcc1c0dd3c617b05be9536150bce15429fdef6cc1674ebdc05656e25f8093ed
PS512 String is: 029611004198873abf2eb4a94a71de290292672e1acb813bad0db315e730e1a8c8ae44f13e02f6bbb4c6d2d2b923ab5180dfe638b4650d9276968b58dc9de483b05b4531228f96f1f2e1452e1eae14e9a2a85c5783d052906080ac706c06afe95aa88ac73647cb4ed48acdf7e2bcd6b521e32b97725ccdf3857b02e3b22a9b8bacd6f60ab0e4a60efaf4084a60548331c8c50f31cfcc972503c41cc3bb081d314fba7a37744b2d0bbcf0b8190fb5b52b18c0126b246c1760470c387d8440f0ce28b81b9b5acda2d04986d5be728663842a433e7aa458a67c15eed6e92eae6315c29555407b24228ea2eecf492fe282e17ebdb076ab9564c477597e9b6700450b
```

源代码请参考[GitHub仓库]

[GitHub仓库]: https://github.com/jimmyseraph/study-crypto