---
title: 怎样才算完整的接口自动化测试案例
author: 六道先生
date: 2022-12-12 10:41:05
categories:
- IT
tags:
- 接口测试
- Rust
- 自动化测试
+ description: 测试人员的编码水平参差不齐，所以看到了太多像屎山一样的自动化测试的代码，忍不了了，来说说接口自动化测试案例应该写成什么样子吧。
---

# 代码 VS 低（零）代码平台
你在公司里开展自动化测试，是使用纯代码的方式还是利用已有的低代码或者零代码平台？本人的观点，一直很清晰，自动化测试，最佳的方案就是走纯代码。为啥？一定会有人跳出来反驳我，说“我们公司的低代码平台/零代码平台就很好用啊，很方便啊”。没错，好用的一定会有，但是大概率没有深入去实现一个自动化测试用例，能把一个自动化测试用例写清楚，并且具备高可维护性的低代码/零代码平台不是没有，而是需要很多年去打磨，才能做得出来。有公司愿意去付出这样的代价吗？投入产出比未免有点低。纯代码的方式去实现自动化测试其实更合理一些。也许有人会反驳我，觉得低代码/零代码平台能让更多编码能力不行的测试人员上手自动化测试。我倒觉得这是一种退步，技术方向的退步。测试工程师作为IT从业人员，而且是技术类的，不会写代码首先就是不合理的，而且降低行业要求，其实等于降低行业门槛，从而也毁了这个行业，从这个角度，我更是反对低代码或者零代码平台。当然，这是我的个人观点，也代表了个人的坚持，不代表观点一定正确。

# 接口测试自动化案例的要素
一个好的接口自动化测试应该包含哪些要素？总结一下，基本需要这些：
+ 环境
+ 前置条件（比如数据准备、登录信息等）
+ 测试数据
+ 接口定义
+ 操作步骤
+ 断言
+ 清理

所有语言都是苍白的，咱们来具体看个例子。

## 目标接口
我们这里不举登录接口的例子，那都臭了街了，网上说的太多，这里举一个具有典型性的例子：修改订单接口————`modify`。

先看接口的定义：

> 路径：`/order/{orderNbr}/modify` <br>
> 请求方式：`GET` <br>
> URL参数：`address="新地址"` <br>
> Header：`Access-Token: Token串` <br>
> 返回：
> ```json
> {
>   "code": 1000,
>   "message": "success"
> }
> ```
> `orderNbr`为订单编号

这个接口需要先有登录后的`Token`，需要存在一份已有订单，才可以成功修改，并且这样的修改，影响了数据库的数据（写接口的特点）。

## 编写接口测试代码
我们这里用`Rust`为例来写测试代码。

### 代码文件组织
我们用一个文件对应一个接口，这样有利于接口测试的管理。这里我们创建一个文件叫`modify.rs`，同`modify`接口名称保持一致。

至于其他的，比如数据库操作的通用方法，`cache`操作的通用方法等等，可以另建`mod`来整理。

### 准备第三方依赖
根据接口测试的特点，我们需要准备好我们所需要用到的第三方库，比如`请求库`、`数据库连接库`等。看如下清单：
```Toml
# 请求库
reqwest = { version = "0.11", features = ["json"] } 
# 异步库
tokio = { version = "1", features = ["full"] } 
# 序列化库
serde_json = "1.0.88"
serde_derive = "1.0.147"
serde = "1.0.147"
# 测试参数化库
rstest = "0.15.0"
# url解析库
url = "2.3.1"
# redis库
redis = "0.22.1"
# mysql库
mysql = "23.0.1"
# 日志库
tracing = "0.1.37"
tracing-test = "0.2.3"
```

### 定义接口
我们首先在`modify.rs`中定义好这个接口的请求结构和响应结构，当然这个接口因为是`GET`请求，没有请求的`body`，所以请求结构这里就不定义了：
```rust
// modify.rs
// Modify接口定义返回结构体
#[derive(Deserialize, Serialize, Debug)]
pub struct ModifyResponse {
    pub code: u32,
    pub message: String,
}
```

然后定义接口调用的方法：
```rust
// modify.rs

// 定义Modify接口调用函数
pub async fn modify(base_url: &str, token: &str, order_number: &str, address: &str) -> Result<ModifyResponse, Box<dyn std::error::Error>> {
    let client = reqwest::Client::new();

    // address字符串因为要放在URL中，所以需要进行URL编码
    let address = url::form_urlencoded::byte_serialize(address.as_bytes()).collect::<String>();

    // 根据环境拼装接口地址
    let url = format!("{}/order/{}/modify?address={}", base_url, order_number, address);

    // 发送请求
    let resp = client.get(&url).header("Access-Token", token).send().await;
    info!("modify resp: {:?}", resp);

    
    // 处理返回结果
    return match resp {
        Ok(resp) => {
            let resp = resp.json::<ModifyResponse>().await;
            match resp {
                Ok(resp) => Ok(resp),
                Err(err) => Err(Box::new(err)),
            }
        }
        Err(err) => Err(Box::new(err)),
    };
}
```
> 注意一定要把请求的url拆成两部分，一部分是主机地址（base_url），一部分是路径。这样针对不同的环境，只需要更换主机地址就可以了。这样的调用，可以匹配不同的环境。

### 测试案例
接下来就是写针对这个接口的测试案例了。

首先我们要考虑这个接口的依赖，这个接口必须在`header`中传递`Access-Token`，来传递登录信息。我们可以有两种方式处理，一种，是在调用这个接口之前，先调用登录接口来获取这个`Token`，并把这个`Token`加入到当前这个接口调用中。这种方式我并不是很推荐，因为这样一来，这个接口的测试就必须依赖登录接口，如果登录接口出了问题，就会造成这个接口的测试统统失败。不符合测试独立性的原则。当然，如果实在没有办法绕过，这种方法也是可以考虑的。

另一种，就是根据登录获取`Token`的实际原理，绕过登录接口。在这个例子中，服务端是通过登录成功后，生成一个`Token`存入`redis`中，作为`key`，并把用户的一些信息序列化成一个`json`串作为`value`。当需要`Token`的接口请求进来是，就去`redis`中找到对应的进行校验。那么我们可以直接往`redis`中写入`Token`和`json`值，那么用这个`Token`就直接可以通过校验了：
```rust
// redis mod.rs
#[derive(Deserialize, Serialize, Clone)]
pub struct UserInfo {
    #[serde(rename = "accountId")]
    pub account_id: u32,
    #[serde(rename = "accountName")]
    pub account_name: String,
    pub cellphone: String,
    pub gender: u8,
}

impl UserInfo {
    pub fn new(account_id: u32, account_name: String, cellphone: String, gender: u8) -> Self {
        Self {
            account_id,
            account_name,
            cellphone,
            gender,
        }
    }
    
    pub fn prepare_login_user(self, url: &str, token: &str) -> Result<(), Box<dyn std::error::Error>>{
        let client = redis::Client::open(url)?;
        let mut con = client.get_connection()?;
        let value = serde_json::to_string(&self)?;
        let _ : () = con.set(token, value)?;
        Ok(())
    }
}
```
> 我们这样就可以使用`prepare_login_user`方法直接实现登录，而不需要去调用登录接口，解除了对其他接口的耦合性。

另外，调用这个接口，需要在数据库中已经存在一张订单，才可以修改订单地址。这也有多种方式实现。

一种是先调用创建订单接口，把订单准备好，再调用当前这个修改订单地址的接口。但是同样的道理，也会造成接口间的依赖。

另一种，就是在测试环境准备时，数据库就事先埋好一些数据，包括用于这个测试的订单。这种环境准备时埋入的数据，我称为`铺底数据`，适合于不太会变化，大部分案例都需要用的数据。比如一些基础用户信息，一些产品信息等等。而这个订单信息，可能就这一个case会用，并且会被这个接口变更，我们不可能在这个案例执行一边后，就重新执行`铺底数据`的初始化，这样每次执行显得太`重`了。

这类数据，我更推荐在案例执行之前，写在方法的`init`中，单独为这个案例准备一条数据，案例执行完成后，就清理掉：
```rust
// modify.rs

    fn init_db(db_conf: &MySQLConfig, account_id: u32, order_number: &str) {
        info!("init db data");
        let mut conn = db_conf.get_conn();
        let result = conn.exec_drop(
            r"INSERT INTO `t_order` (`orderNbr`, `orderStatus`, `buyerId`, `address`, `createTime`, `updateTime`) 
                VALUES (?, ?, ?, ?, now(), now());",
            (
                order_number,
                0,
                account_id,
                "Streat Z, Pudong district, Shanghai",),
        );
        assert!(result.is_ok(), "init db failed");
    }

    fn clean_db(db_conf: &MySQLConfig, order_number: &str) {
        info!("clean db data in `t_order`");
        let mut conn =db_conf.get_conn();
        let result = conn.exec_drop(
            r"DELETE FROM `t_order` WHERE `orderNbr` = ?;",
            (order_number,),
        );
        assert!(result.is_ok(), "clean db failed");
    }
```
> 这两个方法放在测试模块中待被调用。

现在可以正式写这个测试了：
```rust
// modify.rs

    #[tracing_test::traced_test]
    #[rstest]
    #[case("s001", "Streat A, Pudong district, Shanghai", 1000, "Modify address success")]
    #[tokio::test]
    async fn test_modify_success(#[case] order_nbr: &str, #[case] address: &str, #[case] code: u32, #[case] message: &str) {
        info!("case: test_modify_success start, with params: order_nbr: {}, address: {}", order_nbr, address);
        // ---- 初始化测试数据 ----
        let env = super::super::get_env();
        info!("get env: {:?}", env);
        let mysql_conf = MySQLConfig::new(
            env.db_conf.host.clone(), 
            env.db_conf.port,
            env.db_conf.username.clone(), 
            env.db_conf.password.clone(), 
            "coffeedb".to_string(),
        );
        let token: &str = "eyJhbGciOiJIUzI1NiJ9.eyJhdWQiOiJsaXVkYW8iLCJleHAiOjE2NzA5ODkyNDV9.aegReDbly0asm4lC6aOCvn1gW26_cFGqmxqBeV-JI90";
        let user_info = UserInfo::new(
            1, 
            "liudao".to_string(), 
            "13511111111".to_string(), 
            0,
        );

        init(&env.redis_conf.url, &mysql_conf, token, user_info.clone(), order_nbr);

        // ---- 执行测试 ----
        // 发起请求
        let resp = modify(&env.url, token, order_nbr, address).await.unwrap();

        // ---- 验证结果 ----
        // 获取数据库数据
        let mut conn = mysql_conf.get_conn();
        let record = conn.exec_map(
            r"SELECT `orderNbr`, `orderStatus`, `buyerId`, `address` FROM `t_order` WHERE `orderNbr` = ?", 
            (order_nbr,), 
            |(order_nbr, order_status, buyer_id, address): (String, u8, u32, String)| {
                OrderInfo {
                    order_nbr,
                    order_status,
                    buyer_id,
                    address,
                }
            },
        ).unwrap();

        let result = panic::catch_unwind( || {
            //返回结果断言
            assert_eq!(resp.code, code);
            assert_eq!(resp.message, message);

            // 数据库记录断言
            assert_eq!(record.len(), 1);
            assert_eq!(record, vec![OrderInfo {
                order_nbr: order_nbr.to_string(),
                order_status: 0,
                buyer_id: user_info.account_id,
                address: address.to_string(),
            }]);
        });
        
        // ---- 清理测试数据 ----
        clean(&mysql_conf, order_nbr);

        if let Err(e) = result {
            panic::resume_unwind(e);
        }
       

    }
```
> 在这个测试案例中，有数据参数化，有日志输出，有`init`准备，有`clean`清理测试数据，有接口调用，有结果校验。<br>
可以通过在执行时加入环境变量`ENV=xxx`来指定被测环境，允许在多个测试环境间切换。<br>
并且在结果校验中，包含了请求返回数据的校验，以及影响的数据库字段的校验。<br>

这就是一个相当完整的接口测试案例了。

# 写在结尾
当然，这里只是用`rust`举了一个比较典型的例子，工作中一定还会遇到各种特殊的情况，多考虑案例的独立性，并且满足编程的基本原则，就不会出大的问题。

正是因为有这么多需要考虑的点，所以要做一个接口测试的低代码或者零代码平台，需要处理的细节太多了，没有几年的打磨，很难满足这么多要求。而我所展示的，只是接口测试的基本要求，任何一条不满足，这个接口测试就是存在问题的。所以使用代码形式来写接口测试，是最容易满足这一系列条件的。

如果您是使用其他编程语言来实现接口自动化测试的，也需要考虑这些条件。既然要写代码，那就请把它写好，写的像个样子。

完整代码，请看[GitHub传送门]。

[GitHub传送门]: https://github.com/jimmyseraph/api-test-demo