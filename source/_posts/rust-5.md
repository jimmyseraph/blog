---
title: Rust学习笔记（5）
author: 六道先生
date: 2022-03-11 11:09:49
categories:
- IT
tags:
- Rust
- 编程语言
+ description: 学习rust的第五天开始了，主要学习stuct结构体。
---
# 定义和实例化
结构体和tuple类似，都可以将一些相关的值组织在一起，唯一不同的是结构体里面的每一个变量需要命名，所以可读性上比tuple更好，而且也不用关心里面的值的顺序了。定义和赋值跟golang很像，直接用例子来看：
```Rust
struct User {
    active: bool,
    username: String,
    email: String,
    sign_in_count: u64,
}

fn main() {
    let user1 = User {
        email: String::from("someone@example.com"),
        username: String::from("someusername123"),
        active: true,
        sign_in_count: 1,
    };
}
```
然后学一种struct赋值的简易形式，先看基本的：
```Rust
fn build_user(email: String, username: String) -> User {
    User {
        email: email,
        username: username,
        active: true,
        sign_in_count: 1,
    }
}
```
email和username的参数名字和struct中的field的名字一致，那就可以简写：
```Rust
fn build_user(email: String, username: String) -> User {
    User {
        email,
        username,
        active: true,
        sign_in_count: 1,
    }
}
```
# 用一个struct给另一个struct赋值
还是以上面那个struct为例，假设有一个user2，它的field中username、active、sign_in_count的值和user1一样，只有email不一样，那一般是这样赋值：
```Rust
fn main() {
    // --snip--

    let user2 = User {
        active: user1.active,
        username: user1.username,
        email: String::from("another@example.com"),
        sign_in_count: user1.sign_in_count,
    };
}
```
不过Rust中有更方便的方式：
```Rust
fn main() {
    // --snip--

    let user2 = User {
        email: String::from("another@example.com"),
        ..user1
    };
}
```
> 这写法有点像ES6的语法，只不过ES6中是三个`.`，而Rust是一个`.`。不过需要注意，如果user1中有field发生“move”行为，那user1中那个field就失效了。就像上面这个例子，user1中的username，在赋值user2后，就失效了，后面不可以再访问。

# tuple struct结构体
Rust允许定义类似tuple的struct，像这样：
```Rust
struct Color(i32, i32, i32);
struct Point(i32, i32, i32);

fn main() {
    let black = Color(0, 0, 0);
    let origin = Point(0, 0, 0);
}
```
> 这里的black和orgin虽然看似值一样，但是不能互相赋值，因为类型不同。

# 空结构体
就是没有任何字段的结构体：
```Rust
struct AlwaysEqual;

fn main() {
    let subject = AlwaysEqual;
}
```
# 关于struct的ownership
在struct中，一般不使用引用，因为Rust希望struct中的字段都能完整的拥有值，有统一的生命周期。如果要在其中使用引用，必须申明“生命周期”（lifetime），这个概念我们后面看。先看这个错误的例子：
```Rust
struct User {
    active: bool,
    username: &str,
    email: &str,
    sign_in_count: u64,
}

fn main() {
    let user1 = User {
        email: "someone@example.com",
        username: "someusername123",
        active: true,
        sign_in_count: 1,
    };
}
```
将会报错：
```Shell
$ cargo run
   Compiling structs v0.1.0 (file:///projects/structs)
error[E0106]: missing lifetime specifier
 --> src/main.rs:3:15
  |
3 |     username: &str,
  |               ^ expected named lifetime parameter
  |
help: consider introducing a named lifetime parameter
  |
1 ~ struct User<'a> {
2 |     active: bool,
3 ~     username: &'a str,
  |

error[E0106]: missing lifetime specifier
 --> src/main.rs:4:12
  |
4 |     email: &str,
  |            ^ expected named lifetime parameter
  |
help: consider introducing a named lifetime parameter
  |
1 ~ struct User<'a> {
2 |     active: bool,
3 |     username: &str,
4 ~     email: &'a str,
  |
```
# struct打印
struct默认情况下不能直接用println进行打印，看下面的例子：
```Rust
struct Rectangle {
    width: u32,
    height: u32,
}

fn main() {
    let rect1 = Rectangle {
        width: 30,
        height: 50,
    };

    println!("rect1 is {}", rect1);
}
```
这将会报错：
```Shell
error[E0277]: `Rectangle` doesn't implement `std::fmt::Display`
```
需要打开debug，才可以使用`{:?}`打印，或者`{:#?}`以更好看的形式打印：
```Rust
#[derive(Debug)]
struct Rectangle {
    width: u32,
    height: u32,
}

fn main() {
    let rect1 = Rectangle {
        width: 30,
        height: 50,
    };

    println!("rect1 is {:#?}", rect1);
}
```
打印效果是这样的：
```Shell
$ cargo run
   Compiling rectangles v0.1.0 (file:///projects/rectangles)
    Finished dev [unoptimized + debuginfo] target(s) in 0.48s
     Running `target/debug/rectangles`
rect1 is Rectangle {
    width: 30,
    height: 50,
}
```
# 标准错误输出
dbg!是标准错误输出的宏，跟标准输出println对应：
```Rust
#[derive(Debug)]
struct Rectangle {
    width: u32,
    height: u32,
}

fn main() {
    let scale = 2;
    let rect1 = Rectangle {
        width: dbg!(30 * scale),
        height: 50,
    };

    dbg!(&rect1);
}
```
可以看到如下输出：
```Shell
$ cargo run
   Compiling rectangles v0.1.0 (file:///projects/rectangles)
    Finished dev [unoptimized + debuginfo] target(s) in 0.61s
     Running `target/debug/rectangles`
[src/main.rs:10] 30 * scale = 60
[src/main.rs:14] &rect1 = Rectangle {
    width: 60,
    height: 50,
}
```
# struct的方法
定义实现一个struct的方法，这个有点类似go语言，看下面的例子：
```Rust
#[derive(Debug)]
struct Rectangle {
    width: u32,
    height: u32,
}

impl Rectangle {
    fn area(&self) -> u32 {
        self.width * self.height
    }
}

fn main() {
    let rect1 = Rectangle {
        width: 30,
        height: 50,
    };

    println!(
        "The area of the rectangle is {} square pixels.",
        rect1.area()
    );
}
```
> 关键字`impl`表示实现后面的struct的方法，在里面所定义的所有方法，都从属于这个struct。<br>
area方法第一个参数是`&self`，表示这个struct的实例的引用，而在使用中，我们用`rect1`这个实例，调用area方法，这个实例就是代表了`&self`，所以area就不用这个参数了传入了。

在C/C++中，其实还有一个`->`符号调用方法的语法，当对象实例调用内部方法，那就是用`.`，当对象指针调用内部方法，就是用`->`，不过Rust没有这么麻烦，它会自行判断，帮你添加`.`，`&`或者`*`，来匹配方法调用。所以下面两种写法其实是一回事，没有区别：
```Rust
p1.distance(&p2);
(&p1).distance(&p2);
```
# 关联函数
关联函数（Associated functions），其实就是java中的静态函数的概念，在Rust中这样定义：
```Rust
impl Rectangle {
    fn square(size: u32) -> Rectangle {
        Rectangle {
            width: size,
            height: size,
        }
    }
}
```
> 这个在impl中的方法，没有把`&self`作为参数，这个方法称为关联函数，它的调用方法是使用`::`，看下面的例子：

```Rust
let sq = Rectangle::square(3);
```
