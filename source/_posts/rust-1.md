---
title: Rust学习笔记（1）
author: 六道先生
date: 2022-03-07 09:53:51
categories:
- IT
tags:
- Rust
- 编程语言
+ description: 从零开始学习Rust编程，谁让它现在这么火呢？总要赶上时代潮流啊，不能让自己落在后面。
---
# 初识Rust
官网：https://www.rust-lang.org/ <br>
安装:<br>
通过命令行方式安装：
```Shell
$ curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```
升级：
```Shell
$ rustup update
```
卸载：
```Shell
$ rustup self uninstall
```
# 工具集
默认三个工具：
+ rustup —— rust管理工具，用于管理项目创建初始化，工具升级卸载等
+ rustc —— rust编译器，编译rs
+ cargo —— 仓库管理工具，用于管理各自依赖和模块以及工具

查看版本:
```Shell
$ rustup --version
$ rustc --version
$ cargo --version
```
rust沙盒在线环境：https://play.rust-lang.org/
# Hello World
创建一个hello.rs（或者在线上沙盒环境），输入代码：
```Rust
fn main() {
    println!("Hello, world!");
}
```
在命令行编译运行：
```Shell
$ rustc hello.rs
$ ./hello
Hello, world!
```
解读一下hello.rs
> main就是主函数入口，跟所有其他语言类似，不多做解释。<br>
这里的println并不是函数，而是宏（macros），`!`表示调用宏，而不是函数，宏和函数的区别后面再学。<br>
语句使用分号作为结束符，这一点和c还有java一致的。

rust是一种预先编译（ahead-of-time compiled）语言，和C/C++、Golang等类似，通过rustc可以把代码编译成可执行文件给别人运行。
# VS Code开发环境配置
貌似很简单，就安装一个叫Rust的插件就行了，安装完成后就有自动完成、代码分析等功能了，很方便。
{% asset_img rust-extension.jpg vscode-rust-extension %}