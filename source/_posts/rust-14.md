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