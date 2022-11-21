---
title: Rust接口自动化测试
author: 六道先生
date: 2022-11-21 11:20:31
categories:
- IT
tags:
- 接口测试
- Rust
- 自动化测试
+ description: Rust如此优秀和高效的编程语言，如果不能为测试工程师们所用，那就太可惜了。所以我们在此探讨下如何使用Rust进行API自动化测试。
---
# 前言
以Python如此低效的语言进行接口自动化测试，我一直不觉得是什么好的解决方案，但是一直也没有更好的替代方案。因为目前主流的Java语言，也不比Python高效多少，只是社区更加成熟，能解决的问题更多样。不过这个优点，其实看来也算不得什么特别的优势，尤其是在接口测试方面，自动化，讲究的是越简单，越安全，越高效越好。<br>
Rust的出现让我眼前一亮，其安全性和媲美C语言的执行效率让我觉得它应该就是一款合适自动化测试的编程语言。

# 实践探索
## Rust环境
Rust开发环境请详见[Rust学习笔记1]，这里就不再多言了。然后我们需要创建一个项目，用来编写我们的自动化测试用例。这里我们需要考虑，是创建`binary`项目，还是`library`项目。我们这个项目本身并不需要作为一个main来运行，所以这里我选择创建`library`项目。
```Shell
$ cargo new demo-api-test --lib
```
## 目录结构
项目有了，我们要考虑下如果组织我们的测试代码，我们在src下创建对应服务的目录，好比我们目前有`user`服务，`order`服务，那我们可以创建如下目录结构:
```
demo-api-test
  └─ src
      ├─ user
      └─ order
```
在`user`目录下，我们测试`user`服务中的所有接口，每个接口一个文件。`order`目录同理。

## 测试级别
在Rust中存在三种形式的测试，分别是：
* 单元测试
* 文档测试
* 集成测试

文档测试是通过写在注释中，来给出一些关于下面的函数的使用案例，个人感觉不是特别适合我们接口测试的场景。而单元测试和集成测试都可以。<br>
从使用的方便程度上来说，我们可以一个代码文件归纳一个接口，这个接口的测试也都写在这个文件中，所以采用单元测试的框架更为合理一些。当然这里指的是单接口的功能测试，如果是多接口的业务测试，可以采用集成测试框架，这样似乎更合理。也满足了Rust中这两个测试框架的要求。

## 实践
现在有一个demo服务，我们这里叫goapi，里面有一个sayhi接口，现在我们在项目`src`目录下创建`goapi`目录，然后在`lib.rs`中添加goapi模块：
```Rust
pub mod goapi;
```
在`goapi`目录下创建一个`mod.rs`和`sayhi.rs`。在`mod.rs`中添加sayhi模块：
```Rust
pub mod sayhi;
```
在`sayhi.rs`文件中，我们首先定义我们的API接口：
```Rust
use serde_derive::{Deserialize, Serialize};

/*
SayHiRequest is the request struct that is used to send to sayhi api.
 */
#[derive(Deserialize, Serialize)]
pub struct SayHiRequest {
    pub name: String,
    pub age: u32,
}

/*
SayHiResponse is the response struct that is used to receive the response of sayhi api.
 */
#[derive(Deserialize, Serialize)]
pub struct SayHiResponse {
    pub code: u32,
    pub message: String,
}

const BASE_URL: &str = "http://localhost:8088";
const SAYHI_PATH: &str = "/sayhi";
/*
sayhi is the function that is used to send request to sayhi api.
 */
pub async fn sayhi(req: SayHiRequest) -> Result<SayHiResponse, Box<dyn std::error::Error>> {
    let client = reqwest::Client::new();
    let url = format!("{}{}", BASE_URL, SAYHI_PATH);
    let resp = client.post(&url).json(&req).send().await;
    return match resp {
        Ok(resp) => {
            let resp = resp.json::<SayHiResponse>().await;
            match resp {
                Ok(resp) => Ok(resp),
                Err(err) => Err(Box::new(err)),
            }
        }
        Err(err) => Err(Box::new(err)),
    };
}
```
> 因为我们的接口为restful架构的，传输格式都为json，所以这里我们引入了`serde_json`模块来处理json。<br>
这里我们使用`reqwest`来处理http请求。

## 编写普通测试
我们先写一个最简单的接口测试：
```Rust
#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn test_sayhi() {
        let req = SayHiRequest {
            name: "John".to_string(),
            age: 18,
        };
        let resp = sayhi(req).await.unwrap();
        assert_eq!(resp.code, 1000);
        assert_eq!(resp.message, format!("Hi, {}, you are {} years old.", "John", 18));
    }
}
```

## 参数化
如果需要做参数化，那需要引入rstest模块，类似下面的写法：
```Rust
    #[rstest]
    #[case("Louis", 18, "Hi, Louis, you are 18 years old.")]
    #[case("Tom", 20, "Hi, Tom, you are 20 years old.")]
    #[tokio::test]
    async fn test_sayhi_with_rstest(#[case] name: &str, #[case] age: u32, #[case] message: &str) {
        let req = SayHiRequest {
            name: name.to_string(),
            age,
        };
        let resp = sayhi(req).await.unwrap();
        assert_eq!(resp.code, 1000);
        assert_eq!(resp.message, message);
    }
```

## 完整代码
* Cargo.toml文件
```toml
[package]
name = "demo-api-test"
version = "0.1.0"
edition = "2021"

[dependencies]
reqwest = { version = "0.11", features = ["json"] }
tokio = { version = "1", features = ["full"] }
serde_json = "1.0.88"
serde_derive = "1.0.147"
serde = "1.0.147"
rstest = "0.15.0"
```

* sayhi.rs文件
```Rust
use serde_derive::{Deserialize, Serialize};


/*
SayHiRequest is the request struct that is used to send to sayhi api.
 */
#[derive(Deserialize, Serialize)]
pub struct SayHiRequest {
    pub name: String,
    pub age: u32,
}

/*
SayHiResponse is the response struct that is used to receive the response of sayhi api.
 */
#[derive(Deserialize, Serialize)]
pub struct SayHiResponse {
    pub code: u32,
    pub message: String,
}

const BASE_URL: &str = "http://localhost:8088";
const SAYHI_PATH: &str = "/sayhi";
/*
sayhi is the function that is used to send request to sayhi api.
 */
pub async fn sayhi(req: SayHiRequest) -> Result<SayHiResponse, Box<dyn std::error::Error>> {
    let client = reqwest::Client::new();
    let url = format!("{}{}", BASE_URL, SAYHI_PATH);
    let resp = client.post(&url).json(&req).send().await;
    return match resp {
        Ok(resp) => {
            let resp = resp.json::<SayHiResponse>().await;
            match resp {
                Ok(resp) => Ok(resp),
                Err(err) => Err(Box::new(err)),
            }
        }
        Err(err) => Err(Box::new(err)),
    };
}

#[cfg(test)]
mod tests {
    use rstest::rstest;
    use super::*;

    #[tokio::test]
    async fn test_sayhi() {
        let req = SayHiRequest {
            name: "John".to_string(),
            age: 18,
        };
        let resp = sayhi(req).await.unwrap();
        assert_eq!(resp.code, 1000);
        assert_eq!(resp.message, format!("Hi, {}, you are {} years old.", "John", 18));
    }

    #[rstest]
    #[case("Louis", 18, "Hi, Louis, you are 18 years old.")]
    #[case("Tom", 20, "Hi, Tom, you are 20 years old.")]
    #[tokio::test]
    async fn test_sayhi_with_rstest(#[case] name: &str, #[case] age: u32, #[case] message: &str) {
        let req = SayHiRequest {
            name: name.to_string(),
            age,
        };
        let resp = sayhi(req).await.unwrap();
        assert_eq!(resp.code, 1000);
        assert_eq!(resp.message, message);
    }
}
```
## 执行
在命令行运行测试：
```Shell
$ cargo test
```
执行结果：
```Shell
$ cargo test
    Finished test [unoptimized + debuginfo] target(s) in 0.13s
     Running unittests src/lib.rs (target/debug/deps/demo_api_test-dd62dff229b446e1)

running 3 tests
test goapi::sayhi::tests::test_sayhi_with_rstest::case_1 ... ok
test goapi::sayhi::tests::test_sayhi ... ok
test goapi::sayhi::tests::test_sayhi_with_rstest::case_2 ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.01s
```
> Rust的特点，是所有的测试用例并发执行，3个用例一起执行，0.01s完成，效率相当高。

[Rust学习笔记1]: https://blog.testops.vip/2022/03/07/rust-1/