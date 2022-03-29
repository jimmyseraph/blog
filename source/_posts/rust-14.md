---
title: Rust学习笔记（14）
author: 六道先生
date: 2022-03-29 13:03:26
categories:
- IT
tags:
- Rust
- 编程语言
+ description: 学习rust的第十四天开始了，主要学习Rust关于Cargo和crate.io更多的一些知识。
---
# 关于cargo的build
默认情况下，直接运行`cargo build`会直接编译开发版本，如果增加了参数`--release`就会进行发布版编译。这两者的区别，大家应该都知道：
```Shell
$ cargo build
    Finished dev [unoptimized + debuginfo] target(s) in 0.0s
$ cargo build --release
    Finished release [optimized] target(s) in 0.0s
```
Cargo如果在你项目的cargo.toml文件中找不到配置信息，会直接使用默认的配置信息进行dev或者release的构建。当然你也可以在toml中进行配置，配置信息用`[profile.dev]`以及`[profile.release]`来说明：
```Toml
[profile.dev]
opt-level = 0

[profile.release]
opt-level = 3
```
> opt-level是一个优化系数，取值范围是0-3，0表示不进行优化，3表示最高级优化。优化会带来额外的编译时间，所以如果你需要频繁构建，那这个值就设低，比如dev构建。

# 文档注释
类似java的文档注释，最后可以用javadoc命令生成api文档一样，Rust也有文档注释，使用三个斜杠来注释`///`，并且支持Markdown语法。比如这样：<br>
```Rust
/// Adds one to the number given.
///
/// # Examples
///
/// ```
/// let arg = 5;
/// let answer = my_crate::add_one(arg);
///
/// assert_eq!(6, answer);
/// ```
pub fn add_one(x: i32) -> i32 {
    x + 1
}
```
在命令行，可以使用rustdoc命令来生成html文档，当然，在cargo项目中，可以使用`cargo doc --open`来方便的直接编译出html文档并在浏览器中打开：<br>
{% asset_img trpl14-01.png trpl14-1 %}

# 文档测试
文档注解中如果有Example段，那就可以当作文档中的测试代码，可以使用`--doc`参数来执行文档测试：
```Shell
   Doc-tests my_crate

running 1 test
test src/lib.rs - add_one (line 5) ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.27s
```
# crate文档注释
在文件的开头，可以使用`//!`或者`/*! ... */`来注释整个crate，注意这个注释只能写在文件头，不能用在下面的每个函数或者struct上。`//!`用于单行的，`/*! ... */`用于多行的。看下面的例子：
```Rust
//! # My Crate
//!
//! `my_crate` is a collection of utilities to make performing certain
//! calculations more convenient.

/// Adds one to the number given.
// --snip--
```
看生成的文档效果：<br>
{% asset_img trpl14-02.png trpl14-2 %}

> 这段注释会出现在`My Crate`这个crate的开始。

# 重导出（Re-exports）
我们假设写了一个名叫Art的crate库，那么我们在`lib.rs`中如此定义：
```Rust
//! # Art
//!
//! A library for modeling artistic concepts.

pub mod kinds {
    /// The primary colors according to the RYB color model.
    pub enum PrimaryColor {
        Red,
        Yellow,
        Blue,
    }

    /// The secondary colors according to the RYB color model.
    pub enum SecondaryColor {
        Orange,
        Green,
        Purple,
    }
}

pub mod utils {
    use crate::kinds::*;

    /// Combines two primary colors in equal amounts to create
    /// a secondary color.
    pub fn mix(c1: PrimaryColor, c2: PrimaryColor) -> SecondaryColor {
        // --snip--
        unimplemented!();
    }
}
```
这里面有两个mod，一个叫kinds，一个叫utils，我们在main中需要这样使用：
```Rust
use art::kinds::PrimaryColor;
use art::utils::mix;

fn main() {
    let red = PrimaryColor::Red;
    let yellow = PrimaryColor::Yellow;
    mix(red, yellow);
}
```
这两个`use`感觉长了点，那怎么让用户使用更方便呢？现在我们在`lib.rs`中稍微改改：
```Rust
//! # Art
//!
//! A library for modeling artistic concepts.

pub use self::kinds::PrimaryColor;
pub use self::kinds::SecondaryColor;
pub use self::utils::mix;

pub mod kinds {
    // --snip--
}

pub mod utils {
    // --snip--
}
```
我们使用了`pub use self::...`这种形式，进行重导出，看看HTML中的显示：<br>
{% asset_img trpl14-03.png trpl14-3 %}

重导出之后，我们使用时候就比较简单了：
```Rust
use art::mix;
use art::PrimaryColor;

fn main() {
    // --snip--
}
```
# 发布到crates.io
想把自己开发的crate发布到crates.io上，首先需要登录crates.io网站，去获取一个API Token。目前crates.io只支持github账号登录，不过使用chrome浏览器去做这一步操作有个问题，不知道是不是版本问题，chrome浏览器上只要一点`login with Github`，跳出来授权页面又瞬间关闭。试了一下safari浏览器，倒是可以正常操作。授权登录后，在个人配置页面上可以生成一个`API Token`，然后我们可以在本地命令行执行：
```Shell
$ cargo login <API Token>
```
这样就会把你的认证信息保存到`~/.cargo/credentials`中，后面就可以操作发布了。<br>
比如我们现在要发布之前写的`guessing_game`，首先我们要确保`Cargo.toml`配置文件中的package名称是正确的，并且在crates.io上是唯一的。而且还要加上`description`和`lisence`：
```Toml
[package]
name = "guessing_game"
version = "0.1.0"
edition = "2021"
description = "A fun game where you guess what number the computer has chosen."
license = "MIT OR Apache-2.0"
```
这样执行`cargo publish`就能顺利发布自己的crate了：
```Shell
$ cargo publish
    Updating crates.io index
   Packaging guessing_game v0.1.0 (file:///projects/guessing_game)
   Verifying guessing_game v0.1.0 (file:///projects/guessing_game)
   Compiling guessing_game v0.1.0
(file:///projects/guessing_game/target/package/guessing_game-0.1.0)
    Finished dev [unoptimized + debuginfo] target(s) in 0.19s
   Uploading guessing_game v0.1.0 (file:///projects/guessing_game)
```
当有新版本需要publish的时候，需要修改版本号version，否则会发布失败。如果要去掉某个不合适的版本，可以使用`cargo yank`命令：
```Shell
$ cargo yank --vers 1.0.1
```
也可以撤销这个版本的删除：
```Shell
$ cargo yank --vers 1.0.1 --undo
```
# Workspace
前面我们都是只创建了一个crate，一个`main.rs`和一个`lib.rs`。但是随着我们的项目越写越大，一个crate肯定不够用了，我们需要一个可执行的crate和多个库crates。这时候，我们可以使用workspace的方式进行管理。在这里，我们假设这个workspace叫add，其中需要有一个可执行crate叫adder，提供main，它依赖两个库crate，一个叫add-one，提供一个add_one函数，另一个库crate，叫add-two，提供一个add_two函数。<br>
一步一步来，首先我们创建workspace目录：
```Shell
$ mkdir add
$ cd add
```
在add目录下新建一个Cargo.toml文件：
```Toml
[workspace]

members = [
    "adder",
]
```
然后我们在add目录下创建可执行crate：
```Shell
$ cargo new adder
     Created binary (application) `adder` package
```
这时候，我们的目录结构像下面这样：
```Plain
├── Cargo.lock
├── Cargo.toml
├── adder
│   ├── Cargo.toml
│   └── src
│       └── main.rs
└── target
```
接下来，我们继续添加一个库crate：
```Toml
[workspace]

members = [
    "adder",
    "add-one",
]
```
命令行创建这个库crate：
```Shell
$ cargo new add-one --lib
     Created library `add-one` package
```
此时目录结构变成这样：
```Plain
├── Cargo.lock
├── Cargo.toml
├── add-one
│   ├── Cargo.toml
│   └── src
│       └── lib.rs
├── adder
│   ├── Cargo.toml
│   └── src
│       └── main.rs
└── target
```
我们在`add-one/src/lib.rs`中写一个函数：
```Rust
pub fn add_one(x: i32) -> i32 {
    x + 1
}
```
我们在adder目录下面的Cargo.toml中需要引入add-one这个crate：
```Toml
[dependencies]
add-one = { path = "../add-one" }
```
那么在main.rs中就可以使用了：
```Rust
use add_one;

fn main() {
    let num = 10;
    println!(
        "Hello, world! {} plus one is {}!",
        num,
        add_one::add_one(num)
    );
}
```
现在试一下编译，在add目录下，执行build命令：
```Shell
$ cargo build
   Compiling add-one v0.1.0 (file:///projects/add/add-one)
   Compiling adder v0.1.0 (file:///projects/add/adder)
    Finished dev [unoptimized + debuginfo] target(s) in 0.68s
```
如果要运行可执行crate，需要用`-p`参数指定package：
```Shell
$ cargo run -p adder
    Finished dev [unoptimized + debuginfo] target(s) in 0.0s
     Running `target/debug/adder`
Hello, world! 10 plus one is 11!
```
# 在workspace中使用外部依赖
假设我们要在add-one这个crate中使用rand这个外部依赖，那可以在add-one目录下面的Cargo.toml文件中添加依赖。它不可以被其他的crate使用。如果workspace中其他的crate也要用rand，需要在自己的package下面的Cargo.toml文件中去添加依赖。多个不同crates的Cargo.toml文件依赖了同一个rand，会在workspace根目录的Cargo.lock文件中锁定同一个版本。
# 在workspace中添加测试
我们可以在库crate中添加测试，比如`add-one/src/lib.rs`中：
```Rust
pub fn add_one(x: i32) -> i32 {
    x + 1
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn it_works() {
        assert_eq!(3, add_one(2));
    }
}
```
直接执行测试：
```Shell
   Compiling add-one v0.1.0 (file:///projects/add/add-one)
   Compiling adder v0.1.0 (file:///projects/add/adder)
    Finished test [unoptimized + debuginfo] target(s) in 0.27s
     Running target/debug/deps/add_one-f0253159197f7841

running 1 test
test tests::it_works ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

     Running target/debug/deps/adder-49979ff40686fa8e

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests add-one

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```
如果需要指定package来执行测试，同样用`-p`参数：
```Shell
$ cargo test -p add-one
    Finished test [unoptimized + debuginfo] target(s) in 0.00s
     Running target/debug/deps/add_one-b3235fea9a156f74

running 1 test
test tests::it_works ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests add-one

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```
注意，workspace中的crates需要一个一个单独的publish，workspace是不可以publish的。
# 从crates.io下载安装可执行crate
我们可以从crates.io上下载可执行crate并安装到本地，作为一个可执行工具，使用的参数是install：
```Shell
$ cargo install ripgrep
    Updating crates.io index
  Downloaded ripgrep v11.0.2
  Downloaded 1 crate (243.3 KB) in 0.88s
  Installing ripgrep v11.0.2
--snip--
   Compiling ripgrep v11.0.2
    Finished release [optimized + debuginfo] target(s) in 3m 10s
  Installing ~/.cargo/bin/rg
   Installed package `ripgrep v11.0.2` (executable `rg`)
```
安装的可执行文件存放在`~/.cargo/bin`中。

# 以cargo子命令形式运行
cargo命令的可扩展性很强，只要在你的`$PATH`下有任何一个可执行文件的命名方式是`cargo-xxx`的形式，也就是说以`cargo-`开头的，就可以使用`cargo xxx`这种方式去把这个可执行文件当作cargo命令的子命令来执行。这是一个非常方便有趣的设计。