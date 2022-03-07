---
title: Rust学习笔记（2）
author: 六道先生
date: 2022-03-07 11:39:18
categories:
- IT
tags:
- Rust
- 编程语言
+ description: 学习rust的第二天开始了，加油。
---
# Cargo包管理器
## cargo创建项目
在rust中，使用cargo工具来进行包的管理，和第一章的例子不同，如果要使用cargo进行包管理，需要使用cargo命令来创建项目：
```Shell
$ cargo new <name>
```
> 用实际项目名称替换这里的`<name>`。<br>
new会创建一个对应名称的目录，并已经配置好了git本地仓库，如果已经处于git本地仓库中，则不会覆盖已存在的仓库。

如果在已有项目中创建配置cargo管理，可以用init指令：
```Shell
$ cargo init
```
> 不管是new还是init，都会在目录下生成一个Cargo.toml文件，文件的内容类似这样：
```Toml
[package]
name = "demo1"
version = "0.1.0"
edition = "2021"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]

```
## cargo编译和运行项目
被cargo管理的项目可以直接使用cargo进行编译，而不需要使用rustc：
```Shell
$ cargo build
```
构建成功后，会在当前目录下生成一个target目录，其下的debug目录里面会有编译后的可执行文件。这里和使用rustc编译后的结果有点区别，rustc编译一个文件，成功后生成的是和被编译的文件同名的可执行文件，而使用cargo构建，生成在debug目录下的可执行文件，是根据toml文件中`name`的名称命名的。<br>
另外，可以直接使用`cargo run`来执行当前的package：
```Shell
$ cargo run
   Finished dev [unoptimized + debuginfo] target(s) in 0.05s
     Running `target/debug/demo1`
Hello, world!
```
如果想构建release，可以使用如下命令：
```Shell
$ cargo build --release
```
将会在`target/release`目录下生成可执行文件
# 编程 —— Guessing Game
```Rust
use std::io;

fn main() {
    println!("Guess the number!");

    println!("Please input your guess.");

    let mut guess = String::new();

    io::stdin()
        .read_line(&mut guess)
        .expect("Failed to read line");

    println!("You guessed: {}", guess);
}
```
解说一下：
```Rust
use std::io;
```
> `use`关键字引入了一个标准库，这里有点像C#，这一行被称为“序曲”或者“前奏”（prelude），std是crate，有点类似其他语言中package的意思，像namespace（个人理解），怎么翻译呢，暂时叫“箱子”吧。io应该就是crate中的模块（module）了。<br>

```Rust
let mut guess = String::new();
```
> 这一行是定义了一个变量guess，用于保存用的户输入。let是定义变量的关键字，mut关键字用于说明变量是否可变。
```Rust
let apples = 5; // immutable
let mut bananas = 5; // mutable
```
> 没有mut说明的变量，就是不可变的。这一点rust和其他语言差异比较大，像其他语言，一般要定义不可变的常量，通常需要有类似`const`或者`final`这样的关键字来说明，而rust反而是没有关键字说明的就是不可变的常量。<br>
`String`应该是一个内置类型（type），而`new()`方法，创建了一个空的String实例。这种语法跟C#很像。

```Rust
io::stdin()
    .read_line(&mut guess)
    .expect("Failed to read line");
```
> `stdin`是io库中的一个函数，返回一个Stdin类型的实例，看名字就知道是标准输入，用于控制台标准输入的控制。<br>
`read_line`方法应该是属于Stdin实例的一个方法，用于在控制台读取输入的一行内容（已回车为结束标识），`&`符号表示后面的参数是一个引用（reference），默认情况下，引用是不可变的，只能读取值，不能修改值，即使这个变量是用mut修饰的。所以这里使用`&mut guess`来说明这个引用可修改值，而不是使用`&guess`。<br>
`expect`用于处理异常，就类似于try-catch。

```Rust
println!("You guessed: {}", guess);
```
> println里面的`{}`用法，明显就是占位符的使用了，这个很容易理解。

下面需要继续完成代码，我们引入rand，来产生一个随机数，作为猜测的目标。这个crate需要在`Cargo.toml`中的dependencies引入：
```Toml
rand = "0.8.5"
```
> 0.8.5明显是版本号

现在构建一下项目，就可以看到整个下载依赖的过程：
```Shell
$ cargo build
  Updating crates.io index
  Downloaded rand v0.8.5
  Downloaded ppv-lite86 v0.2.16
  Downloaded rand_chacha v0.3.1
  Downloaded libc v0.2.119
  Downloaded rand_core v0.6.3
  Downloaded getrandom v0.2.5
  Downloaded cfg-if v1.0.0
  Downloaded 7 crates (757.9 KB) in 2.63s
    Blocking waiting for file lock on package cache
   Compiling libc v0.2.119
   Compiling cfg-if v1.0.0
   Compiling ppv-lite86 v0.2.16
   Compiling getrandom v0.2.5
   Compiling rand_core v0.6.3
   Compiling rand_chacha v0.3.1
   Compiling rand v0.8.5
   Compiling demo1 v0.1.0 (/Users/luyiyi/rustProj/demo1)
    Finished dev [unoptimized + debuginfo] target(s) in 4m 46s
```
当进行首次build时，cargo会检查toml文件中的依赖有没有变化，如果有新的依赖，就会去`crates.io`上进行查找，并下载。完成后，会生成一个`Cargo.lock`文件，这个文件用于锁定你的依赖的版本，比如rand发布了0.8.6版本，你在项目中再次执行build的时候，并不会去升级，因为lock文件已经锁定了这个版本。但是如果你是首次build，还没有lock文件时，即使你在toml文件中指定rand版本是`v0.8.5`，还是会下载`v0.8.6`版本的。所以在这里所指定的版本，其实是指大于等于0.8.5（但是小于0.9.0）的版本。这一点跟node的`package.json`很像。<br>
如果在0.8.5版本被lock之后，还想升级0.8.6的话，可以使用update指令：
```Shell
$ cargo update
```
它会忽略lock文件，对依赖进行升级，但是升级的范围，还是大于等于0.8.5，但是小于0.9.0。如果需要升级到0.9.0这个大版本，需要修改toml文件，并重新进行build。<br>
改造之前的代码：
```Rust
use std::io;
use rand::{thread_rng, Rng};
use std::cmp::Ordering;

fn main() {

    println!("Guess the number!");

    let mut rng = thread_rng();
    let secret_number : u32 = rng.gen_range(1..101);

    println!("The secret number is: {}", secret_number);

    println!("Please input your guess.");

    let mut guess = String::new();
    io::stdin()
        .read_line(&mut guess)
        .expect("Failed to read line");
    
    let guess: u32 = guess.trim().parse().expect("Please type a number!");
    
    println!("You guessed: {}", guess);

    match guess.cmp(&secret_number.to_string()) {
        Ordering::Less => println!("Too small!"),
        Ordering::Greater => println!("Too big!"),
        Ordering::Equal => println!("You win!"),
    }

}
```
> `use rand::{thread_rng, Rng}`这个引用比较特别，表示在rand这个crate下，引入两个东西，thread_rng明显是方法，而Rng，根据官方介绍，叫做trait，怎么解释呢，我就翻译成特征吧。这个概念后面会学到，这里只需要理解为产生随机数的具体实现。这个概念有点像“接口——实现”的关系。很明显Rnd在代码中没有被使用过，而下面的语句：
```Rust
let mut rng = thread_rng();
let secret_number : u32 = rng.gen_range(1..101);
```
> 使用了thread_rng函数，返回了一个`ThreadRng`的实例，这个实例，明显是线程安全的，而这个实例可以使用`gen_range`方法，这个方法是在Rng这个trait中定义的，所以这就说明了在Rust中，交给接口的实现，需要在use中引入。（记重点）<br>
另外，`let secret_number : u32`这个定义多了对secret_number这个变量的类型说明（u32，明显是无符号int32类型），显然是因为无法从后面的gen_range方法推断前面的变量类型的原因。如果不加这个类型说明，编译过程会报错。

类型变化：
```Rust
let guess: u32 = guess.trim().parse().expect("Please type a number!");
```
> 这里把guess从原来的字符串转为了数字（u32），并做了异常处理。

接着看对比：
```Rust
match guess.cmp(&secret_number) {
    Ordering::Less => println!("Too small!"),
    Ordering::Greater => println!("Too big!"),
    Ordering::Equal => println!("You win!"),
}
```
> 这是一个match表达式，其中调用了cmp方法，cmp方法可以用于所有可比较类型，这里我们把guess和secret_number进行比较。<br>
引入了Ordering类型，用了其中的枚举类型：Less、Greater、Equal，`=>`符合后面就是满足条件时，所执行的语句。<br>
match很像其他语言的switch-case。

执行一下：
```Shell
$ cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.00s
     Running `target/debug/demo1`
Guess the number!
The secret number is: 14
Please input your guess.
80
You guessed: 80

Too big!
```

进一步改造，希望猜错的话继续才，猜对了才退出，这里需要增加一个loop，并处理一下输入的时候非数字的异常：
```Rust
use rand::Rng;
use std::cmp::Ordering;
use std::io;

fn main() {
    println!("Guess the number!");

    let secret_number = rand::thread_rng().gen_range(1..101);

    loop {
        println!("Please input your guess.");

        let mut guess = String::new();

        io::stdin()
            .read_line(&mut guess)
            .expect("Failed to read line");

        let guess: u32 = match guess.trim().parse() {
            Ok(num) => num,
            Err(_) => continue,
        };

        println!("You guessed: {}", guess);

        match guess.cmp(&secret_number) {
            Ordering::Less => println!("Too small!"),
            Ordering::Greater => println!("Too big!"),
            Ordering::Equal => {
                println!("You win!");
                break;
            }
        }
    }
}
```
> 注意输入的字符串转数字时候，也用了match表达式，Ok和Err也都是枚举类型。