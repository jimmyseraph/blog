---
title: Rust学习笔记（6）
author: 六道先生
date: 2022-03-12 14:35:24
categories:
- IT
tags:
- Rust
- 编程语言
+ description: 学习rust的第六天开始了，主要学习Rust的枚举类型。
---
# 定义枚举类型
枚举类型基本定义方式：
```Rust
enum IpAddrKind {
    V4,
    V6,
}
fn main() {
    let four = IpAddrKind::V4;
    let six = IpAddrKind::V6;
}
```
来点具体的例子
```Rust
    enum IpAddrKind {
        V4,
        V6,
    }

    struct IpAddr {
        kind: IpAddrKind,
        address: String,
    }

    let home = IpAddr {
        kind: IpAddrKind::V4,
        address: String::from("127.0.0.1"),
    };

    let loopback = IpAddr {
        kind: IpAddrKind::V6,
        address: String::from("::1"),
    };
```
上面这个写法，在一个结构体里面，混合了一个枚举类型和一个String类型。不过还有更简单的表达方式：
```Rust
    enum IpAddr {
        V4(String),
        V6(String),
    }

    let home = IpAddr::V4(String::from("127.0.0.1"));

    let loopback = IpAddr::V6(String::from("::1"));
```
> 这个写法，其实就等价于定义了V4和V6两个关联方法，只是没有写实现的地方。

还有泛型的写法：
```Rust
enum Option<T> {
    None,
    Some(T),
}
```
这个用法跟java中的泛型一样，根据实际赋值时候的类型，确定具体类型：
```Rust
    let some_number = Some(5);
    let some_string = Some("a string");

    let absent_number: Option<i32> = Option::<i32>::None;
```
# match控制流结构
枚举类型用match来进行控制是最合适的，和其他语言一样，就类似用switch-case配合枚举类型一样，看下面的例子：
```Rust
enum Coin {
    Penny,
    Nickel,
    Dime,
    Quarter,
}

fn value_in_cents(coin: Coin) -> u8 {
    match coin {
        Coin::Penny => {
            println!("Lucky penny!");
            1
        }
        Coin::Nickel => 5,
        Coin::Dime => 10,
        Coin::Quarter => 25,
    }
}
```
同样的，match也可以处理带有值的枚举：
```Rust
#[derive(Debug)] 
enum UsState {
    Alabama,
    Alaska,
    // --snip--
}

enum Coin {
    Penny,
    Nickel,
    Dime,
    Quarter(UsState),
}
fn main() {
    let value = value_in_cents(Coin::Quarter(UsState::Alabama));

    println!("{}", value)
}

fn value_in_cents(coin: Coin) -> u8 {
    match coin {
        Coin::Penny => 1,
        Coin::Nickel => 5,
        Coin::Dime => 10,
        Coin::Quarter(state) => {
            println!("State quarter from {:?}!", state);
            25
        }
    }
}
```
带有泛型的枚举也可以用match匹配：
```Rust
#[derive(Debug)]
enum Option<T> {
    None,
    Some(T),
}
fn main() {

    let five = Option::<i32>::Some(5);
    println!("{:?}", five);

    let six = plus_one(five);
    let none = plus_one(Option::<i32>::None);

    println!("{:?}, {:?}", six, none)
}

fn plus_one(x: Option<i32>) -> Option<i32> {
    match x {
        Option::<i32>::None => Option::<i32>::None,
        Option::<i32>::Some(i) => Option::<i32>::Some(i + 1),
    }
}
```
注意了，match处理的那个None，虽然直接就返回了None，但是也不能缺少。Rust编译时会知道match在匹配枚举类型时缺少了分支没有，一旦缺少，就会提示报错。那么match中有没有像java中的switch-case里面的default来处理其他分支呢？看例子：
```Rust
    let dice_roll = 9;
    match dice_roll {
        3 => add_fancy_hat(),
        7 => remove_fancy_hat(),
        other => move_player(other),
    }

    fn add_fancy_hat() {}
    fn remove_fancy_hat() {}
    fn move_player(num_spaces: u8) {}
```
如果处理other分支并不需要参数传入，那可以用`_`来替代other：
```Rust
    let dice_roll = 9;
    match dice_roll {
        3 => add_fancy_hat(),
        7 => remove_fancy_hat(),
        _ => reroll(),
    }

    fn add_fancy_hat() {}
    fn remove_fancy_hat() {}
    fn reroll() {}
```
如果other分支不需要做任何处理，可以写成这样：
```Rust
    let dice_roll = 9;
    match dice_roll {
        3 => add_fancy_hat(),
        7 => remove_fancy_hat(),
        _ => (),
    }

    fn add_fancy_hat() {}
    fn remove_fancy_hat() {}
```
# 用`if let`实现精简控制流
`if let`语法比较特别，先看原始的例子：
```Rust
    let config_max = Some(3u8);
    match config_max {
        Some(max) => println!("The maximum is configured to be {}", max),
        _ => (),
    }
```
> Some就是之前例子里面的枚举类型之一。

我们可以用`if let`进行简写：
```Rust
    let config_max = Some(3u8);
    if let Some(max) = config_max {
        println!("The maximum is configured to be {}", max);
    }
```
> let后面跟的就是类似match中跟的匹配形式（pattern），config_max就是用于判断是否符合形式的输入。

再看一个带有else的例子，原始match格式如下：
```Rust
    let mut count = 0;
    match coin {
        Coin::Quarter(state) => println!("State quarter from {:?}!", state),
        _ => count += 1,
    }
```
改写成`if let`形式：
```Rust
    let mut count = 0;
    if let Coin::Quarter(state) = coin {
        println!("State quarter from {:?}!", state);
    } else {
        count += 1;
    }
```