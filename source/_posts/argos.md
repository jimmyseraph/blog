---
title: 阿耳戈斯（Argos）Web后端框架
author: 六道先生
date: 2023-09-13 16:52:54
categories:
- IT
tags:
- Rust
- 框架
+ description: 用Rust来开发Web后端可行吗？来看看Argos框架。
---

# Argos框架

阿耳戈斯（Argos）这个名字来自于希腊神话中的百眼巨人，传说他有一百只眼睛，即使睡觉时也总有一些还睁着，因此得到别名“潘诺普忒斯”（Πανόπτης，“总在看着的”）。

当然，这里是rust语言开发的一款Web后端框架。

## 为何用Argos框架

rust语言的Web后端框架不少，有底层的`hyper`框架，有上层的`warp`框架等等，不过在使用上都有诸多不方便的地方。

`Argos`框架更易使用，通过`#[route]`自动注册路由，通过`#[filter]`自动注册过滤器，这两点就要比`warp`框架方便了太多。对于需要快速创建一个API服务的，`Argos`框架是非常合适的一个选择。

## 基本用法

我这里就不介绍如何搭建rust开发环境了，如果你是刚刚开始接触rust，建议先看下基础语法，rust开发环境可以参考本人的[Rust学习笔记（1）](https://blog.testops.vip/2022/03/07/rust-1/)

### 添加依赖

使用`Argos`框架，需要同时引入`argos`以及`argos-macros`，当然，为了能够异步运行，`tokio`也必不可少：

```toml
[dependencies]
argos = "0.1.0"
argos-macros = "0.1.0"
tokio = { version = "1.32.0", features = ["full"] }
```

### 定义API

现在我们可以定义一个“Hello”的函数，通过`GET`方法访问，访问路径为`/api/hello`，接受一个参数叫`name`：

```rust
#[route(GET, path = "/api/hello", formatter = "text")]
pub fn hello(req: HttpRequest) -> Result<String, ReturnError<String>> {
    let url_params = req.url_params();
    let name = url_params.get("name");
    match name {
        Some(name) => Ok(format!("hello {}!", name)),
        None => Err(
            ReturnError::new(
                400,
                "name is required".to_string(),
            )
        ),
    }
}
```

> 宏属性`route`定义在`hello`函数上，第一个值是请求方法，第二个值是访问路径，第三个值是返回类型，这里指定为最简单的文本类型。<br />
> 自定义函数需要满足几个要求，第一，函数只能有唯一的一个参数，类型为`argos::request::HttpRequest`，这个参数是客户端请求的所有内容。第二，返回类型必须是`Result<A, ReturnError<B>>`，这里的泛型A和B必须满足`Display`特征。<br />

### 启动服务

现在可以在`main`中启动web服务：

```rust
#[tokio::main]
async fn main() {
    let server = Server::builder(([127, 0, 0, 1], 3000).into())
        .build()
        .await
        .unwrap();
    server.start().await.unwrap();
}
```

> 服务将绑定`127.0.0.1:3000`。

### 测试

如果我们使用curl访问`http://127.0.0.1:3000/api/hello?name=liudao`，将会得到响应：`hello, liudao`。

## Restful类型的接口

如果要创建一个`restful`类型的接口，请求和返回都是`json`类型，也相当容易。

看下面的例子：

```rust
#[derive(Serialize, Deserialize)]
pub struct Hello {
    name: String,
    greeting: String,
}

#[derive(Serialize, Deserialize, Clone)]
pub struct MyError {
    code: u32,
    msg: String,
}

impl Display for MyError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "{{code:{}, msg:{}}}", self.code, self.msg)
    }
}

#[route(GET, path = "/api/hello", formatter = "json")]
pub fn hello(req: HttpRequest) -> Result<Hello, ReturnError<MyError>> {
    let url_params = req.url_params();
    let name = url_params.get("name");
    match name {
        Some(name) => Ok(Hello { name: name.to_string(), greeting: "hello".to_string() }),
        None => Err(
            ReturnError::new(
                400,
                MyError { code: 1, msg: "name is required".to_string() },
            )
        ),
    }
}
```

> 这里我们定义了一个`Hello`结构，注意我们使用`serde`的序列化和反序列化宏来实现结构体和字符串之间的转换。<br>
> 同样的，错误返回也是需要自定义一个结构体，这里我们定义了`MyError`，并需要为它实现`Display`特性<br />

## 路径参数

Argos框架的`route`同样支持路径参数，比如`/api/user/:id`，这里的`:id`就是路径参数，id表示路径参数的名称。路径参数可以从`HttpRequest`中获取。

## 定义过滤器

Argos最强大的一点，就是可以定义过滤器，过滤器将在请求函数被执行之前先一步执行，多说无益，直接看例子：

```rust
#[route(GET, path = "/api/hello", formatter = "text")]
pub fn hello(req: HttpRequest) -> Result<String, ReturnError<String>> {
    let url_params = req.url_params();
    let name = url_params.get("name");
    let path_params = req.path_params();
    Ok(format!("hello {:?}, {:?}, {:?}", name, path_params, req.attributes()))
}

#[filter(path_pattern="/api/hello.*", order=1)]
pub fn path_filter(mut req: HttpRequest) -> Chain {
    let attr = req.attributes_mut();
    attr.insert("foo".to_string(), "bar".to_string());
    if req.headers().contains_key("token") {
        Chain::Continune(req)
    } else {
        let mut return_err = ReturnError::new(
            401, 
            "not authorized".to_string(),
        );
        return_err.headers.append("filter", HeaderValue::from_str("rejected").unwrap());
        Chain::Reject(return_err)
    }
    
}
```

`hello`函数就不多做解释了，就是我们之前那个最简单的例子。主要的是`filter`宏属性。

这里的`filter`宏属性申明了两个值，一个是`path_pattern`，表示一个匹配值，它支持正则表达式。这里我们赋值为`/api/hello.*`，这表示请求路径如果满足这个表达式，都会被这个过滤器处理。目前Argos框架只支持路径匹配过滤器。

`order`表示这个过滤器的优先级，数字越小，表示优先级越高。请求进入服务后，会根据过滤器的优先级，一层层进行过滤，对满足匹配条件的，就会进入过滤器函数，不满足则跳过。最后才会进入到API函数。

过滤器中，可以对请求内容进行读写处理，如果需要继续往后，交给后面的过滤器，则只需要返回`Chain::Continune(req)`即可，如果需要拒绝掉该请求，则返回`Chain::Reject`。这样就会直接返回该请求，而不会继续往后面处理。

## 支持TLS

Argos框架还支持TLS服务，也就是所谓的`HTTPS`。当然首先你需要证书以及私钥文件。可以自己使用OpenSSL生成，当然自己生成的因为没有CA颁发的证书信任链，不能在正式环境上使用。

一条命令就可以生成证书和私钥：

```shell
openssl req -newkey rsa:2048 -nodes -keyout rsa_private.key -x509 -days 365 -out cert.crt
```

这里的`rsa_private.key`就是私钥文件，`cert.crt`就是服务器证书，当然都是`PEM`格式的。

启动服务时，只需要将这两个文件加载进去即可：

```rust
#[tokio::main]
async fn main() {
    let server = Server::builder(([127, 0, 0, 1], 3000).into())
        .ssl(
            "rsa_private.key", 
            "cert.crt", 
            argos::server::SslFiletype::PEM,
        ).unwrap()
        .build()
        .await
        .unwrap();
    server.start().await.unwrap();
}
```

## 后续

Argos框架基于`hyper`框架进行开发，目前还较为简易，更多特性会在后续版本中逐步加入。